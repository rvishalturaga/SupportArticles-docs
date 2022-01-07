---
title: Troubleshoot high-CPU-usage issues
description: Provides a procedure to help you fix high-CPU-usage issues for a server that is running SQL Server.
ms.date: 11/05/2021
ms.prod-support-area-path: Performance
ms.topic: troubleshooting
ms.prod: sql
author: cobibi 
ms.author: v-yunhya
---
# Troubleshoot high-CPU-usage issues in SQL Server

_Applies to:_ &nbsp; SQL Server

This article provides a step-by-step procedure to diagnose and fix issues that are caused by high CPU usage on a computer that's running Microsoft SQL Server.

Although there are many possible causes of high CPU that occur in SQL Server, the following are the most common ones:

- High logical reads that are caused by table or index scans because of the following:
  - Out-of-date statistics
  - Missing indexes
  - [Parameter sensitive plan (PSP) issues](/azure/azure-sql/identify-query-performance-issues)
  - Poorly designed queries
- Increases in workload

You can use the following steps to troubleshoot high-CPU-usage issues in SQL Server.

## Step 1: Verify that SQL Server is causing high CPU

Use one of the following tools to check whether the SQL Server process is actually contributing to high CPU:

- Task Manager (On the **Process** tab, check whether the CPU value for **SQL Server Windows NT-64 Bit** is close to 100 percent)
- Performance and Resource Monitor ([perfmon](/previous-versions/windows/it-pro/windows-server-2012-R2-and-2012/cc731067(v=ws.11)))
  - Counter: `Process/%User Time`, `% Privileged Time`
  - Instance: sqlservr
  
- You can use the following Powershell script to collect the counter data over 60 sec period:

  ```Powershell
      $serverName = "YourServerName"
      $Counters = @(
        ("\\$serverName" +"\Process(sqlservr*)\% User Time"),
        ("\\$serverName" +"\Process(sqlservr*)\% Privileged Time")
      )
      Get-Counter -Counter $Counters -MaxSamples 30 | ForEach {
          $_.CounterSamples | ForEach {
              [pscustomobject]@{
                  TimeStamp = $_.TimeStamp
                  Path = $_.Path
                  Value = ([Math]::Round($_.CookedValue, 3))
              } 
              Start-Sleep -s 2
          }
      }
    ```

  If you notice that `% User Time` is consistently greater than 90 percent, this would confirm that the SQL Server process is causing high CPU. However, if you notice that `% Privileged time` is consistently greater than 90 percent, this would indicate that either anti-virus software or other drivers or another OS component on the computer are contributing to high CPU. You should work with your system administrator to analyze the root cause of this behavior.

## Step 2: Identify queries contributing to CPU usage

If the Sqlservr.exe process is causing high CPU, identify the queries that are responsible for this by using the following query:

```sql
SELECT TOP 10 s.session_id,
           r.status,
           r.cpu_time,
           r.logical_reads,
           r.reads,
           r.writes,
           r.total_elapsed_time / (1000 * 60) 'Elaps M',
           SUBSTRING(st.TEXT, (r.statement_start_offset / 2) + 1,
           ((CASE r.statement_end_offset
                WHEN -1 THEN DATALENGTH(st.TEXT)
                ELSE r.statement_end_offset
            END - r.statement_start_offset) / 2) + 1) AS statement_text,
           COALESCE(QUOTENAME(DB_NAME(st.dbid)) + N'.' + QUOTENAME(OBJECT_SCHEMA_NAME(st.objectid, st.dbid)) 
           + N'.' + QUOTENAME(OBJECT_NAME(st.objectid, st.dbid)), '') AS command_text,
           r.command,
           s.login_name,
           s.host_name,
           s.program_name,
           s.last_request_end_time,
           s.login_time,
           r.open_transaction_count
FROM sys.dm_exec_sessions AS s
JOIN sys.dm_exec_requests AS r ON r.session_id = s.session_id CROSS APPLY sys.Dm_exec_sql_text(r.sql_handle) AS st
WHERE r.session_id != @@SPID
ORDER BY r.cpu_time DESC
```

## Step 3: Update statistics

