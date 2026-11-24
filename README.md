Check	Test ID	Result	Evidence	Details or Action	Comments(Grady)
All workbook columns	Schema	Pass (presence)	All 169 expected fields are accounted for. Excel matched 168 SQL column names and all 169 Snowflake names. SQL metadata identified the remaining helper field.	SQL has two underscores between Background and Previous. Snowflake has one. Use the mapping below for future Excel runs.
C2S columns and five restored columns	Schema	Pass (presence)	Both sources contain all 12 C2S fields and all five restored fields: Approval_Date and the name and UID of each reviewer collaborator level.	Close the missing-column finding. The sample results below cover values.
Missing or repeated IDs	GT02	Pass	SQL Server: 7,783 rows and unique Content_IDs. Snowflake: 7,982 rows and unique Content_IDs. Both sources returned zero missing and zero repeated IDs for Content_ID and Global_Trade_Assessment_ID.	Complete for both keys.
Total assessment rows	GT03	Needs Review	SQL Server returned 7,783 rows and Snowflake returned 7,982. Snowflake has 199 more rows overall.	Use the direct database counts. The earlier SQL file count of 7,975 was 192 higher than the database result.
Same assessments in both sources	GT04	Needs Review	The export ID comparison found 7,779 shared Content_IDs, 4 SQL-only and 203 Snowflake-only. These counts reconcile to the native totals of 7,783 and 7,982.	Confirm whether the source-only assessments are expected at the compared refresh cutoffs.
Other field values on shared assessments	GT04	Not verified	The Excel comparison returned errors for field-level checks. The database sample covered the five restored fields.	For full value sign-off, complete the remaining required field comparisons by shared Content_ID or record a scope decision.
Reviewer collaborator names and UIDs	GT04	Pass (sample, ignoring name case)	For Content_IDs 6207595, 6597301 and 6597303, all six first- and second-level collaborator UID comparisons matched. Both names matched on all three IDs when case was ignored.	For 6207595, both names use uppercase in SQL and title case in Snowflake. Record this difference if exact case is required.
Approval date	GT05	Pass (sample)	Approval dates matched for all three shared Content_IDs: 6207595 (2026-05-29), 6597301 (2026-02-26) and 6597303 (2025-11-19).	Complete for these three populated records.
Created date	GT06	Needs Review	SQL Created_Date is mapped to Snowflake ARCHER_CONTENT_GLOBAL_TRADE_ASSESSMENT_FIRST_PUBLISHED. Whether both fields represent the same event is unconfirmed.	Ask the data owner to confirm the business meaning before accepting this mapping.
Authorization package links	GT07	Needs Review	The Snowflake export showed 13 assessment rows with a package reference and no resolved package name. The Excel multi-package check returned an error.	Confirm the disposition of the 13 rows and verify multi-package links if required for C2S reporting.
Latest update and Archer confirmation	Dates	Needs Review	Latest update: SQL 2026-11-23, Snowflake 2026-11-20. Latest Archer confirmation: SQL 2026-11-24, Snowflake 2026-11-23.	Both pairs are native calendar-date maxima. Use refresh records to determine whether the cutoffs explain the differences.
Refresh order	GT01	Not verified	The database summaries report assessment dates. Job execution times were outside these results.	For end-to-end sign-off, check that Matillion finished before the corresponding C2S refresh.
Downstream C2S results	GT08	Not verified	The source checks did not test dimGlobalTrade, downstream package links or affected reporting results.	For end-to-end sign-off, verify the affected downstream results for that refresh.
Global Trade UAT decision	Overall	Needs Review	Column presence and both ID checks passed. The five restored values were checked on three shared records, with a name-case difference. Coverage, package links and date questions remain.	Close the verified items. Record acceptance or resolution of the exceptions and complete any remaining in-scope checks before overall sign-off.

Reproducing the completed checks (optional)

Source row counts, keys and latest dates

