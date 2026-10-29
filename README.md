SET NOCOUNT ON;

IF DB_NAME() <> N'C2S_Development'
BEGIN
    THROW 51000, 'Select C2S_Development in SSMS before running this test.', 1;
END;

DROP TABLE IF EXISTS #CatoViewActual;
DROP TABLE IF EXISTS #CatoLatestCandidates;

SELECT
    [Business Unit],
    [Hostname],
    [FQDN],
    [Last Scanned],
    [TotalOpen],
    [Open 30 Days Past SLA],
    [is_external],
    [VerdantURL]
INTO #CatoViewActual
FROM [dbo].[view_c2sreport_tbl_cATO];

;WITH Ranked AS (
    SELECT
        A.[BUSINESS_UNIT] AS [Business Unit],
        A.[HOSTNAME] AS [Hostname],
        A.[FQDN] AS [FQDN],
        A.[LAST_SCANNED] AS [Last Scanned],
        A.[TOTAL_OPEN] AS [TotalOpen],
        A.[OPEN_30_DAYS_PAST_SLA] AS [Open 30 Days Past SLA],
        A.[IS_EXTERNAL] AS [is_external],
        A.[VERDANT_URL] AS [VerdantURL],
        DENSE_RANK() OVER (
            PARTITION BY A.[HOSTNAME]
            ORDER BY A.[LAST_SCANNED] DESC
        ) AS LatestRank
    FROM [ACP_Connection].[dbo].[C2S_CATO_SUMMARY_DATA] AS A
    WHERE EXISTS (
        SELECT 1
        FROM [dbo].[ArcherMetricsHardwareBaseTable] AS B
        WHERE B.[HostNameFirstPartHW] = A.[HOSTNAME]
    )
)
SELECT DISTINCT
    [Business Unit],
    [Hostname],
    [FQDN],
    [Last Scanned],
    [TotalOpen],
    [Open 30 Days Past SLA],
    [is_external],
    [VerdantURL]
INTO #CatoLatestCandidates
FROM Ranked
WHERE LatestRank = 1;

SELECT
    DB_NAME() AS [DatabaseName],
    CASE WHEN EXISTS (
        SELECT 1
        FROM sys.sql_expression_dependencies
        WHERE referencing_id = OBJECT_ID(N'dbo.view_c2sreport_tbl_cATO', N'V')
          AND referenced_server_name IS NULL
          AND referenced_database_name = N'ACP_Connection'
          AND referenced_schema_name = N'dbo'
          AND referenced_entity_name = N'C2S_CATO_SUMMARY_DATA'
    ) THEN 1 ELSE 0 END AS UsesLandingSource,
    (SELECT COUNT_BIG(*)
     FROM (
         SELECT [Hostname] FROM #CatoLatestCandidates GROUP BY [Hostname]
     ) AS H) AS ExpectedHostnames,
    (SELECT COUNT_BIG(*) FROM #CatoViewActual) AS ViewRows,
    (SELECT COUNT_BIG(*)
     FROM (
         SELECT [Hostname] FROM #CatoLatestCandidates
         EXCEPT
         SELECT [Hostname] FROM #CatoViewActual
     ) AS M) AS MissingHostnames,
    (SELECT COUNT_BIG(*)
     FROM (
         SELECT [Business Unit], [Hostname], [FQDN], [Last Scanned],
                [TotalOpen], [Open 30 Days Past SLA], [is_external], [VerdantURL]
         FROM #CatoViewActual
         EXCEPT
         SELECT [Business Unit], [Hostname], [FQDN], [Last Scanned],
                [TotalOpen], [Open 30 Days Past SLA], [is_external], [VerdantURL]
         FROM #CatoLatestCandidates
     ) AS U) AS UnexpectedRows,
    (SELECT COUNT_BIG(*)
     FROM (
         SELECT [Hostname]
         FROM #CatoViewActual
         GROUP BY [Hostname]
         HAVING COUNT_BIG(*) > 1
     ) AS D) AS DuplicateHostnames;

DROP TABLE #CatoViewActual;
DROP TABLE #CatoLatestCandidates;
