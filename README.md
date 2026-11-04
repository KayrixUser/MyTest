USE [C2S_Development];
GO

SELECT @@SERVERNAME AS ServerName, DB_NAME() AS DatabaseName,
       GETDATE() AS CheckTime;
GO

-- cATO_Summary
;WITH ExpectedColumns AS (
    SELECT *
    FROM (VALUES
        (1, N'AssetGuid', N'uniqueidentifier'),
        (2, N'Business Unit', N'varchar'),
        (3, N'Hostname', N'varchar'),
        (4, N'FQDN', N'varchar'),
        (5, N'Last Scanned', N'datetime2'),
        (6, N'SAP', N'varchar'),
        (7, N'TotalOpen', N'int'),
        (8, N'Open 30 Days Past SLA', N'int'),
        (9, N'VerdantURL', N'varchar'),
        (10, N'is_external', N'bit')
    ) AS e(ColumnOrder, ColumnName, ExpectedType)
),
TargetContract AS (
    SELECT e.ColumnOrder, e.ColumnName, e.ExpectedType,
           c.column_id AS ProdColumnId, t.name AS ProdType,
           c.max_length AS MaxLengthBytes, c.scale AS ProdScale,
           c.is_nullable AS ProdNullable, c.collation_name AS ProdCollation,
           CASE WHEN c.column_id IS NULL OR c.column_id <> e.ColumnOrder
                  OR t.name <> e.ExpectedType OR t.is_user_defined <> 0
                  OR (e.ExpectedType IN (N'varchar', N'nvarchar')
                      AND (c.collation_name IS NULL
                           OR c.collation_name <> N'SQL_Latin1_General_CP1_CI_AS'))
                  OR (e.ExpectedType = N'datetime2' AND c.scale <> 7)
                THEN 1 ELSE 0 END AS UnexpectedMetadata
    FROM ExpectedColumns AS e
    LEFT JOIN [C2S].sys.columns AS c
      ON c.object_id = OBJECT_ID(N'C2S.dbo.cATO_Summary', N'V')
     AND c.name COLLATE DATABASE_DEFAULT = e.ColumnName
    LEFT JOIN [C2S].sys.types AS t ON t.user_type_id = c.user_type_id
),
DataValues AS (
    SELECT x.*
    FROM [dbo].[cATO_Summary] AS s
    CROSS APPLY (VALUES
        (N'AssetGuid', CONVERT(nvarchar(max), s.[AssetGuid]),
         CASE WHEN s.[AssetGuid] IS NOT NULL AND TRY_CONVERT(uniqueidentifier, s.[AssetGuid]) IS NULL THEN 1 ELSE 0 END,
         0,
         CASE WHEN s.[AssetGuid] IS NOT NULL AND TRY_CONVERT(uniqueidentifier, s.[AssetGuid]) IS NOT NULL AND DATALENGTH(CONVERT(nvarchar(max), s.[AssetGuid])) <> 72 THEN 1 ELSE 0 END),
        (N'Business Unit', CONVERT(nvarchar(max), s.[Business Unit]),
         0,
         0,
         0),
        (N'Hostname', CONVERT(nvarchar(max), s.[Hostname]),
         0,
         0,
         0),
        (N'FQDN', CONVERT(nvarchar(max), s.[FQDN]),
         0,
         0,
         0),
        (N'Last Scanned', CONVERT(nvarchar(max), s.[Last Scanned]),
         0,
         0,
         0),
        (N'SAP', CONVERT(nvarchar(max), s.[SAP]),
         0,
         0,
         0),
        (N'TotalOpen', CONVERT(nvarchar(max), s.[TotalOpen]),
         CASE WHEN s.[TotalOpen] IS NOT NULL AND TRY_CONVERT(int, s.[TotalOpen]) IS NULL THEN 1 ELSE 0 END,
         CASE WHEN TRY_CONVERT(int, s.[TotalOpen]) IS NOT NULL AND CONVERT(decimal(38,10), TRY_CONVERT(int, s.[TotalOpen])) <> CONVERT(decimal(38,10), s.[TotalOpen]) THEN 1 ELSE 0 END,
         0),
        (N'Open 30 Days Past SLA', CONVERT(nvarchar(max), s.[Open 30 Days Past SLA]),
         CASE WHEN s.[Open 30 Days Past SLA] IS NOT NULL AND (NULLIF(LTRIM(RTRIM(s.[Open 30 Days Past SLA])), N'') IS NULL OR TRY_CONVERT(int, s.[Open 30 Days Past SLA]) IS NULL) THEN 1 ELSE 0 END,
         0,
         0),
        (N'VerdantURL', CONVERT(nvarchar(max), s.[VerdantURL]),
         0,
         0,
         0),
        (N'is_external', CONVERT(nvarchar(max), s.[is_external]),
         0,
         0,
         0)
    ) AS x(ColumnName, TextValue, ConversionFailure, ValueChange, NoncanonicalGuid)
),
EncodedValues AS (
    SELECT d.*,
           CONVERT(varchar(max),
               d.TextValue COLLATE SQL_Latin1_General_CP1_CI_AS)
               COLLATE SQL_Latin1_General_CP1_CI_AS AS VarcharValue
    FROM DataValues AS d
)
SELECT
    N'cATO_Summary' AS ViewName,
    c.ColumnOrder,
    c.ColumnName,
    CASE
        WHEN c.ProdType IN (N'varchar', N'nvarchar') THEN
            c.ProdType + N'(' +
            CASE WHEN c.MaxLengthBytes = -1 THEN N'max'
                 WHEN c.ProdType = N'nvarchar'
                      THEN CONVERT(nvarchar(10), c.MaxLengthBytes / 2)
                 ELSE CONVERT(nvarchar(10), c.MaxLengthBytes) END + N')'
        WHEN c.ProdType = N'datetime2' THEN
            c.ProdType + N'(' + CONVERT(nvarchar(10), c.ProdScale) + N')'
        ELSE c.ProdType
    END AS RequiredProdType,
    c.ProdNullable,
    c.UnexpectedMetadata,
    COUNT_BIG(d.ColumnName) AS RowsChecked,
    COUNT_BIG(CASE WHEN d.ColumnName IS NOT NULL
                        AND c.ProdNullable = 0 AND d.TextValue IS NULL
                   THEN 1 END) AS RequiredNullRows,
    COUNT_BIG(CASE WHEN d.ConversionFailure = 1 THEN 1 END)
        AS ConversionFailureRows,
    COUNT_BIG(CASE
        WHEN c.MaxLengthBytes <> -1 AND
             ((c.ProdType = N'nvarchar'
               AND DATALENGTH(d.TextValue) > c.MaxLengthBytes)
              OR
              (c.ProdType = N'varchar'
               AND DATALENGTH(d.VarcharValue) > c.MaxLengthBytes))
        THEN 1 END) AS LengthOverflowRows,
    COUNT_BIG(CASE
        WHEN c.ProdType = N'varchar' AND d.TextValue IS NOT NULL AND
             (DATALENGTH(d.TextValue) <>
                  DATALENGTH(CONVERT(nvarchar(max), d.VarcharValue))
              OR CONVERT(varbinary(max), d.TextValue) <>
                  CONVERT(varbinary(max),
                      CONVERT(nvarchar(max), d.VarcharValue)))
        THEN 1 END) AS CodePageLossRows,
    COUNT_BIG(CASE WHEN d.ValueChange = 1 THEN 1 END) AS ValueChangeRows,
    COUNT_BIG(CASE WHEN d.NoncanonicalGuid = 1 THEN 1 END)
        AS NoncanonicalGuidRows
