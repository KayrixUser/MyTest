USE [C2S];
GO

-- 1. Confirm the actual context, metadata access and wrapper identity.
SELECT
    @@SERVERNAME AS ServerName,
    DB_NAME() AS DatabaseName,
    USER_NAME() AS DatabaseUser,
    IS_SRVROLEMEMBER(N'sysadmin') AS IsSysadmin,
    HAS_PERMS_BY_NAME(DB_NAME(), N'DATABASE', N'VIEW DEFINITION')
        AS CanViewDatabaseDefinitions,
    OBJECT_ID(N'dbo.cATO_Summary') AS WrapperObjectId,
    OBJECT_ID(N'dbo.cATO_Summary', N'V') AS WrapperViewId;

-- 2. Installed column names, types and nullability, in output order.
SELECT
    s.name AS SchemaName,
    o.name AS ObjectName,
    o.type_desc AS ObjectType,
    c.column_id AS ColumnOrder,
    QUOTENAME(c.name) AS ExactColumnName,
    DATALENGTH(c.name) AS ColumnNameBytes,
    SCHEMA_NAME(t.schema_id) AS TypeSchema,
    t.name +
        CASE
            WHEN t.name IN (N'varchar', N'char', N'varbinary', N'binary')
                THEN N'(' + CASE WHEN c.max_length = -1 THEN N'max'
                     ELSE CONVERT(nvarchar(10), c.max_length) END + N')'
            WHEN t.name IN (N'nvarchar', N'nchar')
                THEN N'(' + CASE WHEN c.max_length = -1 THEN N'max'
                     ELSE CONVERT(nvarchar(10), c.max_length / 2) END + N')'
            WHEN t.name IN (N'decimal', N'numeric')
                THEN N'(' + CONVERT(nvarchar(10), c.precision) + N','
                     + CONVERT(nvarchar(10), c.scale) + N')'
            WHEN t.name IN (N'datetime2', N'datetimeoffset', N'time')
                THEN N'(' + CONVERT(nvarchar(10), c.scale) + N')'
            ELSE N''
        END AS DeclaredType,
    t.is_user_defined AS IsUserDefinedType,
    c.is_nullable AS IsNullable,
    c.collation_name AS CollationName
FROM sys.objects AS o
INNER JOIN sys.schemas AS s ON s.schema_id = o.schema_id
INNER JOIN sys.columns AS c ON c.object_id = o.object_id
INNER JOIN sys.types AS t ON t.user_type_id = c.user_type_id
WHERE s.name = N'dbo'
  AND o.name = N'cATO_Summary'
ORDER BY c.column_id;

-- 3. Read the actual installed definition; do not execute its contents.
SELECT
    s.name AS SchemaName,
    o.name AS ObjectName,
    o.type_desc AS ObjectType,
    o.modify_date AS LastModified,
    m.definition AS InstalledDefinition
FROM sys.objects AS o
INNER JOIN sys.schemas AS s ON s.schema_id = o.schema_id
LEFT JOIN sys.sql_modules AS m ON m.object_id = o.object_id
WHERE s.name = N'dbo'
  AND o.name = N'cATO_Summary';
GO
