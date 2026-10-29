-- CATO load validation: SELECT-only.

;WITH Ranked AS (
    SELECT
        A.BUSINESS_UNIT, A.HOSTNAME, A.FQDN, A.LAST_SCANNED,
        A.TOTAL_OPEN, A.OPEN_30_DAYS_PAST_SLA,
        A.IS_EXTERNAL, A.VERDANT_URL,
        DENSE_RANK() OVER (
            PARTITION BY A.HOSTNAME ORDER BY A.LAST_SCANNED DESC
        ) AS ScanRank
    FROM ACP_Connection.dbo.C2S_CATO_SUMMARY_DATA AS A
    WHERE EXISTS (
        SELECT 1
        FROM dbo.ArcherMetricsHardwareBaseTable AS H
        WHERE H.HostNameFirstPartHW = A.HOSTNAME
    )
),
Candidates AS (
    SELECT
        BUSINESS_UNIT, HOSTNAME, FQDN, LAST_SCANNED,
        TOTAL_OPEN, OPEN_30_DAYS_PAST_SLA, IS_EXTERNAL, VERDANT_URL
    FROM Ranked
    WHERE ScanRank = 1
),
Loaded AS (
    SELECT
        [Business Unit], [Hostname], [FQDN], [Last Scanned],
        [TotalOpen], [Open 30 Days Past SLA], [is_external], [VerdantURL]
    FROM dbo.tbl_C2Sreport_cATO
)
SELECT
    DB_NAME() AS DatabaseName,
    (SELECT COUNT_BIG(DISTINCT HOSTNAME) FROM Candidates) AS SourceRows,
    (SELECT COUNT_BIG(*) FROM Loaded) AS LoadedRows,
    (SELECT COUNT_BIG(*) FROM (
        SELECT HOSTNAME FROM Candidates
        EXCEPT
        SELECT [Hostname] FROM Loaded
    ) AS D) AS MissingHostnames,
    (SELECT COUNT_BIG(*) FROM (
        SELECT * FROM Loaded
        EXCEPT
        SELECT * FROM Candidates
    ) AS D) AS InvalidLoadedRows,
    (SELECT COUNT_BIG(*) FROM (
        SELECT [Hostname]
        FROM Loaded
        GROUP BY [Hostname]
        HAVING COUNT_BIG(*) > 1
    ) AS D) AS DuplicateHostnames;
