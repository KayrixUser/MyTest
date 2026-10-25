USE [C2S];
GO

;WITH RankedNew AS (
    SELECT
        A.HOSTNAME,
        A.LAST_SCANNED,
        A.TOTAL_OPEN,
        A.OPEN_30_DAYS_PAST_SLA,
        ROW_NUMBER() OVER (
            PARTITION BY A.HOSTNAME
            ORDER BY A.LAST_SCANNED DESC
        ) AS rn
    FROM ACP_Connection.dbo.C2S_CATO_SUMMARY_DATA A
    INNER JOIN dbo.ArcherMetricsHardwareBaseTable B
        ON B.HostNameFirstPartHW = A.HOSTNAME
),
MetricDifferences AS (
    SELECT
        O.[Hostname],
        CASE
            WHEN O.[Last Scanned] IS NULL
                 AND N.LAST_SCANNED IS NULL THEN 'Both NULL'
            WHEN O.[Last Scanned] = N.LAST_SCANNED THEN 'Same'
            ELSE 'Different'
        END AS ScanComparison,
        O.[Last Scanned] AS OldLastScanned,
        N.LAST_SCANNED AS NewLastScanned,
        O.[TotalOpen] AS OldTotalOpen,
        N.TOTAL_OPEN AS NewTotalOpen,
        O.[Open 30 Days Past SLA] AS OldOpen30Days,
        N.OPEN_30_DAYS_PAST_SLA AS NewOpen30Days
    FROM dbo.view_c2sreport_tbl_cATO O
    INNER JOIN RankedNew N
        ON O.[Hostname] = N.HOSTNAME AND N.rn = 1
    WHERE EXISTS (
        SELECT
            CAST(O.[TotalOpen] AS float),
            CAST(O.[Open 30 Days Past SLA] AS float)
        EXCEPT
        SELECT
            CAST(N.TOTAL_OPEN AS float),
            CAST(N.OPEN_30_DAYS_PAST_SLA AS float)
    )
)
SELECT TOP (25) *
FROM MetricDifferences
ORDER BY
    CASE ScanComparison
        WHEN 'Same' THEN 0
        WHEN 'Both NULL' THEN 1
        ELSE 2
    END,
    Hostname;
