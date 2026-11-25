WITH ssms (id, status, info, package_name) AS (
    SELECT * FROM VALUES
        (3448923, 'Approved', 'Saudi PMX Tools', 'DECOM-RAY-CUI-ORION-APP-KSA-PMX Tools'),
        (3450829, 'In Process', 'Bright Horizons', NULL),
        (3927812, 'Assessment Not Required', 'Data Marketplace', NULL)
)
SELECT s.id AS Content_ID,
       s.status       AS SQL_204, g.RESPONDENT_STATUS          AS SF_204,
       s.info         AS SQL_205, g.IDENTIFYING_INFORMATION_11 AS SF_205,
       s.package_name AS SQL_209, g.AUTHORIZATION_PACKAGE_NAME AS SF_209
FROM ssms s
JOIN RTX_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_GLOBAL_TRADE g
  ON g.ARCHER_CONTENT_GLOBAL_TRADE_ASSESSMENT_CONTENT_ID = s.id
WHERE s.status       IS DISTINCT FROM NULLIF(TRIM(g.RESPONDENT_STATUS), '')
   OR s.info         IS DISTINCT FROM NULLIF(TRIM(g.IDENTIFYING_INFORMATION_11), '')
   OR s.package_name IS DISTINCT FROM NULLIF(TRIM(g.AUTHORIZATION_PACKAGE_NAME), '')
ORDER BY s.id;