After you identify the queries that have the highest CPU consumption, [update statistics](/sql/relational-databases/statistics/statistics#UpdateStatistics) for the relevant tables that are involved in these queries.

If SQL Server is still using high CPU, go to the next step.

## Step 4: Add potentially missing indexes

1. Use the following query to identify queries with high CPU usage that contain at least one missing index in the query plan.

    ```sql
    -- Captures the Total CPU time spent by a query along with the query plan and total executions
    SELECT 
           qs_cpu.total_worker_time/1000 AS total_cpu_time_ms,
           q.[text],
           p.query_plan,
           qs_cpu.execution_count,
           q.dbid,
           q.objectid,
           q.encrypted AS text_encrypted
    FROM
      (SELECT TOP 500 qs.plan_handle,
                  qs.total_worker_time,
                  qs.execution_count
       FROM sys.dm_exec_query_stats qs
       ORDER BY qs.total_worker_time DESC) AS qs_cpu 
       CROSS APPLY sys.dm_exec_sql_text(plan_handle) AS q
       CROSS APPLY sys.dm_exec_query_plan (plan_handle) p
      WHERE p.query_plan.exist('declare namespace 
       qplan="http://schemas.microsoft.com/sqlserver/2004/07/showplan";
            //qplan:MissingIndexes')=1
    ```

1. Review the execution plans for the queries identified, and tune the query by implementing the required changes. The following is an example where SQL Server will point out a missing index for your query. Right-click on the Missing index portion of the query plan and choose **Missing Index Details** to create the index in another window in SSMS.

    :::image type="content" source="media/troubleshoot-high-cpu-usage-issues/high-cpu-missing-index.png" alt-text="Screenshot of the execution plan with missing index." lightbox="media/troubleshoot-high-cpu-usage-issues/high-cpu-missing-index.png":::

1. Use the following [Dynamic Management View](/analysis-services/instances/use-dynamic-management-views-dmvs-to-monitor-analysis-services) (DMV) query to check the missing indexes and apply any recommended indexes that have high improvement measurements.

    ```sql
    SELECT CONVERT (VARCHAR(30),
        GETDATE(),
        126) AS runtime,
        mig.index_group_handle,
        mid.index_handle,
        CONVERT (DECIMAL (28, 1),
                migs.avg_total_user_cost * migs.avg_user_impact * (migs.user_seeks + migs.user_scans)) AS improvement_measure,
                'CREATE INDEX missing_index_' + CONVERT (VARCHAR, mig.index_group_handle) + '_' + CONVERT (VARCHAR, mid.index_handle) 
                + ' ON ' + mid.statement + ' (' + ISNULL (mid.equality_columns,
                '') + CASE WHEN mid.equality_columns IS NOT NULL
                            AND mid.inequality_columns IS NOT NULL THEN ','
                        ELSE ''
                      END + ISNULL (mid.inequality_columns,
                    '') + ')' + ISNULL (' INCLUDE (' + mid.included_columns + ')',
                    '') AS create_index_statement,
        migs.*,
        mid.database_id,
        mid.[object_id]
    FROM sys.dm_db_missing_index_groups mig
    INNER JOIN sys.dm_db_missing_index_group_stats migs ON migs.group_handle = mig.index_group_handle
    INNER JOIN sys.dm_db_missing_index_details mid ON mig.index_handle = mid.index_handle
    WHERE CONVERT (DECIMAL (28, 1),
                   migs.avg_total_user_cost * migs.avg_user_impact * (migs.user_seeks + migs.user_scans)) > 10
    ORDER BY migs.avg_total_user_cost * migs.avg_user_impact * (migs.user_seeks + migs.user_scans) DESC
    ```

## Step 5: Investigate parameter-sensitive issues

Use the [DBCC FREEPROCCACHE](/sql/t-sql/database-console-commands/dbcc-freeproccache-transact-sql) command to check whether the high-CPU-usage issue is fixed.

If the issue still exists, you can add a `RECOMPILE` query hint to each of the high CPU queries that are identified in [step 2](#step-2-identify-queries-contributing-to-cpu-usage).

If the issue is fixed, it's an indication of parameter-sensitive problem (PSP, aka "parameter sniffing issue"). To mitigate the parameter-sensitive issues, use the following methods. Each method has associated tradeoffs and drawbacks.

- Use the [RECOMPILE](/sql/t-sql/queries/hints-transact-sql-query#recompile) query hint at each query execution. This workaround balances compilation time and increased CPU for better plan quality. Here is an example of how you can apply this to your query.

  ```sql
  SELECT * FROM Person.Person 
  WHERE LastName = 'Wood'
  OPTION (RECOMPILE)
  ```

- Use the [option (OPTIMIZE FOR…)](/sql/t-sql/queries/hints-transact-sql-query#optimize-for--variable_name--unknown---literal_constant-_---n--) query hint to override the actual parameter value with a typical parameter value that produces a plan that's good enough for most parameter value possibilities. This option requires a full understanding of optimal parameter values and associated plan characteristics. Here is an example how to use this hint in your query.

  ```sql
  DECLARE @LastName Name = 'Frintu'
  SELECT FirstName, LastName FROM Person.Person 
  WHERE LastName = @LastName
  OPTION (OPTIMIZE FOR (@LastName = 'Wood'))
  ```

- Use the [option (OPTIMIZE FOR UNKNOWN)](/sql/t-sql/queries/hints-transact-sql-query#optimize-for-unknown) query hint to override the actual parameter value with density vector average. You can also do this by capturing the incoming parameter values in local variables, and then using the local variables within the predicates instead of using the parameters themselves. For this fix, the average density must be good enough.

- Use the [DISABLE_PARAMETER_SNIFFING](/sql/t-sql/queries/hints-transact-sql-query#use_hint) query hint to disable parameter sniffing entirely. Here is an example of how to use it in a query:

  ```sql
  SELECT * FROM Person.Address  
  WHERE City = 'SEATTLE' AND PostalCode = 98104
  OPTION (USE HINT ('DISABLE_PARAMETER_SNIFFING'))
  ```

- Use the [KEEPFIXED PLAN](/sql/t-sql/queries/hints-transact-sql-query#keepfixed-plan) query hint to prevent recompilations in cache. This workaround assumes that the "good enough" common plan is the one that's already in cache. You can also disable automatic statistics updates to reduce the chances that the good plan will be evicted and a new bad plan will be compiled.

- Using the [DBCC FREEPROCCACHE](/sql/t-sql/database-console-commands/dbcc-freeproccache-transact-sql) command is a temporary solution until the application code is fixed. You can use `DBCC FREEPROCCACHE (plan_handle)` command to remove only the plan that is causing the issue. For example, to find query plans that reference the Person.Person table in AdventureWorks, you can use this query to look up the query handle. Then you can release the specific query plan from cache by using the `DBCC FREEPROCCACHE (plan_handle)` that is produced in the second column of the query result.

  ```sql
  SELECT text, 'DBCC FREEPROCCACHE (0x' + CONVERT(VARCHAR (512), plan_handle, 2) + ')' AS dbcc_freeproc_command FROM sys.dm_exec_cached_plans
  CROSS APPLY sys.dm_exec_query_plan(plan_handle)
  CROSS APPLY sys.dm_exec_sql_text(plan_handle)
  WHERE text LIKE '%person.person%'
  ```

## Step 6: Disable heavy tracing

Check for [SQL Trace](/sql/relational-databases/sql-trace/sql-trace) or XEvent tracing that affects SQL Server performance and causes high CPU usage. For example, the events are SQL Audit, events cause high XML plans, statement event level events, log-in and log-out operations, locks, and waits.

Run the following queries to identify active XEvent or Server traces:

```sql
PRINT '--Profiler trace summary--'
SELECT traceid, property, CONVERT (VARCHAR(1024), value) AS value FROM :: fn_trace_getinfo(default)
GO
PRINT '--Trace event details--'
      SELECT trace_id,
            status,
            CASE WHEN row_number = 1 THEN path ELSE NULL end AS path,
            CASE WHEN row_number = 1 THEN max_size ELSE NULL end AS max_size,
            CASE WHEN row_number = 1 THEN start_time ELSE NULL end AS start_time,
            CASE WHEN row_number = 1 THEN stop_time ELSE NULL end AS stop_time,
            max_files, 
            is_rowset, 
            is_rollover,
            is_shutdown,
            is_default,
            buffer_count,
            buffer_size,
            last_event_time,
            event_count,
            trace_event_id, 
            trace_event_name, 
            trace_column_id,
            trace_column_name,
            expensive_event   
      FROM 
            (SELECT t.id AS trace_id, 
                  row_number() over (PARTITION BY t.id order by te.trace_event_id, tc.trace_column_id) AS row_number, 
                  t.status, 
                  t.path, 
                  t.max_size, 
                  t.start_time,
                  t.stop_time, 
                  t.max_files, 
                  t.is_rowset, 
                  t.is_rollover,
                  t.is_shutdown,
                  t.is_default,
                  t.buffer_count,
                  t.buffer_size,
                  t.last_event_time,
                  t.event_count,
                  te.trace_event_id, 
                  te.name AS trace_event_name, 
                  tc.trace_column_id,
                  tc.name AS trace_column_name,
                  CASE WHEN te.trace_event_id in (23, 24, 40, 41, 44, 45, 51, 52, 54, 68, 96, 97, 98, 113, 114, 122, 146, 180) 
                  THEN CAST(1 as bit) ELSE CAST(0 AS BIT) END AS expensive_event
            FROM sys.traces t 
                  CROSS APPLY ::fn_trace_geteventinfo(t .id) AS e 
                  JOIN sys.trace_events te ON te.trace_event_id = e.eventid 
                  JOIN sys.trace_columns tc ON e.columnid = trace_column_id) AS x
GO
PRINT '--XEvent Session Details--'
SELECT sess.NAME 'session_name', event_name,xe_event_name, trace_event_id,
CASE
 WHEN xemap.trace_event_id IN ( 23, 24, 40, 41, 44, 45, 51, 52, 54, 68, 96, 97, 98, 113, 114, 122, 146, 180 )
 THEN Cast(1 AS BIT) ELSE Cast(0 AS BIT)
END AS expensive_event
FROM sys.dm_xe_sessions sess
  JOIN sys.dm_xe_session_events evt
  ON sess.address = evt.event_session_address
INNER JOIN sys.trace_xe_event_map xemap
  ON evt.event_name = xemap.xe_event_name
GO
```

## Step 7: Fix SOS_CACHESTORE spinlock contention

If your SQL Server experiences heavy `SOS_CACHESTORE spinlock` contention or you notice that your query plans are often evicted on ad hoc query workloads, review the following article and enable trace flag T174 by using the `DBCC TRACEON (174, -1)` command. If the high-CPU condition is resolved by using T174, enable it as a [startup parameter](/sql/tools/configuration-manager/sql-server-properties-startup-parameters-tab) by using SQL Server Configuration Manager.

[FIX: SOS_CACHESTORE spinlock contention on ad hoc SQL Server plan cache causes high CPU usage in SQL Server](https://support.microsoft.com/topic/kb3026083-fix-sos-cachestore-spinlock-contention-on-ad-hoc-sql-server-plan-cache-causes-high-cpu-usage-in-sql-server-798ca4a5-3813-a3d2-f9c4-89eb1128fe68).

## Step 8: Configure your virtual machine

If you are using a virtual machine, make sure that you aren't overprovisioning CPUs and that they are configured correctly. For more information, see [Troubleshooting ESX/ESXi virtual machine performance issues (2001003)](https://kb.vmware.com/s/article/2001003#CPU%20constraints).

## Step 9: Scale up SQL Server

If individual query instances are using little CPU, but the overall workload of all queries together causes high CPU consumption, consider scaling up your computer by adding more CPUs. Use the following query to help you determine how many queries have exceeded a certain threshold of average and maximum CPU consumption per execution and have executed many times on the system. Be sure to modify the values of the two variables to match your environment.

```sql
-- Shows queries where Max and average CPU time exceeds 200 ms and executed more than 1000 times
DECLARE @cputime_threshold_microsec INT = 200*1000
DECLARE @execution_count INT = 1000

SELECT 
     qs.total_worker_time/1000 total_cpu_time_ms,
       qs.max_worker_time/1000 max_cpu_time_ms,
       (qs.total_worker_time/1000)/execution_count average_cpu_time_ms,
       qs.execution_count,
     q.[text]
FROM
   sys.dm_exec_query_stats qs
   CROSS APPLY sys.dm_exec_sql_text(plan_handle) AS q
WHERE  (qs.total_worker_time/execution_count > @cputime_threshold_microsec  
       OR        qs.max_worker_time > @cputime_threshold_microsec )
       AND execution_count > @execution_count
ORDER BY qs.total_worker_time DESC 
OPTION (RECOMPILE)
```

## See also

- [High CPU or memory grants may occur with queries that use optimized nested loop or batch sort](decreased-perf-high-cpu-optimized-nested-loop.md)
- [Updates and configuration for workloads](recommended-updates-configuration-options.md) (SQL Server 2014 and SQL Server 2012 versions)
- [Recommended updates and configuration options for SQL Server 2016 and later versions with high-performance workloads](recommended-updates-configuration-workloads.md)