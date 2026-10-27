WITH BASE AS (
    SELECT
        CAPTURE_DATE,
        CAPTURED_AT_UTC,
        BUSINESS_UNIT_NAME,
        FINDING_ID,
        COMMITTED_CLOSURE_DATE,
        LOWER(TRIM(OVERALL_STATUS)) = 'in process'
            AS IS_IN_PROCESS,
        DATEDIFF(
            'day', COMMITTED_CLOSURE_DATE, CAPTURE_DATE
        ) AS DAYS_PAST_DUE,
        YEAR(COMMITTED_CLOSURE_DATE) AS DUE_YEAR,
        YEAR(CAPTURE_DATE) AS CAPTURE_YEAR
    FROM RTX_ENTERPRISESERVICES.ES_ESC_GRC_CURATED.C2S_SOX_FINANCE_TREND_HISTORY
),
RULES AS (
    SELECT
        *,
        IS_IN_PROCESS
            AND DAYS_PAST_DUE <= 30 AS MET_RULE,
        IS_IN_PROCESS
            AND (
                (DUE_YEAR = CAPTURE_YEAR AND DAYS_PAST_DUE > 30)
                OR DUE_YEAR <> CAPTURE_YEAR
            ) AS RISK_RULE
    FROM BASE
)
SELECT
    CAPTURE_DATE,
    CAPTURED_AT_UTC,
    BUSINESS_UNIT_NAME,

    COUNT(DISTINCT CASE WHEN MET_RULE THEN FINDING_ID END)
        AS TOTAL_FINDINGS_TARGET_MET,

    COUNT(DISTINCT CASE WHEN RISK_RULE THEN FINDING_ID END)
        AS TOTAL_FINDINGS_TARGET_AT_RISK,

    COUNT(DISTINCT CASE
        WHEN IS_IN_PROCESS AND COMMITTED_CLOSURE_DATE IS NULL
        THEN FINDING_ID
    END) AS IN_PROCESS_MISSING_CLOSURE_DATE,

    COUNT(DISTINCT CASE WHEN MET_RULE THEN FINDING_ID END)
    + COUNT(DISTINCT CASE WHEN RISK_RULE THEN FINDING_ID END)
    - COUNT(DISTINCT CASE
        WHEN MET_RULE OR RISK_RULE THEN FINDING_ID
      END) AS OVERLAPPING_FINDINGS

FROM RULES
GROUP BY
    CAPTURE_DATE,
    CAPTURED_AT_UTC,
    BUSINESS_UNIT_NAME
ORDER BY
    CAPTURE_DATE,
    CAPTURED_AT_UTC,
    BUSINESS_UNIT_NAME;




    ----c2s


    ;WITH OldTLS AS (
    SELECT
        A.hostname,
        A.fqdn,
        COUNT(DISTINCT A.title) AS Total_TLS_Vulnerabilities,
        MAX(CASE
            WHEN A.title IN (
                'TLS Server Supports TLS version 1.0',
                'TLS Server Supports TLS version 1.1'
            ) THEN 1
            ELSE 0
        END) AS TLS_Encryption
    FROM [C2S].[dbo].[C2S_TLS_Assets] A
    INNER JOIN dbo.dimHardware B
        ON A.hostname = B.Archer_Asset_Name
    WHERE A.fqdn != ''
      AND A.is_external != 1
    GROUP BY A.hostname, A.fqdn
)
SELECT
    COUNT_BIG(O.hostname) AS OldRows,
    COUNT_BIG(N.hostname) AS NewRows,
    COUNT_BIG(CASE WHEN N.hostname IS NULL THEN 1 END) AS OldOnly,
    COUNT_BIG(CASE WHEN O.hostname IS NULL THEN 1 END) AS NewOnly,
    COUNT_BIG(CASE
        WHEN O.Total_TLS_Vulnerabilities <> N.Total_TLS_Vulnerabilities
          OR O.TLS_Encryption <> N.TLS_Encryption
        THEN 1
    END) AS ChangedMetrics
FROM OldTLS O
FULL OUTER JOIN dbo.view_c2sreport_tbl_TLSVerdant N
    ON O.hostname = N.hostname
   AND O.fqdn = N.fqdn;
