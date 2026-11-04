USE [C2S_Development];
GO
SET ANSI_NULLS ON;
GO
SET QUOTED_IDENTIFIER ON;
GO

ALTER VIEW [dbo].[C2S_TLS_Assets]
AS
SELECT
    CONVERT(varchar(256),
        [HOSTNAME] COLLATE SQL_Latin1_General_CP1_CI_AS) AS [hostname],
    CONVERT(varchar(628),
        [FQDN] COLLATE SQL_Latin1_General_CP1_CI_AS) AS [fqdn],
    CONVERT(varchar(256),
        [BUSINESS_UNIT] COLLATE SQL_Latin1_General_CP1_CI_AS) AS [business_unit],
    CONVERT(nvarchar(4000),
        [SAPS] COLLATE SQL_Latin1_General_CP1_CI_AS) AS [SAPs],
    [TITLE] AS [title],
    [DESCRIPTION] AS [description],
    CONVERT(datetime, [FIRST_SEEN_UTC]) AS [first_seen_utc],
    CONVERT(int, [RISK_SCORE]) AS [risk_score],
    [SLA_UTC] AS [sla_utc],
    CONVERT(varchar(64),
        [SEVERITY] COLLATE SQL_Latin1_General_CP1_CI_AS) AS [severity],
    [IS_EXTERNAL] AS [is_external]
FROM [ACP_Connection].[dbo].[C2S_TLS_ASSETS_DATA];
GO

-- Update dependent-view metadata after the base view's types change.
-- This system procedure does not run the daily model or send notifications.
EXEC sys.sp_refreshview N'dbo.view_c2sreport_tbl_TLSVerdant';
GO

-- Verify only the changed view; this does not repeat the broad data audit.
;WITH Metadata AS (
    SELECT N'PROD' AS Environment, c.column_id AS ColumnOrder,
           c.name COLLATE DATABASE_DEFAULT AS ColumnName,
           t.name COLLATE DATABASE_DEFAULT AS TypeName,
           t.is_user_defined AS IsUserDefinedType,
           c.system_type_id AS SystemTypeId,
           c.max_length AS MaxLengthBytes, c.precision AS TypePrecision,
           c.scale AS TypeScale, c.is_nullable AS IsNullable,
           c.collation_name COLLATE DATABASE_DEFAULT AS CollationName
    FROM [C2S].sys.columns AS c
    INNER JOIN [C2S].sys.types AS t ON t.user_type_id = c.user_type_id
    WHERE c.object_id = OBJECT_ID(N'C2S.dbo.C2S_TLS_Assets', N'V')

    UNION ALL

    SELECT N'DEV', c.column_id, c.name COLLATE DATABASE_DEFAULT,
           t.name COLLATE DATABASE_DEFAULT, t.is_user_defined,
           c.system_type_id, c.max_length, c.precision,
           c.scale, c.is_nullable, c.collation_name COLLATE DATABASE_DEFAULT
    FROM sys.columns AS c
    INNER JOIN sys.types AS t ON t.user_type_id = c.user_type_id
    WHERE c.object_id = OBJECT_ID(N'dbo.C2S_TLS_Assets', N'V')
),
Formatted AS (
    SELECT *,
           TypeName +
           CASE
             WHEN TypeName IN (N'varchar', N'nvarchar', N'char', N'nchar')
               THEN N'(' +
                    CASE WHEN MaxLengthBytes = -1 THEN N'max'
                         WHEN TypeName IN (N'nvarchar', N'nchar')
                           THEN CONVERT(nvarchar(10), MaxLengthBytes / 2)
                         ELSE CONVERT(nvarchar(10), MaxLengthBytes) END + N')'
             WHEN TypeName IN (N'decimal', N'numeric')
               THEN N'(' + CONVERT(nvarchar(10), TypePrecision) + N',' +
                          CONVERT(nvarchar(10), TypeScale) + N')'
             WHEN TypeName IN (N'datetime2', N'datetimeoffset', N'time')
               THEN N'(' + CONVERT(nvarchar(10), TypeScale) + N')'
             ELSE N''
           END AS DisplayType
    FROM Metadata
),
Prod AS (SELECT * FROM Formatted WHERE Environment = N'PROD'),
Dev AS (SELECT * FROM Formatted WHERE Environment = N'DEV')
SELECT
    COALESCE(p.ColumnOrder, d.ColumnOrder) AS ColumnOrder,
    p.ColumnName AS ProdColumn,
    d.ColumnName AS DevColumn,
    p.DisplayType AS ProdType,
    d.DisplayType AS DevType,
    CASE WHEN p.ColumnOrder = d.ColumnOrder
           AND CONVERT(varbinary(max), p.ColumnName) =
               CONVERT(varbinary(max), d.ColumnName)
           AND p.SystemTypeId = d.SystemTypeId
           AND p.TypeName = d.TypeName
           AND p.IsUserDefinedType = 0 AND d.IsUserDefinedType = 0
           AND p.MaxLengthBytes = d.MaxLengthBytes
           AND p.TypePrecision = d.TypePrecision
           AND p.TypeScale = d.TypeScale
           AND ISNULL(p.CollationName, N'') = ISNULL(d.CollationName, N'')
         THEN N'MATCH' ELSE N'REVIEW' END AS DatatypeAndLayout,
    p.IsNullable AS ProdNullable,
    d.IsNullable AS DevNullable,
    CASE WHEN p.IsNullable = d.IsNullable THEN N'MATCH'
         ELSE N'DIFFERENT' END AS NullabilityMetadata
FROM Prod AS p
FULL OUTER JOIN Dev AS d ON d.ColumnOrder = p.ColumnOrder
ORDER BY COALESCE(p.ColumnOrder, d.ColumnOrder);

-- Exercise the existing DEV reporting view after its metadata refresh.
SELECT TOP (1)
    [hostname], [fqdn], [Total_TLS_Vulnerabilities], [TLS_Encryption]
FROM [dbo].[view_c2sreport_tbl_TLSVerdant];
