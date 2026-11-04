USE [C2S_Development];
GO

SELECT
    @@SERVERNAME AS ServerName,
    DB_NAME() AS ExecutionDatabase,
    p.name AS ProdColumn,
    pt.name AS ProdType,
    p.is_nullable AS ProdNullable,
    dt.name AS DevType,
    d.is_nullable AS DevNullable,
    a.RowsChecked,
    a.NullRows,
    CASE
        WHEN p.column_id IS NULL OR d.column_id IS NULL
            THEN N'REVIEW: metadata missing'
        WHEN pt.name IS NULL OR dt.name IS NULL
          OR pt.name <> N'bit' OR dt.name <> N'bit'
          OR pt.is_user_defined <> 0 OR dt.is_user_defined <> 0
            THEN N'REVIEW: datatype'
        WHEN a.RowsChecked = 0 THEN N'REVIEW: empty view'
        WHEN p.is_nullable = 0 AND a.NullRows > 0
            THEN N'REVIEW: required NULLs'
        ELSE N'PASS'
    END AS FieldCheck
FROM (
    SELECT COUNT_BIG(*) AS RowsChecked,
           COUNT_BIG(CASE WHEN [is_external] IS NULL THEN 1 END) AS NullRows
    FROM [dbo].[cATO_Summary]
) AS a
LEFT JOIN [C2S].sys.columns AS p
    ON p.object_id = OBJECT_ID(N'C2S.dbo.cATO_Summary', N'V')
   AND p.name = N'is_external'
LEFT JOIN [C2S].sys.types AS pt ON pt.user_type_id = p.user_type_id
LEFT JOIN sys.columns AS d
    ON d.object_id = OBJECT_ID(N'dbo.cATO_Summary', N'V')
   AND d.name = N'is_external'
LEFT JOIN sys.types AS dt ON dt.user_type_id = d.user_type_id;
GO
