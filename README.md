SET NOCOUNT ON;

DECLARE @TargetId int = OBJECT_ID(N'PA499.cATO_Summary', N'U');
IF @TargetId IS NULL
    THROW 50000, 'Select the database containing PA499.cATO_Summary (PROD: C2S).', 1;

DECLARE @Map table
(
    Position int NOT NULL PRIMARY KEY,
    TargetColumn sysname NOT NULL,
    SourceColumn sysname NOT NULL,
    ExpectedType sysname NOT NULL
);

INSERT INTO @Map (Position, TargetColumn, SourceColumn, ExpectedType)
VALUES
    (1, N'AssetGuid', N'ASSETGUID', N'uniqueidentifier'),
    (2, N'Business Unit', N'BUSINESS_UNIT', N'varchar'),
    (3, N'Hostname', N'HOSTNAME', N'varchar'),
    (4, N'FQDN', N'FQDN', N'varchar'),
    (5, N'Last Scanned', N'LAST_SCANNED', N'datetime2'),
    (6, N'SAP', N'SAP', N'varchar'),
    (7, N'TotalOpen', N'TOTAL_OPEN', N'int'),
    (8, N'Open 30 Days Past SLA', N'OPEN_30_DAYS_PAST_SLA', N'int'),
    (9, N'is_external', N'IS_EXTERNAL', N'bit'),
    (10, N'VerdantURL', N'VERDANT_URL', N'varchar');

IF EXISTS
(
    SELECT 1
    FROM @Map AS m
    LEFT JOIN sys.columns AS c
      ON c.object_id = @TargetId AND c.name = m.TargetColumn
    WHERE c.column_id IS NULL
       OR TYPE_NAME(c.system_type_id) <> m.ExpectedType
       OR (m.ExpectedType = N'datetime2' AND c.scale <> 7)
)
    THROW 50001, 'PA499 metadata differs from the supplied types. Review its current definition before proceeding.', 1;

IF EXISTS
(
    SELECT 1
    FROM @Map AS m
    LEFT JOIN [ACP_Connection].sys.tables AS t
      ON t.name = N'C2S_CATO_SUMMARY_DATA'
     AND t.schema_id = (SELECT schema_id FROM [ACP_Connection].sys.schemas WHERE name = N'dbo')
    LEFT JOIN [ACP_Connection].sys.columns AS c
      ON c.object_id = t.object_id AND c.name = m.SourceColumn
    WHERE c.column_id IS NULL
       OR (m.ExpectedType IN (N'varchar', N'uniqueidentifier') AND c.system_type_id <> 231)
       OR (m.SourceColumn = N'OPEN_30_DAYS_PAST_SLA' AND c.system_type_id <> 231)
       OR (m.SourceColumn = N'TOTAL_OPEN' AND (c.system_type_id NOT IN (106, 108) OR c.scale <> 0))
       OR (m.ExpectedType = N'datetime2' AND (c.system_type_id <> 42 OR c.scale <> 7))
       OR (m.ExpectedType = N'bit' AND c.system_type_id <> 104)
)
    THROW 50002, 'ACP metadata differs from the supplied types or is not visible. Review before proceeding.', 1;

DECLARE @Position int, @TargetColumn sysname, @SourceColumn sysname,
        @Kind sysname, @MaxBytes int, @Nullable bit, @Collation sysname;
DECLARE @Source nvarchar(max), @Text nvarchar(max), @Encoded nvarchar(max),
        @RoundTrip nvarchar(max), @Rule nvarchar(max), @Problem nvarchar(max);
DECLARE @Projection nvarchar(max) = N'COUNT_BIG(*) AS SourceRows',
        @Rows nvarchar(max) = N'', @Sql nvarchar(max);

DECLARE ColumnRules CURSOR LOCAL FAST_FORWARD FOR
    SELECT m.Position, m.TargetColumn, m.SourceColumn, m.ExpectedType,
           c.max_length, c.is_nullable, c.collation_name
    FROM @Map AS m
    JOIN sys.columns AS c
      ON c.object_id = @TargetId AND c.name = m.TargetColumn
    ORDER BY m.Position;

OPEN ColumnRules;
FETCH NEXT FROM ColumnRules INTO @Position, @TargetColumn, @SourceColumn,
    @Kind, @MaxBytes, @Nullable, @Collation;

