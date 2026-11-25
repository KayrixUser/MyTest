Run in SQL Server. Copy the entire result column without the header.
SELECT REPLACE((
    SELECT TRIM(g.Content_ID)             AS [id],
           g.Respondent_Status           AS [v.RESPONDENT_STATUS],
           g.Identifying_Information_11  AS [v.IDENTIFYING_INFORMATION_11],
           g.Authorization_Package       AS [v.AUTHORIZATION_PACKAGE]
    FOR JSON PATH, INCLUDE_NULL_VALUES, WITHOUT_ARRAY_WRAPPER
), '$', '\u0024') AS SQL_Row
FROM ACP_Connection.dbo.ArcherMetrics_GlobalTrade_PROD AS g;


Run in Snowflake. Replace PASTE_SQL_ROWS_HERE with the copied column. Keep the $$ on both sides.
WITH src AS (
    SELECT PARSE_JSON(value) AS j
    FROM TABLE(SPLIT_TO_TABLE($$PASTE_SQL_ROWS_HERE$$, CHR(10)))
),
checks AS (
    SELECT g.ARCHER_CONTENT_GLOBAL_TRADE_ASSESSMENT_CONTENT_ID AS id,
           f.key AS field,
           NULLIF(TRIM(AS_VARCHAR(f.value)), '') AS sql_value,
           NULLIF(TRIM(CASE f.key
               WHEN 'RESPONDENT_STATUS'          THEN g.RESPONDENT_STATUS
               WHEN 'IDENTIFYING_INFORMATION_11' THEN g.IDENTIFYING_INFORMATION_11
               WHEN 'AUTHORIZATION_PACKAGE'      THEN g.AUTHORIZATION_PACKAGE_NAME
           END), '') AS sf_value
    FROM src s
    JOIN RTX_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_GLOBAL_TRADE g
      ON g.ARCHER_CONTENT_GLOBAL_TRADE_ASSESSMENT_CONTENT_ID = TRY_TO_NUMBER(s.j:id::VARCHAR),
    LATERAL FLATTEN(INPUT => s.j:v) f
)
SELECT CASE field WHEN 'RESPONDENT_STATUS' THEN 204
                  WHEN 'IDENTIFYING_INFORMATION_11' THEN 205
                  ELSE 209 END AS excel_row,
       id, sql_value, sf_value,
       'SELECT Content_ID, ' || field ||
       ' FROM ACP_Connection.dbo.ArcherMetrics_GlobalTrade_PROD WHERE Content_ID = '''
       || id::VARCHAR || ''';' AS sql_select,
       'SELECT ARCHER_CONTENT_GLOBAL_TRADE_ASSESSMENT_CONTENT_ID, ' ||
       IFF(field = 'AUTHORIZATION_PACKAGE', 'AUTHORIZATION_PACKAGE_NAME', field) ||
       ' FROM RTX_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_GLOBAL_TRADE WHERE ARCHER_CONTENT_GLOBAL_TRADE_ASSESSMENT_CONTENT_ID = '
       || id::VARCHAR || ';' AS sf_select
FROM checks
WHERE sql_value IS DISTINCT FROM sf_value
QUALIFY ROW_NUMBER() OVER (PARTITION BY field ORDER BY id) = 1
ORDER BY excel_row;
