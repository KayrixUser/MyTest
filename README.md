
USE [C2S];
GO

-- 1. Metadata visibility (1 = granted; 0/NULL means coverage needs review).
SELECT N'1 - Metadata visibility' AS Check_Name,
       N'C2S' AS Database_Name,
       HAS_DBACCESS(N'C2S') AS Has_DB_Access,
       HAS_PERMS_BY_NAME(N'C2S', N'DATABASE', N'VIEW DEFINITION') AS Has_View_Definition,
       (SELECT COUNT_BIG(*) FROM [C2S].sys.sql_modules) AS Visible_SQL_Modules,
       (SELECT COUNT_BIG(*) FROM [C2S].sys.sql_modules WHERE definition IS NULL) AS Unreadable_Definitions
UNION ALL
SELECT N'1 - Metadata visibility', N'ACP_Connection',
       HAS_DBACCESS(N'ACP_Connection'),
       HAS_PERMS_BY_NAME(N'ACP_Connection', N'DATABASE', N'VIEW DEFINITION'),
       (SELECT COUNT_BIG(*) FROM [ACP_Connection].sys.sql_modules),
       (SELECT COUNT_BIG(*) FROM [ACP_Connection].sys.sql_modules WHERE definition IS NULL);
GO

-- 2. All seven observed ACP Archer TABLE schemas. No business rows are read.
SELECT N'2 - ACP table columns' AS Check_Name,
       N'ACP_Connection' AS Database_Name,
       s.name AS Schema_Name, t.name AS Table_Name,
       c.column_id AS Column_Order, c.name AS Column_Name,
       ty.name AS Data_Type, c.max_length AS Max_Length_Bytes,
       c.precision AS Numeric_Precision, c.scale AS Numeric_Scale,
       c.is_nullable, c.is_identity, c.is_computed, c.collation_name
FROM [ACP_Connection].sys.tables AS t
JOIN [ACP_Connection].sys.schemas AS s ON s.schema_id = t.schema_id
JOIN [ACP_Connection].sys.columns AS c ON c.object_id = t.object_id
JOIN [ACP_Connection].sys.types AS ty ON ty.user_type_id = c.user_type_id
WHERE t.name IN (
    N'ArcherMetrics_AuthorizationPackage_PROD',
    N'ArcherMetrics_AuthorizationPackage_PROD_All',
    N'ArcherMetrics_Hardware_PROD',
    N'ArcherMetrics_Hardware_PROD_All',
    N'ArcherMetrics_POAMs_PROD',
    N'ArcherMetrics_POAMs_PROD_All',
    N'ArcherMetrics_GlobalTrade_PROD'
)
ORDER BY s.name, t.name, c.column_id;
GO

-- 3. Candidate modules: Archer/table references OR remote-access references.
-- Includes procedures/views/functions/triggers, readers and possible writers.
-- Inspect the returned definition through SSMS to establish actual INSERT,
-- SELECT INTO, UPDATE, MERGE, BULK INSERT or call-chain behavior.
WITH Modules AS (
    SELECT N'C2S' AS Database_Name,
           s.name COLLATE DATABASE_DEFAULT AS Schema_Name,
           o.name COLLATE DATABASE_DEFAULT AS Object_Name,
           o.type_desc COLLATE DATABASE_DEFAULT AS Object_Type,
           m.definition COLLATE DATABASE_DEFAULT AS Definition
    FROM [C2S].sys.sql_modules AS m
    JOIN [C2S].sys.objects AS o ON o.object_id = m.object_id
    JOIN [C2S].sys.schemas AS s ON s.schema_id = o.schema_id
    UNION ALL
    SELECT N'ACP_Connection', s.name COLLATE DATABASE_DEFAULT,
           o.name COLLATE DATABASE_DEFAULT, o.type_desc COLLATE DATABASE_DEFAULT,
           m.definition COLLATE DATABASE_DEFAULT
    FROM [ACP_Connection].sys.sql_modules AS m
    JOIN [ACP_Connection].sys.objects AS o ON o.object_id = m.object_id
    JOIN [ACP_Connection].sys.schemas AS s ON s.schema_id = o.schema_id
)
SELECT N'3 - Module candidates' AS Check_Name,
       m.Database_Name, m.Schema_Name, m.Object_Name, m.Object_Type,
       CASE WHEN UPPER(m.Definition) LIKE N'%ARCHER%' THEN 1 ELSE 0 END AS Mentions_Archer,
       CASE WHEN UPPER(m.Definition) LIKE N'%ACP[_]CONNECTION%' THEN 1 ELSE 0 END AS Mentions_ACP,
       CASE WHEN UPPER(m.Definition) LIKE N'%OPENQUERY%'
                  OR UPPER(m.Definition) LIKE N'%OPENROWSET%'
                  OR UPPER(m.Definition) LIKE N'%OPENDATASOURCE%'
            THEN 1 ELSE 0 END AS Remote_Keyword_Match,
       r.Linked_Server_Name_Match
