USE [DBADB]
GO

/****** Object:  Table [dbo].[DL_REPORT_XML_DATA]    Script Date: 2/17/2026 7:02:19 PM ******/
SET ANSI_NULLS ON
GO

SET QUOTED_IDENTIFIER ON
GO

CREATE TABLE #DL_REPORT_XML_DATA(
	[EVENTTIME] [datetime] NULL,
	[XMLREPORT] [xml] NULL,
	[ID] [int] IDENTITY(1,1) NOT NULL,
PRIMARY KEY CLUSTERED 
(
	[ID] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]
GO

----

INSERT INTO #DL_REPORT_XML_DATA
           ([EVENTTIME]
           ,[XMLREPORT]) 
select  timestamp_utc at time zone 'UTC' at time zone 'India Standard Time' as Event_time_IST,
cast(ef.event_data as xml).query('/event/data/value/deadlock') AS Deadlock_XML
from sys.fn_xe_file_target_read_file('system_health*.xel',NULL, NULL, NULL) ef
where ef.object_name='xml_deadlock_report' and 
CAST(ef.timestamp_utc AS DATETIME2(7)) >  DATEADD(day, -2, GETUTCDATE()) 


----------------------------------
------------------------------------
DECLARE @yesterday NVARCHAR(19) = FORMAT(DATEADD(DAY,-1,GETDATE()), 'yyyy-MM-dd 14:00:00');
DECLARE @today NVARCHAR(19) = FORMAT(GETDATE(), 'yyyy-MM-dd 14:00:00');


WITH DeadlockEvents AS (
    SELECT XMLREPORT AS DeadlockXML , ID ,EventTime
    FROM #DL_REPORT_XML_DATA
),
Victim AS (
    SELECT 
        EventTime,
        DeadlockXML,
        V.value('@id', 'NVARCHAR(100)') AS VictimProcessID
    FROM DeadlockEvents
    CROSS APPLY DeadlockXML.nodes('/deadlock/victim-list/victimProcess') AS VP(V)
),
ProcessDetails AS (
    SELECT 
        D.EventTime,
        d.DeadlockXML,
        V.VictimProcessID,
        P.value('@id', 'NVARCHAR(100)') AS ProcessID,
        P.value('(inputbuf)[1]', 'NVARCHAR(MAX)') AS InputBuf,
        (
            SELECT STRING_AGG(F.value('@procname', 'NVARCHAR(200)'), ' -> ')
            FROM P.nodes('executionStack/frame') AS FS(F)
            WHERE F.value('@procname', 'NVARCHAR(200)') NOT IN ('unknown')
        ) AS SPCallChain,
        (
            SELECT TOP 1 F.value('text()[1]', 'NVARCHAR(MAX)')
            FROM P.nodes('executionStack/frame') AS FS(F)
            ORDER BY F.value('@line', 'INT') DESC
        ) AS FinalQuery,
        R.value('@objectname', 'NVARCHAR(200)') AS LockedObject,
        R.value('local-name(.)', 'NVARCHAR(100)') AS LockType,
        D.ID
    FROM Victim V
    JOIN DeadlockEvents D ON D.EventTime = V.EventTime
    CROSS APPLY D.DeadlockXML.nodes('/deadlock/process-list/process') AS PL(P)
    OUTER APPLY D.DeadlockXML.nodes('/deadlock/resource-list/*') AS RL(R)
),
FinalReport AS (
    SELECT 
        EventTime,
        MAX(CASE WHEN ProcessID = VictimProcessID THEN FinalQuery END) AS VictimQuery,
        MAX(CASE WHEN ProcessID <> VictimProcessID THEN FinalQuery END) AS WaiterQuery,
        MAX(CASE WHEN ProcessID = VictimProcessID THEN InputBuf END) AS VictimExecutedStmt,
        MAX(CASE WHEN ProcessID <> VictimProcessID THEN InputBuf END) AS WaiterExecutedStmt,
        --MAX(CASE WHEN ProcessID = VictimProcessID THEN SPCallChain END) AS VictimSPChain,
        --MAX(CASE WHEN ProcessID <> VictimProcessID THEN SPCallChain END) AS WaiterSPChain,
        MAX(CASE WHEN ProcessID = VictimProcessID THEN LockedObject END) AS DeadlockResource,
        MAX(CASE WHEN ProcessID <> VictimProcessID THEN LockType END) AS LockType,
        D.ID
    FROM ProcessDetails D
    GROUP BY EventTime, ID
)

-- Step 3: Insert into Temp Table
SELECT * 
INTO #DeadlockReport
FROM FinalReport
WHERE EventTime >= @today
ORDER BY EventTime;

SELECT * from #DeadlockReport --Quereis

SELECT cOUNT(ID) FROM #DeadlockReport --count

drop tABLE #DeadlockReport
drop tABLE #DL_REPORT_XML_DATA