FROM TargetContract AS c
LEFT JOIN EncodedValues AS d ON d.ColumnName = c.ColumnName
GROUP BY c.ColumnOrder, c.ColumnName, c.ProdType, c.MaxLengthBytes,
         c.ProdScale, c.ProdNullable, c.UnexpectedMetadata
ORDER BY c.ColumnOrder;
GO

-- view_c2sreport_tbl_cATO
;WITH ExpectedColumns AS (
    SELECT *
    FROM (VALUES
        (1, N'Business Unit', N'varchar'),
        (2, N'Hostname', N'varchar'),
        (3, N'FQDN', N'varchar'),
        (4, N'Last Scanned', N'datetime2'),
        (5, N'TotalOpen', N'int'),
        (6, N'Open 30 Days Past SLA', N'int'),
        (7, N'is_external', N'bit'),
        (8, N'VerdantURL', N'varchar')
    ) AS e(ColumnOrder, ColumnName, ExpectedType)
),
TargetContract AS (
    SELECT e.ColumnOrder, e.ColumnName, e.ExpectedType,
           c.column_id AS ProdColumnId, t.name AS ProdType,
           c.max_length AS MaxLengthBytes, c.scale AS ProdScale,
           c.is_nullable AS ProdNullable, c.collation_name AS ProdCollation,
           CASE WHEN c.column_id IS NULL OR c.column_id <> e.ColumnOrder
                  OR t.name <> e.ExpectedType OR t.is_user_defined <> 0
                  OR (e.ExpectedType IN (N'varchar', N'nvarchar')
                      AND (c.collation_name IS NULL
                           OR c.collation_name <> N'SQL_Latin1_General_CP1_CI_AS'))
                  OR (e.ExpectedType = N'datetime2' AND c.scale <> 7)
                THEN 1 ELSE 0 END AS UnexpectedMetadata
    FROM ExpectedColumns AS e
    LEFT JOIN [C2S].sys.columns AS c
      ON c.object_id = OBJECT_ID(N'C2S.dbo.view_c2sreport_tbl_cATO', N'V')
     AND c.name COLLATE DATABASE_DEFAULT = e.ColumnName
    LEFT JOIN [C2S].sys.types AS t ON t.user_type_id = c.user_type_id
),
DataValues AS (
    SELECT x.*
    FROM [dbo].[view_c2sreport_tbl_cATO] AS s
    CROSS APPLY (VALUES
        (N'Business Unit', CONVERT(nvarchar(max), s.[Business Unit]),
         0,
         0,
         0),
        (N'Hostname', CONVERT(nvarchar(max), s.[Hostname]),
         0,
         0,
         0),
        (N'FQDN', CONVERT(nvarchar(max), s.[FQDN]),
         0,
         0,
         0),
        (N'Last Scanned', CONVERT(nvarchar(max), s.[Last Scanned]),
         0,
         0,
         0),
        (N'TotalOpen', CONVERT(nvarchar(max), s.[TotalOpen]),
         CASE WHEN s.[TotalOpen] IS NOT NULL AND TRY_CONVERT(int, s.[TotalOpen]) IS NULL THEN 1 ELSE 0 END,
         CASE WHEN TRY_CONVERT(int, s.[TotalOpen]) IS NOT NULL AND CONVERT(decimal(38,10), TRY_CONVERT(int, s.[TotalOpen])) <> CONVERT(decimal(38,10), s.[TotalOpen]) THEN 1 ELSE 0 END,
         0),
        (N'Open 30 Days Past SLA', CONVERT(nvarchar(max), s.[Open 30 Days Past SLA]),
         CASE WHEN s.[Open 30 Days Past SLA] IS NOT NULL AND (NULLIF(LTRIM(RTRIM(s.[Open 30 Days Past SLA])), N'') IS NULL OR TRY_CONVERT(int, s.[Open 30 Days Past SLA]) IS NULL) THEN 1 ELSE 0 END,
         0,
         0),
        (N'is_external', CONVERT(nvarchar(max), s.[is_external]),
         0,
         0,
         0),
        (N'VerdantURL', CONVERT(nvarchar(max), s.[VerdantURL]),
         0,
         0,
         0)
    ) AS x(ColumnName, TextValue, ConversionFailure, ValueChange, NoncanonicalGuid)
),
EncodedValues AS (
    SELECT d.*,
           CONVERT(varchar(max),
               d.TextValue COLLATE SQL_Latin1_General_CP1_CI_AS)
               COLLATE SQL_Latin1_General_CP1_CI_AS AS VarcharValue
    FROM DataValues AS d
)
SELECT
    N'view_c2sreport_tbl_cATO' AS ViewName,
    c.ColumnOrder,
    c.ColumnName,
    CASE
        WHEN c.ProdType IN (N'varchar', N'nvarchar') THEN
            c.ProdType + N'(' +
            CASE WHEN c.MaxLengthBytes = -1 THEN N'max'
                 WHEN c.ProdType = N'nvarchar'
                      THEN CONVERT(nvarchar(10), c.MaxLengthBytes / 2)
                 ELSE CONVERT(nvarchar(10), c.MaxLengthBytes) END + N')'
        WHEN c.ProdType = N'datetime2' THEN
            c.ProdType + N'(' + CONVERT(nvarchar(10), c.ProdScale) + N')'
        ELSE c.ProdType
    END AS RequiredProdType,
    c.ProdNullable,
    c.UnexpectedMetadata,
    COUNT_BIG(d.ColumnName) AS RowsChecked,
    COUNT_BIG(CASE WHEN d.ColumnName IS NOT NULL
                        AND c.ProdNullable = 0 AND d.TextValue IS NULL
                   THEN 1 END) AS RequiredNullRows,
    COUNT_BIG(CASE WHEN d.ConversionFailure = 1 THEN 1 END)
        AS ConversionFailureRows,
    COUNT_BIG(CASE
        WHEN c.MaxLengthBytes <> -1 AND
             ((c.ProdType = N'nvarchar'
               AND DATALENGTH(d.TextValue) > c.MaxLengthBytes)
              OR
              (c.ProdType = N'varchar'
               AND DATALENGTH(d.VarcharValue) > c.MaxLengthBytes))
        THEN 1 END) AS LengthOverflowRows,
    COUNT_BIG(CASE
        WHEN c.ProdType = N'varchar' AND d.TextValue IS NOT NULL AND
             (DATALENGTH(d.TextValue) <>
                  DATALENGTH(CONVERT(nvarchar(max), d.VarcharValue))
              OR CONVERT(varbinary(max), d.TextValue) <>
                  CONVERT(varbinary(max),
                      CONVERT(nvarchar(max), d.VarcharValue)))
        THEN 1 END) AS CodePageLossRows,
    COUNT_BIG(CASE WHEN d.ValueChange = 1 THEN 1 END) AS ValueChangeRows,
    COUNT_BIG(CASE WHEN d.NoncanonicalGuid = 1 THEN 1 END)
        AS NoncanonicalGuidRows