FROM Modules AS m
OUTER APPLY (
    SELECT TOP (1) ls.name AS Linked_Server_Name_Match
    FROM sys.servers AS ls
    WHERE ls.is_linked = 1
      AND CHARINDEX(UPPER(ls.name COLLATE DATABASE_DEFAULT), UPPER(m.Definition)) > 0
    ORDER BY ls.name
) AS r
WHERE UPPER(m.Definition) LIKE N'%ARCHER%'
   OR UPPER(m.Definition) LIKE N'%ACP[_]CONNECTION%'
   OR UPPER(m.Definition) LIKE N'%OPENQUERY%'
   OR UPPER(m.Definition) LIKE N'%OPENROWSET%'
   OR UPPER(m.Definition) LIKE N'%OPENDATASOURCE%'
   OR r.Linked_Server_Name_Match IS NOT NULL
ORDER BY m.Database_Name, m.Object_Type, m.Schema_Name, m.Object_Name;
GO

-- 4. Recorded dependencies. Cross-server names may be present even without
-- a resolved object ID. This complements, but does not replace, text search.
WITH Dependencies AS (
    SELECT N'C2S' AS Database_Name,
           s.name COLLATE DATABASE_DEFAULT AS Schema_Name,
           o.name COLLATE DATABASE_DEFAULT AS Object_Name,
           d.referenced_server_name COLLATE DATABASE_DEFAULT AS Referenced_Server,
           d.referenced_database_name COLLATE DATABASE_DEFAULT AS Referenced_Database,
           d.referenced_schema_name COLLATE DATABASE_DEFAULT AS Referenced_Schema,
           d.referenced_entity_name COLLATE DATABASE_DEFAULT AS Referenced_Object
    FROM [C2S].sys.sql_expression_dependencies AS d
    JOIN [C2S].sys.objects AS o ON o.object_id = d.referencing_id
    JOIN [C2S].sys.schemas AS s ON s.schema_id = o.schema_id
    WHERE d.referencing_class = 1
    UNION ALL
    SELECT N'ACP_Connection', s.name COLLATE DATABASE_DEFAULT,
           o.name COLLATE DATABASE_DEFAULT,
           d.referenced_server_name COLLATE DATABASE_DEFAULT,
           d.referenced_database_name COLLATE DATABASE_DEFAULT,
           d.referenced_schema_name COLLATE DATABASE_DEFAULT,
           d.referenced_entity_name COLLATE DATABASE_DEFAULT
    FROM [ACP_Connection].sys.sql_expression_dependencies AS d
    JOIN [ACP_Connection].sys.objects AS o ON o.object_id = d.referencing_id
    JOIN [ACP_Connection].sys.schemas AS s ON s.schema_id = o.schema_id
    WHERE d.referencing_class = 1
)
SELECT DISTINCT N'4 - Recorded dependencies' AS Check_Name, *
FROM Dependencies
WHERE Referenced_Server IS NOT NULL
   OR UPPER(Referenced_Database) = N'ACP_CONNECTION'
   OR UPPER(Referenced_Object) LIKE N'%ARCHER%'
   OR UPPER(Object_Name) LIKE N'%ARCHER%'
