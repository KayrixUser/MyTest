## Validation (DEV)

### Execution results

- The user confirmed applying all three revised view definitions.
- Reviewed cATO and TLS checks passed for column layout, types and tested data conversions.
- Earlier scoped model and calculation comparisons passed.
- Recorded nullable-metadata differences were accepted; this does not mean NULL data is ignored.
- DEV model data can be older than the daily ACP feed, so different snapshot counts alone do not indicate a failed migration.

### Email tests

- The missed-update email test remains open.
- No successful email-delivery result is claimed for that test.
- PROD deployment and post-deployment validation remain planned.

## Production Sequence & Actions

The reviewed `C2S:Scheduled_Tasks_Daily_Z` job starts at **04:00 on the SQL Server schedule clock**. Keep its existing order:

1. Main C2S model refresh.
2. Asset calculations.
3. SAP calculations.
4. Base monitoring logs.
5. Base notification email.
6. Dimensions notification email.

Complete both ACP loads before this sequence. Do not copy Splunk's history-job schedule into the Verdant process.

## Object Reference — Quick Lookup

| Purpose | Objects |
| --- | --- |
| cATO model output | `dbo.tbl_C2Sreport_cATO`, `dbo.dimVerdant` |
| TLS model output | `dbo.tbl_C2Sreport_TLSVerdant`, `dbo.dimTLSVerdant` |
| Monitoring records | `dbo.BaseViewCheckLogs` |
| Procedure execution records | `dbo.ScheduledTasksLogs` |
| Retained legacy cATO table | `PA499.cATO_Summary` |
| Retained legacy loader | `dbo.REFRESH_PA499_Weekly` |

## Promotion Order

- [ ] Complete the pending missed-update email test and record its result.
- [ ] Save current PROD definitions, job settings and the rollback steps.
- [ ] Confirm completed ACP loads and access for the PROD execution account.
- [ ] Apply the reviewed changes using `USE [C2S]`; keep `ACP_Connection` as the source database.
- [ ] Refresh the dependent TLS view metadata.
- [ ] Stop PA499 refresh calls only after the supported readers have moved.
- [ ] Run one coordinated model refresh through the existing job; avoid also running its child procedures separately.
- [ ] Verify model outputs, monitoring timestamps, job failure behavior, notifications and the affected Power BI results.

## Retirement & Rollback

| Component | Decision |
| --- | --- |
| `PA499.cATO_Summary` and `dbo.REFRESH_PA499_Weekly` | Retain for rollback during this release. They are later cleanup candidates after successful operation, dependency checks and closure of the rollback window. |
| PD034 refresh path and associated daily/weekly wrappers/jobs | Retirement is on hold. No deletion or job disabling is included here. |
| Entire `PA499` schema | No schema-wide deletion. |
| Shared logs and active model objects | Keep. |

Retaining an unused table or procedure does not run it automatically. The PA499 table becomes stale once its refresh stops; rollback must restore the saved definitions and confirm usable legacy data before reporting resumes.

## Data Flow Diagram