WHILE @@FETCH_STATUS = 0
BEGIN
    SET @Source = N'A.' + QUOTENAME(@SourceColumn);
    SET @Text = N'CONVERT(nvarchar(max), ' + @Source + N')';
    SET @Problem = N'1 = 0';

    IF @Kind = N'varchar'
    BEGIN
        -- Convert using the destination column's actual code page. Compare
        -- the round trip as bytes to detect character loss, including '?'.
        SET @Encoded = N'CONVERT(varchar(max), ' + @Text
                     + N' COLLATE ' + QUOTENAME(@Collation) + N')';
        SET @RoundTrip = N'CONVERT(nvarchar(max), ' + @Encoded + N')';
        SET @Problem =
            CASE WHEN @MaxBytes = -1 THEN N'1 = 0'
                 ELSE N'DATALENGTH(' + @Encoded + N') > '
                      + CONVERT(nvarchar(20), @MaxBytes) END
            + N' OR DATALENGTH(' + @RoundTrip + N') <> DATALENGTH(' + @Text + N')'
            + N' OR CONVERT(varbinary(max), ' + @RoundTrip + N')'
            + N' <> CONVERT(varbinary(max), ' + @Text + N')';
    END;

    IF @Kind = N'uniqueidentifier'
    BEGIN
        -- Strict 36-character canonical GUID check: TRY_CONVERT by itself
        -- can accept an overlong value by truncating its suffix.
        SET @Problem = N'DATALENGTH(' + @Text + N') <> 72'
            + N' OR TRY_CONVERT(uniqueidentifier, ' + @Source + N') IS NULL'
            + N' OR CONVERT(varbinary(max), LOWER(' + @Text + N'))'
            + N' <> CONVERT(varbinary(max), LOWER(CONVERT(nvarchar(36), '
            + N'TRY_CONVERT(uniqueidentifier, ' + @Source + N'))))';
    END;

    IF @Kind = N'int'
    BEGIN
        -- Reject blank text, overflow and fractional numeric values.
        SET @Problem = N'LEN(LTRIM(RTRIM(' + @Text + N'))) = 0'
            + N' OR TRY_CONVERT(int, ' + @Source + N') IS NULL'
            + N' OR TRY_CONVERT(decimal(38,10), ' + @Source + N') IS NULL'
            + N' OR TRY_CONVERT(decimal(38,10), ' + @Source + N')'
            + N' <> TRY_CONVERT(int, ' + @Source + N')';
    END;

    IF @Kind = N'bit'
        SET @Problem = N'LEN(LTRIM(RTRIM(' + @Text + N'))) = 0'
            + N' OR TRY_CONVERT(decimal(38,10), ' + @Source + N') IS NULL'
            + N' OR TRY_CONVERT(decimal(38,10), ' + @Source + N') NOT IN (0, 1)';

    IF @Kind = N'datetime2'
        SET @Problem = N'TRY_CONVERT(datetime2(7), ' + @Source + N') IS NULL';

    SET @Rule = N'CASE WHEN ' + @Source + N' IS NULL THEN '
        + CASE WHEN @Nullable = 1 THEN N'0' ELSE N'1' END
        + N' WHEN (' + @Problem + N') THEN 1 ELSE 0 END';

    SET @Projection += N', SUM(CONVERT(bigint, ' + @Rule + N')) AS '
                     + QUOTENAME(N'C' + CONVERT(nvarchar(10), @Position));
    SET @Rows += CASE WHEN @Rows = N'' THEN N'' ELSE N',' END
        + N'(' + CONVERT(nvarchar(10), @Position) + N', N'''
        + REPLACE(@TargetColumn, N'''', N'''''') + N''', P.'
        + QUOTENAME(N'C' + CONVERT(nvarchar(10), @Position)) + N')';

    FETCH NEXT FROM ColumnRules INTO @Position, @TargetColumn, @SourceColumn,
        @Kind, @MaxBytes, @Nullable, @Collation;
END;

CLOSE ColumnRules;
DEALLOCATE ColumnRules;

SET @Sql = N';WITH P AS
(
    SELECT ' + @Projection + N'
    FROM [ACP_Connection].[dbo].[C2S_CATO_SUMMARY_DATA] AS A
)
SELECT
    DB_NAME() AS DatabaseName,
    P.SourceRows,
    V.TargetColumn,
    COALESCE(V.RowsNeedingReview, CONVERT(bigint, 0)) AS RowsNeedingReview,
    CASE WHEN P.SourceRows = 0 THEN N''NO SOURCE ROWS''
         WHEN COALESCE(V.RowsNeedingReview, 0) > 0 THEN N''REVIEW''
         ELSE N''PASS CURRENT VALUES'' END AS CheckResult
FROM P
CROSS APPLY (VALUES ' + @Rows + N') AS V(Position, TargetColumn, RowsNeedingReview)
ORDER BY V.Position;';

-- This dynamic statement consists only of the SELECT above.
EXEC sys.sp_executesql @Sql;
