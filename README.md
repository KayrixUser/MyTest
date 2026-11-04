USE [C2S_Development];
GO

-- 1. All three views should be visible in both databases.
SELECT
    @@SERVERNAME AS ServerName,
    DB_NAME() AS ExecutionDatabase,
    IS_SRVROLEMEMBER(N'sysadmin') AS IsSysadmin,
    wanted.ViewName,
    OBJECT_ID(N'C2S.dbo.' + wanted.ViewName, N'V') AS ProdViewId,
    OBJECT_ID(N'C2S_Development.dbo.' + wanted.ViewName, N'V') AS DevViewId
FROM (VALUES
    (N'C2S_TLS_Assets'),
    (N'view_c2sreport_tbl_cATO'),
    (N'cATO_Summary')
) AS wanted(ViewName)
ORDER BY wanted.ViewName;

-- 2. Exact recorded metadata: compare PROD and DEV rows for each column.
-- MaxLengthBytes is bytes, including for Unicode; -1 denotes MAX for MAX types.
-- Nullable: 0 = NOT NULL, 1 = nullable. Do not default data to force a match.
;WITH ColumnMetadata AS (
    SELECT
        N'PROD' AS Environment,
        v.name COLLATE DATABASE_DEFAULT AS ViewName,
        c.column_id AS ColumnOrder,
        c.name COLLATE DATABASE_DEFAULT AS ColumnName,
        ts.name COLLATE DATABASE_DEFAULT AS TypeSchema,
        t.name COLLATE DATABASE_DEFAULT AS TypeName,
        t.is_user_defined AS IsUserDefinedType,
        c.system_type_id AS SystemTypeId,
        c.max_length AS MaxLengthBytes,
        c.precision AS [Precision],
        c.scale AS Scale,
        c.is_nullable AS Nullable,
        c.collation_name COLLATE DATABASE_DEFAULT AS CollationName
    FROM [C2S].sys.views AS v
    INNER JOIN [C2S].sys.schemas AS s ON s.schema_id = v.schema_id
    INNER JOIN [C2S].sys.columns AS c ON c.object_id = v.object_id
    INNER JOIN [C2S].sys.types AS t ON t.user_type_id = c.user_type_id
    INNER JOIN [C2S].sys.schemas AS ts ON ts.schema_id = t.schema_id
    WHERE s.name = N'dbo'
      AND v.name IN (N'C2S_TLS_Assets', N'view_c2sreport_tbl_cATO', N'cATO_Summary')

    UNION ALL

    SELECT
        N'DEV',
        v.name COLLATE DATABASE_DEFAULT,
        c.column_id,
        c.name COLLATE DATABASE_DEFAULT,
        ts.name COLLATE DATABASE_DEFAULT,
        t.name COLLATE DATABASE_DEFAULT,
        t.is_user_defined,
        c.system_type_id,
        c.max_length,
        c.precision,
        c.scale,
        c.is_nullable,
        c.collation_name COLLATE DATABASE_DEFAULT
    FROM [C2S_Development].sys.views AS v
    INNER JOIN [C2S_Development].sys.schemas AS s ON s.schema_id = v.schema_id
    INNER JOIN [C2S_Development].sys.columns AS c ON c.object_id = v.object_id
    INNER JOIN [C2S_Development].sys.types AS t ON t.user_type_id = c.user_type_id
    INNER JOIN [C2S_Development].sys.schemas AS ts ON ts.schema_id = t.schema_id
    WHERE s.name = N'dbo'
      AND v.name IN (N'C2S_TLS_Assets', N'view_c2sreport_tbl_cATO', N'cATO_Summary')
)
SELECT *
FROM ColumnMetadata
ORDER BY ViewName, ColumnOrder,
    CASE WHEN Environment = N'PROD' THEN 0 ELSE 1 END;
GO

-- 3/4. Read only the query history already retained by PROD Query Store.
-- This is limited to C2S history, capture/retention and the latest 100 text hits.
-- It cannot identify every external caller or prove that an object is unused.
BEGIN TRY
    SELECT
        N'C2S' AS DatabaseName,
        actual_state_desc AS QueryStoreState,
        query_capture_mode_desc AS CaptureMode,
        stale_query_threshold_days AS RetentionDays
    FROM [C2S].sys.database_query_store_options;

    SELECT TOP (100)
        COUNT_BIG(*) OVER () AS MatchingQueryCount,
        q.query_id AS QueryId,
        q.object_id AS ModuleObjectId,
        q.last_execution_time AS LastExecutionTime,
        qt.query_sql_text AS QueryText
    FROM [C2S].sys.query_store_query AS q
    INNER JOIN [C2S].sys.query_store_query_text AS qt
        ON qt.query_text_id = q.query_text_id
    WHERE CHARINDEX(N'CATO_SUMMARY', UPPER(qt.query_sql_text)) > 0
      AND CHARINDEX(N'QUERY_STORE_QUERY_TEXT', UPPER(qt.query_sql_text)) = 0
    ORDER BY q.last_execution_time DESC, q.query_id DESC;
END TRY
BEGIN CATCH
    SELECT
        N'Usage history could not be read; this does not mean no usage.' AS UsageCheckStatus,
        ERROR_NUMBER() AS ErrorNumber,
        ERROR_MESSAGE() AS ErrorMessage;
END CATCH;
GO