```mermaid
flowchart TD
    S["Snowflake Verdant views"] --> M["Daily Matillion loads"]
    M --> A["ACP cATO landing table"]
    M --> B["ACP TLS landing table"]
    A --> C["cATO source and reporting views"]
    B --> T["TLS source and reporting views"]
    C --> R["C2S model refresh and calculations"]
    T --> R
    R --> P["Power BI outputs"]
    A --> L["BaseViewCheckLogs"]
    B --> L
    R -->|"Model checks"| L
    L --> E["Existing notification emails"]



---------------------------------------
USE [C2S];
GO
SET NOCOUNT ON;

SELECT CONVERT(nvarchar(128), SERVERPROPERTY('ServerName')) AS ServerName,
       DB_NAME() AS DatabaseName, s.name AS SchemaName,
       o.name AS ObjectName, o.type_desc AS ObjectType
FROM sys.objects AS o
JOIN sys.schemas AS s ON s.schema_id = o.schema_id
WHERE (s.name = N'dbo' AND o.name = N'REFRESH_PA499_Weekly')
   OR (s.name = N'PA499' AND o.name = N'cATO_Summary');

DROP TABLE IF EXISTS #PA499Coverage;
DROP TABLE IF EXISTS #PA499References;
CREATE TABLE #PA499Coverage
(
    ScopeName nvarchar(128), Status nvarchar(20), Detail nvarchar(4000)
);
CREATE TABLE #PA499References
(
    DatabaseName nvarchar(128), MatchType nvarchar(40),
    ObjectName nvarchar(776), Detail nvarchar(max)
);

IF ISNULL(HAS_PERMS_BY_NAME(NULL, NULL, N'VIEW ANY DATABASE'), 0) <> 1
    INSERT #PA499Coverage VALUES
    (N'Server', N'REVIEW', N'Cannot confirm visibility of every database. DBA review required.');

INSERT #PA499Coverage
SELECT name, N'NOT SCANNED',
       N'Database state: ' + state_desc + N'; inaccessible or offline.'
FROM sys.databases
WHERE name <> N'tempdb'
  AND (state <> 0 OR ISNULL(HAS_DBACCESS(name), 0) <> 1);

DECLARE @AuditDb sysname, @AuditSql nvarchar(max);
DECLARE PA499Databases CURSOR LOCAL FAST_FORWARD FOR
SELECT name FROM sys.databases
WHERE name <> N'tempdb' AND state = 0 AND HAS_DBACCESS(name) = 1;
OPEN PA499Databases;
FETCH NEXT FROM PA499Databases INTO @AuditDb;
WHILE @@FETCH_STATUS = 0
BEGIN
    SET @AuditSql = N'USE ' + QUOTENAME(@AuditDb) + N';
    INSERT #PA499Coverage
    SELECT DB_NAME(),
           CASE WHEN HAS_PERMS_BY_NAME(DB_NAME(), N''DATABASE'', N''VIEW DEFINITION'') = 1
                THEN N''SCANNED'' ELSE N''PARTIAL'' END,
           CASE WHEN HAS_PERMS_BY_NAME(DB_NAME(), N''DATABASE'', N''VIEW DEFINITION'') = 1
                THEN N''Metadata scan completed; text matches include comments.''
                ELSE N''Database-wide VIEW DEFINITION is not confirmed; results may be incomplete.'' END;

    INSERT #PA499References
    SELECT DB_NAME(), N''SQL text'',
           COALESCE(QUOTENAME(OBJECT_SCHEMA_NAME(m.object_id)) + N''.'', N'''') + QUOTENAME(o.name),
           m.definition
    FROM sys.sql_modules AS m
    JOIN sys.objects AS o ON o.object_id = m.object_id
    WHERE o.is_ms_shipped = 0
      AND m.definition COLLATE Latin1_General_100_CI_AS LIKE N''%PA499%'';

    INSERT #PA499Coverage
    SELECT DB_NAME(), N''UNREADABLE'',
           N''Cannot inspect SQL definition for '' +
           COALESCE(QUOTENAME(OBJECT_SCHEMA_NAME(m.object_id)) + N''.'', N'''') + QUOTENAME(o.name)
    FROM sys.sql_modules AS m
    JOIN sys.objects AS o ON o.object_id = m.object_id
    WHERE o.is_ms_shipped = 0 AND m.definition IS NULL;

    INSERT #PA499References
    SELECT DB_NAME(), N''Catalog dependency'',
           COALESCE(QUOTENAME(OBJECT_SCHEMA_NAME(d.referencing_id)) + N''.'', N'''') +
           COALESCE(QUOTENAME(OBJECT_NAME(d.referencing_id)), N''<unresolved entity>''),
           COALESCE(QUOTENAME(d.referenced_server_name) + N''.'', N'''') +
           COALESCE(QUOTENAME(d.referenced_database_name) + N''.'', N'''') +
           COALESCE(QUOTENAME(d.referenced_schema_name) + N''.'', N'''') +
           COALESCE(QUOTENAME(d.referenced_entity_name), N''<unresolved target>'')
    FROM sys.sql_expression_dependencies AS d
    WHERE d.referenced_schema_name COLLATE Latin1_General_100_CI_AS = N''PA499''
       OR d.referenced_entity_name COLLATE Latin1_General_100_CI_AS = N''REFRESH_PA499_Weekly'';

    INSERT #PA499References
    SELECT DB_NAME(), N''Synonym'',
           QUOTENAME(SCHEMA_NAME(schema_id)) + N''.'' + QUOTENAME(name), base_object_name
    FROM sys.synonyms
    WHERE base_object_name COLLATE Latin1_General_100_CI_AS LIKE N''%PA499%'';';
    BEGIN TRY
        EXEC sys.sp_executesql @AuditSql;
    END TRY
    BEGIN CATCH
        INSERT #PA499Coverage VALUES (@AuditDb, N'ERROR', ERROR_MESSAGE());
    END CATCH;
    FETCH NEXT FROM PA499Databases INTO @AuditDb;
END;
CLOSE PA499Databases;
DEALLOCATE PA499Databases;

BEGIN TRY
    INSERT #PA499References
    SELECT N'msdb', N'Agent job step',
           j.name + N' / step ' + CONVERT(nvarchar(12), js.step_id) + N' / ' + js.step_name,
           N'Enabled=' + CONVERT(nvarchar(1), j.enabled) +
           N'; database=' + COALESCE(js.database_name, N'<none>') +
           N'; subsystem=' + js.subsystem + NCHAR(13) + NCHAR(10) + js.command
    FROM msdb.dbo.sysjobs AS j
    JOIN msdb.dbo.sysjobsteps AS js ON js.job_id = j.job_id
    WHERE js.command COLLATE Latin1_General_100_CI_AS LIKE N'%PA499%';
    INSERT #PA499Coverage VALUES
    (N'SQL Agent', N'SCANNED', N'Job-step commands searched; external script/package contents are not inspected.');
END TRY
BEGIN CATCH
    INSERT #PA499Coverage VALUES (N'SQL Agent', N'ERROR', ERROR_MESSAGE());
END CATCH;

IF ISNULL(IS_SRVROLEMEMBER(N'sysadmin'), 0) <> 1
    INSERT #PA499Coverage VALUES
    (N'SQL Agent', N'REVIEW', N'Non-sysadmin session: DBA must confirm that results cover all jobs.');

BEGIN TRY
    INSERT #PA499References
    SELECT N'<server>', N'Server trigger SQL', t.name, m.definition
    FROM sys.server_sql_modules AS m
    JOIN sys.server_triggers AS t ON t.object_id = m.object_id
    WHERE m.definition COLLATE Latin1_General_100_CI_AS LIKE N'%PA499%';
    IF ISNULL(HAS_PERMS_BY_NAME(NULL, NULL, N'VIEW ANY DEFINITION'), 0) <> 1
        INSERT #PA499Coverage VALUES
        (N'Server triggers', N'REVIEW', N'Server-wide definition visibility is not confirmed.');
END TRY
BEGIN CATCH
    INSERT #PA499Coverage VALUES (N'Server triggers', N'ERROR', ERROR_MESSAGE());
END CATCH;

SELECT ScopeName, Status, Detail
FROM #PA499Coverage
ORDER BY CASE WHEN Status = N'SCANNED' THEN 1 ELSE 0 END, ScopeName;

SELECT DatabaseName, MatchType, ObjectName,
       CASE WHEN DatabaseName = N'C2S' AND ObjectName = N'[dbo].[REFRESH_PA499_Weekly]'
            THEN N'Expected legacy loader; remove only with its legacy table after approval.'
            ELSE N'Review active SQL versus comments and whether it targets this C2S pair.' END AS ReviewNote,
       Detail
FROM #PA499References
ORDER BY DatabaseName, ObjectName, MatchType;

PRINT N'Audit complete. Nothing was removed. Missing/partial access is not a clean result.';
PRINT N'Before removal: preserve a restorable copy of the table data and object definitions/permissions; resolve active consumers and confirm external consumers with the owner.';