FROM TargetContract AS c
LEFT JOIN EncodedValues AS d ON d.ColumnName = c.ColumnName
GROUP BY c.ColumnOrder, c.ColumnName, c.ProdType, c.MaxLengthBytes,
         c.ProdScale, c.ProdNullable, c.UnexpectedMetadata
ORDER BY c.ColumnOrder;
GO

-- C2S_TLS_Assets
;WITH ExpectedColumns AS (
    SELECT *
    FROM (VALUES
        (1, N'hostname', N'varchar'),
        (2, N'fqdn', N'varchar'),
        (3, N'business_unit', N'varchar'),
        (4, N'SAPs', N'nvarchar'),
        (5, N'title', N'nvarchar'),
        (6, N'description', N'nvarchar'),
        (7, N'first_seen_utc', N'datetime'),
        (8, N'risk_score', N'int'),
        (9, N'sla_utc', N'datetime2'),
        (10, N'severity', N'varchar'),
        (11, N'is_external', N'bit')
    ) AS e(ColumnOrder, ColumnName, ExpectedType)
),
TargetContract AS (
    SELECT e.ColumnOrder, e.ColumnName, e.ExpectedType,
           c.column_id AS ProdColumnId, t.name AS ProdType,
           c.max_length AS MaxLengthBytes, c.scale AS ProdScale,
           c.is_nullable AS ProdNullable, c.collation_name AS ProdCollation,
           CASE WHEN c.column_id IS NULL OR c.column_id <> e.ColumnOrder
                  OR t.name <> e.ExpectedType OR t.is_user_defined <> 0
                  OR (e.ExpectedType IN (N'varchar', N'nvarchar')
                      AND (c.collation_name IS NULL
                           OR c.collation_name <> N'SQL_Latin1_General_CP1_CI_AS'))
                  OR (e.ExpectedType = N'datetime2' AND c.scale <> 7)
                THEN 1 ELSE 0 END AS UnexpectedMetadata
    FROM ExpectedColumns AS e
    LEFT JOIN [C2S].sys.columns AS c
      ON c.object_id = OBJECT_ID(N'C2S.dbo.C2S_TLS_Assets', N'V')
     AND c.name COLLATE DATABASE_DEFAULT = e.ColumnName
    LEFT JOIN [C2S].sys.types AS t ON t.user_type_id = c.user_type_id
),
DataValues AS (
    SELECT x.*
    FROM [dbo].[C2S_TLS_Assets] AS s
    CROSS APPLY (VALUES
        (N'hostname', CONVERT(nvarchar(max), s.[hostname]),
         0,
         0,
         0),
        (N'fqdn', CONVERT(nvarchar(max), s.[fqdn]),
         0,
         0,
         0),
        (N'business_unit', CONVERT(nvarchar(max), s.[business_unit]),
         0,
         0,
         0),
        (N'SAPs', CONVERT(nvarchar(max), s.[SAPs]),
         0,
         0,
         0),
        (N'title', CONVERT(nvarchar(max), s.[title]),
         0,
         0,
         0),
        (N'description', CONVERT(nvarchar(max), s.[description]),
         0,
         0,
         0),
        (N'first_seen_utc', CONVERT(nvarchar(max), s.[first_seen_utc]),
         CASE WHEN s.[first_seen_utc] IS NOT NULL AND TRY_CONVERT(datetime, s.[first_seen_utc]) IS NULL THEN 1 ELSE 0 END,
         CASE WHEN TRY_CONVERT(datetime, s.[first_seen_utc]) IS NOT NULL AND CONVERT(datetime2(7), TRY_CONVERT(datetime, s.[first_seen_utc])) <> CONVERT(datetime2(7), s.[first_seen_utc]) THEN 1 ELSE 0 END,
         0),
        (N'risk_score', CONVERT(nvarchar(max), s.[risk_score]),
         CASE WHEN s.[risk_score] IS NOT NULL AND TRY_CONVERT(int, s.[risk_score]) IS NULL THEN 1 ELSE 0 END,
         CASE WHEN TRY_CONVERT(int, s.[risk_score]) IS NOT NULL AND CONVERT(decimal(38,10), TRY_CONVERT(int, s.[risk_score])) <> CONVERT(decimal(38,10), s.[risk_score]) THEN 1 ELSE 0 END,
         0),
        (N'sla_utc', CONVERT(nvarchar(max), s.[sla_utc]),
         0,
         0,
         0),
        (N'severity', CONVERT(nvarchar(max), s.[severity]),
         0,
         0,
         0),
        (N'is_external', CONVERT(nvarchar(max), s.[is_external]),
         0,
         0,
         0)
    ) AS x(ColumnName, TextValue, ConversionFailure, ValueChange, NoncanonicalGuid)
),
EncodedValues AS (
    SELECT d.*,
           CONVERT(varchar(max),
               d.TextValue COLLATE SQL_Latin1_General_CP1_CI_AS)
               COLLATE SQL_Latin1_General_CP1_CI_AS AS VarcharValue
    FROM DataValues AS d
)
SELECT
    N'C2S_TLS_Assets' AS ViewName,
    c.ColumnOrder,
    c.ColumnName,
    CASE
        WHEN c.ProdType IN (N'varchar', N'nvarchar') THEN
            c.ProdType + N'(' +
            CASE WHEN c.MaxLengthBytes = -1 THEN N'max'
                 WHEN c.ProdType = N'nvarchar'
                      THEN CONVERT(nvarchar(10), c.MaxLengthBytes / 2)
                 ELSE CONVERT(nvarchar(10), c.MaxLengthBytes) END + N')'
        WHEN c.ProdType = N'datetime2' THEN
            c.ProdType + N'(' + CONVERT(nvarchar(10), c.ProdScale) + N')'
        ELSE c.ProdType
    END AS RequiredProdType,
    c.ProdNullable,
    c.UnexpectedMetadata,
    COUNT_BIG(d.ColumnName) AS RowsChecked,
    COUNT_BIG(CASE WHEN d.ColumnName IS NOT NULL
                        AND c.ProdNullable = 0 AND d.TextValue IS NULL
                   THEN 1 END) AS RequiredNullRows,
    COUNT_BIG(CASE WHEN d.ConversionFailure = 1 THEN 1 END)
        AS ConversionFailureRows,
    COUNT_BIG(CASE
        WHEN c.MaxLengthBytes <> -1 AND
             ((c.ProdType = N'nvarchar'
               AND DATALENGTH(d.TextValue) > c.MaxLengthBytes)
              OR
              (c.ProdType = N'varchar'
               AND DATALENGTH(d.VarcharValue) > c.MaxLengthBytes))
        THEN 1 END) AS LengthOverflowRows,
    COUNT_BIG(CASE
        WHEN c.ProdType = N'varchar' AND d.TextValue IS NOT NULL AND
             (DATALENGTH(d.TextValue) <>
                  DATALENGTH(CONVERT(nvarchar(max), d.VarcharValue))
              OR CONVERT(varbinary(max), d.TextValue) <>
                  CONVERT(varbinary(max),
                      CONVERT(nvarchar(max), d.VarcharValue)))
        THEN 1 END) AS CodePageLossRows,
    COUNT_BIG(CASE WHEN d.ValueChange = 1 THEN 1 END) AS ValueChangeRows,
    COUNT_BIG(CASE WHEN d.NoncanonicalGuid = 1 THEN 1 END)
        AS NoncanonicalGuidRows
FROM TargetContract AS c
LEFT JOIN EncodedValues AS d ON d.ColumnName = c.ColumnName
GROUP BY c.ColumnOrder, c.ColumnName, c.ProdType, c.MaxLengthBytes,
         c.ProdScale, c.ProdNullable, c.UnexpectedMetadata
ORDER BY c.ColumnOrder;
GO
