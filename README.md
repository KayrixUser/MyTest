USE [C2S];
GO

;WITH RankedNew AS (
    SELECT
        A.HOSTNAME, A.BUSINESS_UNIT, A.FQDN, A.LAST_SCANNED,
        A.TOTAL_OPEN, A.OPEN_30_DAYS_PAST_SLA,
        A.IS_EXTERNAL, A.VERDANT_URL,
        ROW_NUMBER() OVER (
            PARTITION BY A.HOSTNAME ORDER BY A.LAST_SCANNED DESC
        ) AS rn
    FROM ACP_Connection.dbo.C2S_CATO_SUMMARY_DATA A
    INNER JOIN dbo.ArcherMetricsHardwareBaseTable B
        ON B.HostNameFirstPartHW = A.HOSTNAME
),
Differences AS (
    SELECT
        CASE WHEN EXISTS (
            SELECT CAST(O.[Business Unit] AS nvarchar(max))
            EXCEPT SELECT N.BUSINESS_UNIT
        ) THEN 1 END AS BusinessUnit,
        CASE WHEN EXISTS (
            SELECT CAST(O.[FQDN] AS nvarchar(max))
            EXCEPT SELECT N.FQDN
        ) THEN 1 END AS FQDN,
        CASE WHEN EXISTS (
            SELECT O.[Last Scanned]
            EXCEPT SELECT N.LAST_SCANNED
        ) THEN 1 END AS LastScanned,
        CASE WHEN EXISTS (
            SELECT CAST(O.[TotalOpen] AS float)
            EXCEPT SELECT CAST(N.TOTAL_OPEN AS float)
        ) THEN 1 END AS TotalOpen,
        CASE WHEN EXISTS (
            SELECT CAST(O.[Open 30 Days Past SLA] AS float)
            EXCEPT SELECT CAST(N.OPEN_30_DAYS_PAST_SLA AS float)
        ) THEN 1 END AS Open30Days,
        CASE WHEN EXISTS (
            SELECT O.[is_external]
            EXCEPT SELECT N.IS_EXTERNAL
        ) THEN 1 END AS IsExternal,
        CASE WHEN EXISTS (
            SELECT CAST(O.[VerdantURL] AS nvarchar(max))
            EXCEPT SELECT N.VERDANT_URL
        ) THEN 1 END AS VerdantURL
    FROM dbo.view_c2sreport_tbl_cATO O
    INNER JOIN RankedNew N
        ON O.[Hostname] = N.HOSTNAME AND N.rn = 1
)
SELECT
    COUNT_BIG(BusinessUnit) AS BusinessUnit,
    COUNT_BIG(FQDN) AS FQDN,
    COUNT_BIG(LastScanned) AS LastScanned,
    COUNT_BIG(TotalOpen) AS TotalOpen,
    COUNT_BIG(Open30Days) AS Open30Days,
    COUNT_BIG(IsExternal) AS IsExternal,
    COUNT_BIG(VerdantURL) AS VerdantURL
FROM Differences;