ORDER BY Database_Name, Schema_Name, Object_Name, Referenced_Server,
         Referenced_Database, Referenced_Schema, Referenced_Object;
GO

-- 5. Synonyms may conceal a local, cross-database or linked-server target.
SELECT N'5 - Synonyms' AS Check_Name, N'C2S' AS Database_Name,
       s.name COLLATE DATABASE_DEFAULT AS Schema_Name,
       sn.name COLLATE DATABASE_DEFAULT AS Synonym_Name,
       sn.base_object_name COLLATE DATABASE_DEFAULT AS Base_Object_Name
FROM [C2S].sys.synonyms AS sn
JOIN [C2S].sys.schemas AS s ON s.schema_id = sn.schema_id
UNION ALL
SELECT N'5 - Synonyms', N'ACP_Connection',
       s.name COLLATE DATABASE_DEFAULT, sn.name COLLATE DATABASE_DEFAULT,
       sn.base_object_name COLLATE DATABASE_DEFAULT
FROM [ACP_Connection].sys.synonyms AS sn
JOIN [ACP_Connection].sys.schemas AS s ON s.schema_id = sn.schema_id
ORDER BY Database_Name, Schema_Name, Synonym_Name;
GO

-- 6. Linked-server registrations only. Does not connect to remote servers.
-- Deliberately excludes provider strings and login mappings.
SELECT N'6 - Linked servers' AS Check_Name,
       name AS Linked_Server_Name, product, provider, data_source, catalog,
       is_data_access_enabled, is_rpc_out_enabled
FROM sys.servers
WHERE is_linked = 1
ORDER BY name;
GO

-- 7. Job candidates: search commands internally, output identifiers only.
-- External execution steps are included because package/script names can
-- conceal Archer loading. This does not execute jobs or export their commands.
SELECT N'7 - Agent job candidates' AS Check_Name,
       j.name AS Job_Name, j.enabled AS Job_Enabled,
       st.step_id AS Step_ID, st.step_name AS Step_Name,
       st.subsystem AS Step_Type, st.database_name AS Step_Database,
       CASE WHEN UPPER(st.command) LIKE N'%ARCHER%'
                  OR UPPER(j.name) LIKE N'%ARCHER%'
            THEN 1 ELSE 0 END AS Archer_Match,
       CASE WHEN UPPER(st.command) LIKE N'%ACP[_]CONNECTION%'
                  OR UPPER(st.database_name) = N'ACP_CONNECTION'
            THEN 1 ELSE 0 END AS ACP_Match,
       r.Linked_Server_Name_Match,
       CASE WHEN st.subsystem IN (N'SSIS', N'CmdExec', N'PowerShell')
            THEN 1 ELSE 0 END AS External_Execution_Step
FROM msdb.dbo.sysjobs AS j
JOIN msdb.dbo.sysjobsteps AS st ON st.job_id = j.job_id
OUTER APPLY (
    SELECT TOP (1) ls.name AS Linked_Server_Name_Match
    FROM sys.servers AS ls
    WHERE ls.is_linked = 1
      AND CHARINDEX(UPPER(ls.name COLLATE DATABASE_DEFAULT),
                    UPPER(st.command COLLATE DATABASE_DEFAULT)) > 0
    ORDER BY ls.name
) AS r
WHERE UPPER(st.command) LIKE N'%ARCHER%'
   OR UPPER(j.name) LIKE N'%ARCHER%'
   OR UPPER(st.command) LIKE N'%ACP[_]CONNECTION%'
   OR UPPER(st.database_name) = N'ACP_CONNECTION'
   OR UPPER(st.command) LIKE N'%OPENQUERY%'
   OR UPPER(st.command) LIKE N'%OPENROWSET%'
   OR UPPER(st.command) LIKE N'%OPENDATASOURCE%'
   OR r.Linked_Server_Name_Match IS NOT NULL
   OR st.subsystem IN (N'SSIS', N'CmdExec', N'PowerShell')
ORDER BY j.name, st.step_id;
GO