SQL Server
WITH g AS (
SELECT NULLIF(TRIM(Content_ID), '') AS cid,
NULLIF(TRIM(Global_Trade_Assessment_ID), '') AS aid,
TRY_CONVERT(datetime2, NULLIF(TRIM(Last_Updated), '')) AS updated,
ConfirmedInArcher AS confirmed
FROM ACP_Connection.dbo.ArcherMetrics_GlobalTrade_PROD
)
SELECT COUNT() AS Rows_Total,
COUNT(DISTINCT cid) AS Content_Unique,
COUNT() - COUNT(cid) AS Content_Missing,
COUNT(cid) - COUNT(DISTINCT cid) AS Content_Repeats,
COUNT(*) - COUNT(aid) AS Assessment_Missing,
COUNT(aid) - COUNT(DISTINCT aid) AS Assessment_Repeats,
CAST(MAX(updated) AS date) AS Latest_Update,
CAST(MAX(confirmed) AS date) AS Latest_Confirmation
FROM g;

Snowflake
WITH g AS (
SELECT ARCHER_CONTENT_GLOBAL_TRADE_ASSESSMENT_CONTENT_ID AS cid,
GLOBAL_TRADE_ASSESSMENT_ID AS aid,
ARCHER_CONTENT_GLOBAL_TRADE_ASSESSMENT_LAST_UPDATED AS updated,
ARCHER_CONTENT_GLOBAL_TRADE_ASSESSMENT_CONFIRMED_IN_ARCHER AS confirmed
FROM RTX_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_GLOBAL_TRADE
)
SELECT COUNT() AS Rows_Total,
COUNT(DISTINCT cid) AS Content_Unique,
COUNT() - COUNT(cid) AS Content_Missing,
COUNT(cid) - COUNT(DISTINCT cid) AS Content_Repeats,
COUNT(*) - COUNT(aid) AS Assessment_Missing,
COUNT(aid) - COUNT(DISTINCT aid) AS Assessment_Repeats,
CAST(MAX(updated) AS date) AS Latest_Update,
CAST(MAX(confirmed) AS date) AS Latest_Confirmation
FROM g;

Same three assessed records and five restored fields

SQL Server
SELECT Content_ID, Approval_Date,
First_Level_Reviewer_Collaborator_Name AS First_Name, First_Level_Reviewer_Collaborator_UID AS First_UID,
Second_Level_Reviewer_Collaborator_Name AS Second_Name, Second_Level_Reviewer_Collaborator_UID AS Second_UID
FROM ACP_Connection.dbo.ArcherMetrics_GlobalTrade_PROD
WHERE Content_ID IN ('6207595', '6597301', '6597303')
ORDER BY Content_ID;

Snowflake
SELECT ARCHER_CONTENT_GLOBAL_TRADE_ASSESSMENT_CONTENT_ID AS Content_ID, APPROVAL_DATE,
FIRST_LEVEL_REVIEWER_COLLABORATOR_NAME_ONLY AS First_Name, FIRST_LEVEL_REVIEWER_COLLABORATOR_ID_ONLY AS First_UID,
SECOND_LEVEL_REVIEWER_COLLABORATOR_ONLY AS Second_Name, SECOND_LEVEL_REVIEWER_COLLABORATOR_ID_ONLY AS Second_UID
FROM RTX_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_GLOBAL_TRADE
WHERE ARCHER_CONTENT_GLOBAL_TRADE_ASSESSMENT_CONTENT_ID IN (6207595, 6597301, 6597303)
ORDER BY Content_ID;

SQL helper header confirmed by metadata

SELECT c.name AS SQL_Column
FROM ACP_Connection.sys.columns AS c
WHERE c.object_id = OBJECT_ID(N'ACP_Connection.dbo.ArcherMetrics_GlobalTrade_PROD')
AND c.name LIKE '%HELPER%BACKGROUND%';

Power Query FieldMap row for future refreshes (SQL name first, Snowflake name second)
{"HELPER_BACKGROUND__PREVIOUS_ASSESSMENT_INFORMATION_COMPLETE", "HELPER_BACKGROUND_PREVIOUS_ASSESSMENT_INFORMATION_COMPLETE"},
