
USE [C2S];

WITH Modules AS
(
    SELECT
        N'C2S' AS DatabaseName,
        s.name COLLATE DATABASE_DEFAULT AS SchemaName,
        o.name COLLATE DATABASE_DEFAULT AS ObjectName,
        m.definition COLLATE DATABASE_DEFAULT AS SqlText
    FROM C2S.sys.sql_modules AS m
    JOIN C2S.sys.objects AS o ON o.object_id = m.object_id
    JOIN C2S.sys.schemas AS s ON s.schema_id = o.schema_id

    UNION ALL

    SELECT
        N'ACP_Connection',
        s.name COLLATE DATABASE_DEFAULT,
        o.name COLLATE DATABASE_DEFAULT,
        m.definition COLLATE DATABASE_DEFAULT
    FROM ACP_Connection.sys.sql_modules AS m
    JOIN ACP_Connection.sys.objects AS o
        ON o.object_id = m.object_id
    JOIN ACP_Connection.sys.schemas AS s
        ON s.schema_id = o.schema_id
)
SELECT
    m.DatabaseName,
    m.SchemaName,
    m.ObjectName,
    t.TargetName,
    SUBSTRING(
        COALESCE(m.SqlText, N''),
        CASE WHEN p.MatchAt > 80 THEN p.MatchAt - 80 ELSE 1 END,
        1000
    ) AS MatchingText
FROM Modules AS m
CROSS JOIN
(
    VALUES
        (N'ArcherMetrics_AuthorizationPackage_PROD'),
        (N'ArcherMetricsAuthorizationPackageBaseTable')
) AS t(TargetName)
CROSS APPLY
(
    VALUES
    (
        PATINDEX(
            N'%[^0-9A-Za-z_]'
            + REPLACE(t.TargetName, N'_', N'[_]')
            + N'[^0-9A-Za-z_]%',
            N' ' + COALESCE(m.SqlText, N'') + N' '
        )
    )
) AS p(MatchAt)
WHERE p.MatchAt > 0
  AND NOT (
      m.SchemaName = N'dbo'
      AND m.ObjectName = N'ArcherMetricsAuthorizationPackageBaseTable'
  )
ORDER BY m.DatabaseName, m.SchemaName, m.ObjectName, t.TargetName;
