USE [C2S];
GO

;WITH ColumnMetadata AS (
    SELECT
        N'Existing view' AS Source,
        c.name COLLATE DATABASE_DEFAULT AS ColumnName,
        t.name COLLATE DATABASE_DEFAULT AS SqlType,
        c.max_length AS LengthBytes,
        c.precision AS [Precision],
        c.scale AS Scale,
        c.is_nullable AS IsNullable,
        c.collation_name COLLATE DATABASE_DEFAULT AS Collation
    FROM sys.columns c
    INNER JOIN sys.types t
        ON t.user_type_id = c.user_type_id
    WHERE c.object_id = OBJECT_ID(N'dbo.C2S_TLS_Assets', N'V')

    UNION ALL

    SELECT
        N'Landing table',
        c.name COLLATE DATABASE_DEFAULT,
        t.name COLLATE DATABASE_DEFAULT,
        c.max_length,
        c.precision,
        c.scale,
        c.is_nullable,
        c.collation_name COLLATE DATABASE_DEFAULT
    FROM ACP_Connection.sys.columns c
    INNER JOIN ACP_Connection.sys.types t
        ON t.user_type_id = c.user_type_id
    WHERE c.object_id =
          OBJECT_ID(N'ACP_Connection.dbo.C2S_TLS_ASSETS_DATA', N'U')
      AND UPPER(c.name COLLATE DATABASE_DEFAULT) IN (
          N'HOSTNAME', N'FQDN', N'BUSINESS_UNIT', N'SAPS',
          N'TITLE', N'DESCRIPTION', N'FIRST_SEEN_UTC',
          N'RISK_SCORE', N'SLA_UTC', N'SEVERITY', N'IS_EXTERNAL'
      )
)
SELECT *
FROM ColumnMetadata
ORDER BY UPPER(ColumnName), Source;
