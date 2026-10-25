
USE [C2S];
GO

;WITH RankedNew AS (
    SELECT
        A.BUSINESS_UNIT AS [Business Unit],
        A.HOSTNAME AS [Hostname],
        A.FQDN AS [FQDN],
        A.LAST_SCANNED AS [Last Scanned],
        A.SAP AS [SAP],
        A.TOTAL_OPEN AS [TotalOpen],
        A.OPEN_30_DAYS_PAST_SLA AS [Open 30 Days Past SLA],
        A.IS_EXTERNAL AS [is_external],
        A.VERDANT_URL AS [VerdantURL],
        ROW_NUMBER() OVER (
            PARTITION BY A.HOSTNAME
            ORDER BY A.LAST_SCANNED DESC
        ) AS rn
    FROM ACP_Connection.dbo.C2S_CATO_SUMMARY_DATA A
    INNER JOIN dbo.ArcherMetricsHardwareBaseTable B
        ON B.HostNameFirstPartHW = A.HOSTNAME
),
Compared AS (
    SELECT
        O.[Hostname] AS OldHostname,
        N.[Hostname] AS NewHostname,
        CASE WHEN O.[Hostname] IS NOT NULL
                   AND N.[Hostname] IS NOT NULL
                   AND EXISTS (
            SELECT
                CAST(O.[Business Unit] AS nvarchar(max)),
                CAST(O.[FQDN] AS nvarchar(max)),
                O.[Last Scanned],
                CAST(O.[TotalOpen] AS float),
                CAST(O.[Open 30 Days Past SLA] AS float),
                O.[is_external],
                CAST(O.[VerdantURL] AS nvarchar(max))
            EXCEPT
            SELECT
                N.[Business Unit], N.[FQDN], N.[Last Scanned],
                CAST(N.[TotalOpen] AS float),
                CAST(N.[Open 30 Days Past SLA] AS float),
                N.[is_external], N.[VerdantURL]
        ) THEN 1 ELSE 0 END AS HasDifference
    FROM dbo.view_c2sreport_tbl_cATO O
    FULL OUTER JOIN (
        SELECT * FROM RankedNew WHERE rn = 1
    ) N ON O.[Hostname] = N.[Hostname]
)
SELECT
    COUNT_BIG(OldHostname) AS OldRows,
    COUNT_BIG(NewHostname) AS NewRows,
    COUNT_BIG(CASE WHEN NewHostname IS NULL THEN 1 END) AS OldOnly,
    COUNT_BIG(CASE WHEN OldHostname IS NULL THEN 1 END) AS NewOnly,
    COUNT_BIG(CASE WHEN HasDifference = 1 THEN 1 END) AS ChangedRows
FROM Compared;
