
USE [C2S_Development];
GO

;WITH Modules AS
(
    SELECT N'ACP_Connection' AS DatabaseName,
           s.name COLLATE DATABASE_DEFAULT AS SchemaName,
           o.name COLLATE DATABASE_DEFAULT AS ObjectName,
           o.type_desc COLLATE DATABASE_DEFAULT AS ObjectType,
           m.definition COLLATE DATABASE_DEFAULT AS SqlDefinition
    FROM [ACP_Connection].sys.sql_modules AS m
    JOIN [ACP_Connection].sys.objects AS o ON o.object_id = m.object_id
    JOIN [ACP_Connection].sys.schemas AS s ON s.schema_id = o.schema_id

    UNION ALL

    SELECT N'C2S', s.name COLLATE DATABASE_DEFAULT,
           o.name COLLATE DATABASE_DEFAULT, o.type_desc COLLATE DATABASE_DEFAULT,
           m.definition COLLATE DATABASE_DEFAULT
    FROM [C2S].sys.sql_modules AS m
    JOIN [C2S].sys.objects AS o ON o.object_id = m.object_id
    JOIN [C2S].sys.schemas AS s ON s.schema_id = o.schema_id

    UNION ALL

    SELECT N'C2S_Development', s.name COLLATE DATABASE_DEFAULT,
           o.name COLLATE DATABASE_DEFAULT, o.type_desc COLLATE DATABASE_DEFAULT,
           m.definition COLLATE DATABASE_DEFAULT
    FROM [C2S_Development].sys.sql_modules AS m
    JOIN [C2S_Development].sys.objects AS o ON o.object_id = m.object_id
    JOIN [C2S_Development].sys.schemas AS s ON s.schema_id = o.schema_id
)
SELECT DatabaseName, SchemaName, ObjectName, ObjectType,
       CASE WHEN CHARINDEX(N'LOG_TYPE_COMPLIANT', UPPER(SqlDefinition)) > 0
            THEN 1 ELSE 0 END AS Mentions_Log_Type_Compliant,
       CASE WHEN CHARINDEX(N'SPLUNK_SOURCE', UPPER(SqlDefinition)) > 0
            THEN 1 ELSE 0 END AS Mentions_Splunk_Source
FROM Modules
WHERE CHARINDEX(N'LOG_TYPE_COMPLIANT', UPPER(SqlDefinition)) > 0
   OR CHARINDEX(N'SPLUNK_SOURCE', UPPER(SqlDefinition)) > 0
ORDER BY DatabaseName, SchemaName, ObjectName;
