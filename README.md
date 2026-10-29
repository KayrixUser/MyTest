;WITH SourceRows AS (
    SELECT
        N'Existing' AS SourceName,
        Hostname,
        CONVERT(nvarchar(max), [Business Unit]) AS BusinessUnit,
        CONVERT(nvarchar(max), FQDN) AS FQDN,
        CONVERT(datetime2(7), [Last Scanned]) AS LastScanned,
        CONVERT(float, TotalOpen) AS TotalOpen,
        CONVERT(float, [Open 30 Days Past SLA]) AS Open30Days,
        CONVERT(int, is_external) AS IsExternal,
        CONVERT(nvarchar(max), VerdantURL) AS VerdantURL
    FROM C2S.PA499.cATO_Summary

    UNION ALL

    SELECT
        N'Landing', HOSTNAME, BUSINESS_UNIT, FQDN, LAST_SCANNED,
        CONVERT(float, TOTAL_OPEN),
        CONVERT(float, OPEN_30_DAYS_PAST_SLA),
        CONVERT(int, IS_EXTERNAL), VERDANT_URL
    FROM ACP_Connection.dbo.C2S_CATO_SUMMARY_DATA
),
Ranked AS (
    SELECT S.*,
        DENSE_RANK() OVER (
            PARTITION BY S.SourceName, S.Hostname
            ORDER BY S.LastScanned DESC
        ) AS ScanRank
    FROM SourceRows AS S
    WHERE EXISTS (
        SELECT 1
        FROM C2S.dbo.ArcherMetricsHardwareBaseTable AS H
        WHERE H.HostNameFirstPartHW = S.Hostname
    )
),
Variants AS (
    SELECT DISTINCT
        SourceName, Hostname, BusinessUnit, FQDN, LastScanned,
        TotalOpen, Open30Days, IsExternal, VerdantURL
    FROM Ranked
    WHERE ScanRank = 1
)
SELECT SourceName, Hostname, COUNT_BIG(*) AS LatestRowVariants
FROM Variants
GROUP BY SourceName, Hostname
HAVING COUNT_BIG(*) > 1
ORDER BY Hostname, SourceName;
