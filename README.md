Check	Test ID	Result	Evidence	Details or Action	Comments(Grady)
Previously missing columns	Schema	Pass	Approval date and all four reviewer collaborator fields are present in both exports (5 of 5).	Column presence verified. Value comparison is separate.
Fields used by C2S	Schema	Pass	Both exports contain all 12 required C2S fields.	Complete for column presence.
All workbook fields	Schema	Needs Review	SQL matched 168 of 169 expected names. Snowflake matched 169 of 169. One SQL helper header appeared under a different name.	Check the SQL helper header with the optional metadata query. It is outside the 12 C2S fields.
Total assessment rows	GT03	Needs Review	SQL export 7,975. Snowflake export 7,982.	Compare with the native row counts for the same refresh.
Same assessments in both exports	GT04	Needs Review	7,779 populated Content_IDs appeared in both exports, 4 only in SQL and 203 only in Snowflake.	Confirm the expected assessment population.
Missing and repeated IDs	GT02	Not verified	The Excel ID checks returned errors. The SQL export has 192 more rows than distinct populated Content_IDs. Snowflake has 7,982 rows and 7,982 distinct populated Content_IDs.	Run the two source summaries for both Content_ID and Global_Trade_Assessment_ID.
Values on the same assessment	GT04 / GT05	Not verified	Excel returned errors for the matched-record checks, including approval date and the collaborator values.	The sample queries below compare the five restored values for the same Content_IDs. Record the number of shared samples checked.
Created date	GT06	Needs Review	The SQL creation date is mapped to Snowflake first publication as a candidate.	The data owner must confirm whether both fields represent the same event.
Package references	GT07	Needs Review	13 Snowflake export rows have a package reference with no resolved package name. SQL has 3,181 populated package names and Snowflake has 3,236.	Review unresolved references. The multi-package check returned an error.
Latest update and confirmation	Dates	Needs Review	The latest readable update was Nov 23 in SQL and Nov 20 in Snowflake. Excel could not read the SQL confirmation date. The latest Snowflake confirmation was Nov 23.	Use the source summaries below for native date results.
Refresh order and downstream results	GT01 / GT08	Separate check	The export comparison covers source data only.	Use refresh logs and the downstream model if full C2S sign-off is required.
Global Trade UAT decision	Overall	Needs Review	The five missing-column fixes and all 12 C2S headers passed. Population and field-value exceptions remain open.	Close the column fix. Record the native results and resolve or accept exceptions before full sign-off.

Quick source check | one result row per database
Run the SQL Server block in SSMS and the Snowflake block in Snowflake, close together. Missing counts null or blank identifiers. Repeats counts extra occurrences of a populated identifier. Compare current native results with the export results only after checking the refresh date. These queries do not perform a cross-database join.

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

Small field check | for a sample-level conclusion on the five restored fields

SQL Server | select three assessment IDs, prioritizing populated collaborator and approval values

SELECT TOP (3) Content_ID, Approval_Date,
First_Level_Reviewer_Collaborator_Name AS First_Name, First_Level_Reviewer_Collaborator_UID AS First_UID,
Second_Level_Reviewer_Collaborator_Name AS Second_Name, Second_Level_Reviewer_Collaborator_UID AS Second_UID
FROM ACP_Connection.dbo.ArcherMetrics_GlobalTrade_PROD
WHERE NULLIF(TRIM(Content_ID), '') IS NOT NULL
ORDER BY (CASE WHEN NULLIF(TRIM(Second_Level_Reviewer_Collaborator_UID), '') IS NULL THEN 0 ELSE 1 END
+ CASE WHEN NULLIF(TRIM(First_Level_Reviewer_Collaborator_UID), '') IS NULL THEN 0 ELSE 1 END
+ CASE WHEN Approval_Date IS NULL THEN 0 ELSE 1 END) DESC, Content_ID;

Snowflake | replace the three example numbers below with the Content_IDs just returned by SQL Server

SELECT ARCHER_CONTENT_GLOBAL_TRADE_ASSESSMENT_CONTENT_ID AS Content_ID, APPROVAL_DATE,
FIRST_LEVEL_REVIEWER_COLLABORATOR_NAME_ONLY AS First_Name, FIRST_LEVEL_REVIEWER_COLLABORATOR_ID_ONLY AS First_UID,
SECOND_LEVEL_REVIEWER_COLLABORATOR_ONLY AS Second_Name, SECOND_LEVEL_REVIEWER_COLLABORATOR_ID_ONLY AS Second_UID
FROM RTX_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_GLOBAL_TRADE
WHERE ARCHER_CONTENT_GLOBAL_TRADE_ASSESSMENT_CONTENT_ID IN (1, 2, 3)
ORDER BY Content_ID;

Compare by Content_ID. Record how many shared, unique samples had matching populated approval dates (calendar day), names and UIDs. Report blanks on both sides as blanks. If an ID is absent on one side, count it only in coverage. If either side returns multiple rows for the ID, exclude it from the field spot check. A sample pass applies only to the records checked. You can share match counts without names or UIDs.

Optional | SQL helper name, if you want to reconcile the 169th header

SELECT c.name AS SQL_Column
FROM ACP_Connection.sys.columns AS c
WHERE c.object_id = OBJECT_ID(N'ACP_Connection.dbo.ArcherMetrics_GlobalTrade_PROD')
AND c.name LIKE '%HELPER%BACKGROUND%';
