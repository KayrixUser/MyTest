
/* READ ONLY — run this whole WITH ... SELECT statement together.
   Investigates ONLY FND-3528 (Archer Content ID 8391422) and policy
   master records currently labelled DT-DM-001.

   Goal: locate recorded policy relationships that the existing
   Finding -> Control Standard -> Policy route does not cover.
   The expected policy code is a DIAGNOSTIC filter, not a new mapping.

   This statement does not create or replace an object, change records,
   or modify the dashboard. It has not been executed against your account
   by ChatGPT.

   How to read ID_TOKEN_CHECK:
   * Finding-side policy fields: searches for the Content ID(s) of the
     target policy master record, NOT a match based on similar names.
   * Policy-side Finding fields: searches for Finding Content ID 8391422.
   * Checks whole trimmed numeric tokens separated by semicolon or comma.
     A negative token check does not rule out a different storage format
     or another relationship elsewhere in Archer.
   * A match is evidence of a stored reference, NOT approval to use any
     calculated, helper, hierarchy, retired or DO_NOT_USE field in reporting.

   NULL fields remain visible. Long displayed values are abbreviated only
   AFTER the complete value has been token-tested. Timestamp values are
   diagnostic source values; they do not themselves prove aligned snapshots.
*/
WITH TARGET_FINDING AS (
    SELECT F.*
    FROM RTX_ENTERPRISESERVICES.ES_ESC_GRC_CURATED.ARCHER_CONTENT_FINDINGS F
    WHERE F.ARCHER_CONTENT_FINDINGS_CONTENT_ID = 8391422
),
TARGET_SHARED_FINDING AS (
    SELECT S.*
    FROM RTX_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.ENTERPRISE_FINDINGS S
    WHERE S.ARCHER_CONTENT_FINDINGS_CONTENT_ID = 8391422
),
TARGET_POLICY AS (
    SELECT P.*
    FROM RTX_ENTERPRISESERVICES.ES_ESC_GRC_CURATED.ARCHER_CONTENT_POLICIES_POLICY_LEVEL_3 P
    WHERE TRIM(P.POLICY_LEVEL_3_NAME) = 'DT-DM-001'
),
TARGET_POLICY_KEYS AS (
    SELECT DISTINCT
        ARCHER_CONTENT_POLICIES_POLICY_LEVEL_3_CONTENT_ID AS POLICY_KEY
    FROM TARGET_POLICY
),
SOURCE_RECORDS AS (
    SELECT
        '01 Curated Finding' AS SOURCE_OBJECT,
        F.ARCHER_CONTENT_FINDINGS_CONTENT_ID AS RECORD_KEY,
        OBJECT_CONSTRUCT_KEEP_NULL(F.*) AS SOURCE_FIELDS
    FROM TARGET_FINDING F

    UNION ALL
    SELECT
        '02 Shared published Finding',
        S.ARCHER_CONTENT_FINDINGS_CONTENT_ID,
        OBJECT_CONSTRUCT_KEEP_NULL(S.*)
    FROM TARGET_SHARED_FINDING S

    UNION ALL
    SELECT
        '03 Policy master: DT-DM-001',
        P.ARCHER_CONTENT_POLICIES_POLICY_LEVEL_3_CONTENT_ID,
        OBJECT_CONSTRUCT_KEEP_NULL(P.*)
    FROM TARGET_POLICY P
),
FIELD_VALUES AS (
    SELECT
        R.SOURCE_OBJECT,
        R.RECORD_KEY,
        K.KEY::VARCHAR AS SOURCE_FIELD,
        CASE WHEN IS_NULL_VALUE(K.VALUE) THEN NULL
             ELSE K.VALUE::VARCHAR END AS SOURCE_VALUE
    FROM SOURCE_RECORDS R,
         LATERAL FLATTEN(INPUT => R.SOURCE_FIELDS) K
    WHERE
        (
            R.SOURCE_OBJECT IN ('01 Curated Finding', '02 Shared published Finding')
            AND (
                   K.KEY::VARCHAR ILIKE '%POLIC%'
                OR K.KEY::VARCHAR ILIKE '%CONTROL%'
                OR K.KEY::VARCHAR ILIKE '%DEVIATION%'
                OR K.KEY::VARCHAR ILIKE '%EXCEPTION%'
                OR K.KEY::VARCHAR ILIKE '%REMEDIATION%'
                OR K.KEY::VARCHAR ILIKE '%TEST%'
                OR K.KEY::VARCHAR ILIKE '%UPDATED%'
                OR K.KEY::VARCHAR ILIKE '%PUBLISHED%'
                OR K.KEY::VARCHAR ILIKE '%CONFIRMED%'
                OR K.KEY::VARCHAR IN (
                    'FINDING_ID', 'FINDING_NAME', 'OVERALL_STATUS',
                    'FINDING_WORKFLOW_STAGE', 'INSERT_DATE', 'UPDATE_DATE'
                )
            )
        )
        OR
        (
            R.SOURCE_OBJECT = '03 Policy master: DT-DM-001'
            AND (
                   K.KEY::VARCHAR ILIKE '%FINDING%'
                OR K.KEY::VARCHAR ILIKE '%UPDATED%'
                OR K.KEY::VARCHAR ILIKE '%PUBLISHED%'
                OR K.KEY::VARCHAR ILIKE '%CONFIRMED%'
                OR K.KEY::VARCHAR IN (
                    'POLICY_LEVEL_3_NAME', 'POLICY_NAME', 'POLICY_TOPIC_NAME',
                    'ADMIN_POLICY_NAME', 'STATUS', 'RETIRED_DATE',
                    'INSERT_DATE', 'UPDATE_DATE'
                )
            )
        )
),
REFERENCE_TOKENS AS (
    SELECT
        D.SOURCE_OBJECT,
        D.RECORD_KEY,
        D.SOURCE_FIELD,
        TRY_TO_NUMBER(NULLIF(TRIM(T.VALUE), '')) AS TOKEN_ID
    FROM FIELD_VALUES D,
         LATERAL SPLIT_TO_TABLE(
             REPLACE(COALESCE(D.SOURCE_VALUE, ''), ',', ';'), ';'
         ) T
    WHERE
        (D.SOURCE_OBJECT = '03 Policy master: DT-DM-001'
         AND D.SOURCE_FIELD ILIKE '%FINDING%')
        OR
        (D.SOURCE_OBJECT IN ('01 Curated Finding', '02 Shared published Finding')
         AND D.SOURCE_FIELD ILIKE '%POLIC%')
),
REFERENCE_CHECKS AS (
    SELECT
        T.SOURCE_OBJECT,
        T.RECORD_KEY,
        T.SOURCE_FIELD,
        MAX(CASE
            WHEN T.SOURCE_OBJECT = '03 Policy master: DT-DM-001'
                 AND T.TOKEN_ID = 8391422 THEN 1
            WHEN T.SOURCE_OBJECT IN ('01 Curated Finding', '02 Shared published Finding')
                 AND P.POLICY_KEY IS NOT NULL THEN 1
            ELSE 0
        END) AS FOUND_TARGET_ID
    FROM REFERENCE_TOKENS T
    LEFT JOIN TARGET_POLICY_KEYS P ON T.TOKEN_ID = P.POLICY_KEY
    GROUP BY T.SOURCE_OBJECT, T.RECORD_KEY, T.SOURCE_FIELD
),
RESULT_ROWS AS (
    SELECT
        '00 Source record counts' AS SOURCE_OBJECT,
        8391422::NUMBER AS RECORD_KEY,
        '01 Curated Finding rows' AS SOURCE_FIELD,
        TO_VARCHAR(COUNT(*)) AS SOURCE_VALUE_PREVIEW,
        'Expected: one row for this Finding' AS ID_TOKEN_CHECK
    FROM TARGET_FINDING

    UNION ALL
    SELECT '00 Source record counts', 8391422,
           '02 Shared published Finding rows', TO_VARCHAR(COUNT(*)),
           'Zero means absent from shared-view population'
    FROM TARGET_SHARED_FINDING

    UNION ALL
    SELECT '00 Source record counts', NULL::NUMBER,
           '03 Policy master rows labelled DT-DM-001', TO_VARCHAR(COUNT(*)),
           'Inspect each returned Policy Content ID; do not guess one'
    FROM TARGET_POLICY

    UNION ALL
    SELECT
        D.SOURCE_OBJECT,
        D.RECORD_KEY,
        D.SOURCE_FIELD,
        CASE
            WHEN D.SOURCE_VALUE IS NULL THEN '[NULL]'
            WHEN TRIM(D.SOURCE_VALUE) = '' THEN '[EMPTY]'
            WHEN LENGTH(D.SOURCE_VALUE) > 1500
                THEN LEFT(D.SOURCE_VALUE, 1500)
                     || ' ... [display abbreviated; full value token-tested]'
            ELSE D.SOURCE_VALUE
        END,
        CASE
            WHEN C.FOUND_TARGET_ID = 1
                 AND D.SOURCE_OBJECT = '03 Policy master: DT-DM-001'
                THEN 'FINDING CONTENT ID 8391422 FOUND'
            WHEN C.FOUND_TARGET_ID = 1
                THEN 'TARGET POLICY CONTENT ID FOUND'
            WHEN C.SOURCE_FIELD IS NULL THEN 'Not an ID-list check'
            WHEN D.SOURCE_VALUE IS NULL OR TRIM(D.SOURCE_VALUE) = ''
                THEN 'Field is empty'
            WHEN D.SOURCE_OBJECT <> '03 Policy master: DT-DM-001'
                 AND NOT EXISTS (SELECT 1 FROM TARGET_POLICY_KEYS)
                THEN 'No target Policy master ID available for comparison'
            ELSE 'No exact target ID token found'
        END
    FROM FIELD_VALUES D
    LEFT JOIN REFERENCE_CHECKS C
        ON D.SOURCE_OBJECT = C.SOURCE_OBJECT
       AND D.RECORD_KEY = C.RECORD_KEY
       AND D.SOURCE_FIELD = C.SOURCE_FIELD
)
SELECT
    SOURCE_OBJECT,
    RECORD_KEY,
    SOURCE_FIELD,
    SOURCE_VALUE_PREVIEW,
    ID_TOKEN_CHECK
FROM RESULT_ROWS
ORDER BY SOURCE_OBJECT, RECORD_KEY, SOURCE_FIELD;
