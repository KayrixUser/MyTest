USE [C2S];
GO

-- 1. Context and visibility: retain this result with the inventory.
SELECT
    CONVERT(nvarchar(128), SERVERPROPERTY('ServerName')) AS ServerName,
    DB_NAME() AS DatabaseName,
    HAS_PERMS_BY_NAME(DB_NAME(), 'DATABASE', 'VIEW DEFINITION') AS CanViewDatabaseDefinitions,
    OBJECT_ID(N'dbo.cATO_Summary', N'V') AS WrapperObjectId,
    IS_SRVROLEMEMBER(N'sysadmin') AS IsSysadmin;

-- 2. Recorded wrapper dependencies plus broader cATO text candidates.
SELECT
    s.name AS SchemaName,
    o.name AS ObjectName,
    o.type_desc AS ObjectType,
    CASE WHEN EXISTS (
        SELECT 1
        FROM sys.sql_expression_dependencies AS d
        WHERE d.referencing_id = o.object_id
          AND d.referenced_id = OBJECT_ID(N'dbo.cATO_Summary', N'V')
    ) THEN N'Recorded wrapper dependency'
      ELSE N'Text candidate - review active code' END AS MatchType
FROM sys.sql_modules AS m
JOIN sys.objects AS o ON o.object_id = m.object_id
JOIN sys.schemas AS s ON s.schema_id = o.schema_id
WHERE o.is_ms_shipped = 0
  AND o.object_id <> ISNULL(OBJECT_ID(N'dbo.cATO_Summary', N'V'), -1)
  AND (
      LOWER(m.definition) LIKE N'%cato_summary%'
      OR EXISTS (
          SELECT 1
          FROM sys.sql_expression_dependencies AS d
          WHERE d.referencing_id = o.object_id
            AND d.referenced_id = OBJECT_ID(N'dbo.cATO_Summary', N'V')
      )
  )
ORDER BY s.name, o.name;

-- 3. Synonyms with cATO target names in C2S.
SELECT
    s.name AS SchemaName,
    sy.name AS SynonymName,
    sy.base_object_name AS TargetObject
FROM sys.synonyms AS sy
JOIN sys.schemas AS s ON s.schema_id = sy.schema_id
WHERE LOWER(sy.base_object_name) LIKE N'%cato_summary%'
ORDER BY s.name, sy.name;

-- 4. Direct cATO/loader references in visible SQL Agent job steps.
-- Existing Weekly -> Weekly_A call chain is already recorded separately.
SELECT
    j.name AS JobName,
    j.enabled AS JobEnabled,
    st.step_id AS StepNumber,
    st.step_name AS StepName,
    st.subsystem AS Subsystem,
    st.database_name AS StepDatabase,
    st.command AS StepCommand
FROM msdb.dbo.sysjobs AS j
JOIN msdb.dbo.sysjobsteps AS st ON st.job_id = j.job_id
WHERE LOWER(st.command) LIKE N'%cato_summary%'
   OR LOWER(st.command) LIKE N'%refresh_pa499_weekly%'
ORDER BY j.name, st.step_id;
