Check	Test ID	Result	Evidence	Details or Action	Comments (Grady)
Global Trade source validation	Overall	Needs Review	All 7,779 shared assessments were compared. Across the 12 non-date fields, 6,882 agreed on every field and 897 had at least one difference. No shared assessment could be confirmed across all 16 fields because the date comparison was incomplete.	Source comparison complete with exceptions. Review the population and field differences. Full date agreement and downstream sign-off remain open.
Required fields	Schema	Pass	All 169 expected columns are present in both sources, including all 12 C2S fields and the five previously missing fields.	The five restored fields are approval date and the name and UID of each reviewer collaborator level.
Missing or repeated IDs	GT02	Pass	Both databases have zero missing and zero repeated values for Content_ID and Global_Trade_Assessment_ID.	Every shared Content_ID was eligible for comparison.
Total assessments	GT03	Needs Review	SQL Server contains 7,783 rows. Snowflake contains 7,982. Snowflake has 199 more rows overall.	Use the membership counts below to explain the difference.
Same assessments in both sources	GT04	Needs Review	Of 7,986 distinct Content_IDs, 7,779 are shared, 4 occur only in SQL Server and 203 occur only in Snowflake. There are 207 unmatched IDs.	Confirm whether the records present in only one source are expected. See query 1.
Assessment ID on the same record	GT04	Pass	Global_Trade_Assessment_ID matched on all 7,779 shared Content_IDs.	See query 1 for a shared example.
Respondent status and assessment type	GT04	Needs Review	Respondent status differed on 24 assessments. Assessment type differed on 3.	Review these values on the same assessment. Field totals below show the matching and blank values.
Decision tree and identifying information	GT04	Needs Review	Decision-tree fields 31, 38 and 39 had 7, 6 and 8 exceptions respectively. Identifying information field 11 had 332 exceptions: 330 different values and 2 blank only in Snowflake.	Counts are per field. A single assessment can appear in more than one count.
Reviewer collaborators	GT04	Needs Review	First-level names differed on 16 assessments beyond capitalization. First-level UIDs agreed wherever populated. Second-level names and UIDs each had 2 values blank only in Snowflake.	Capitalization alone accounted for 122 first-level and 70 second-level name comparisons. See query 2.
Approval date	GT05	Partially verified	4,375 shared assessments had blank approval dates on both sides. The other 3,404 date pairs could not be verified in the full comparison. Three populated dates matched in a direct source check.	The three matching dates support the sample only. See query 2.
Assessment creation date	GT06	Not verified	Creation-date agreement could not be established for the 7,779 shared assessments.	Confirm whether SQL Server Created_Date and Snowflake First_Published represent the same business event.
Authorization package name	GT07	Needs Review	Of 7,779 shared assessments, 2,593 populated names matched and 4,593 were blank on both sides. The remaining 593 had different text (573), a blank only in SQL Server (5), or a blank only in Snowflake (15).	Review the package-name differences. The comparison preserves the order of package names.
Snowflake package references	GT07	Needs Review	Across all 7,982 Snowflake rows, 13 had a package reference with no resolved package name. No rows contained multiple references.	Review the 13 unresolved links. Query 3 returns examples. The current population contains no multi-reference case to validate.
Last update and Archer confirmation	Dates	Partially verified	Latest update: SQL Server Nov 23, Snowflake Nov 20. Latest confirmation: SQL Server Nov 24, Snowflake Nov 23. Per-assessment agreement for both dates could not be established on the 7,779 shared records.	The latest dates were verified directly in each source. Review the respective refresh cutoffs.
Refresh order	GT01	Not verified	Job execution order is outside the source comparison.	For end-to-end sign-off, confirm Matillion completed before the corresponding C2S refresh.
Downstream C2S reporting	GT08	Not verified	Downstream assessment and package relationships are outside the source comparison.	For end-to-end sign-off, validate the affected C2S reporting results for the same refresh.

Field results | 7,779 shared assessments
Each field totals 7,779. Counts can overlap across fields. Calendar dates are compared by day. Collaborator names and UIDs are compared as separate lists, ignoring list order. Name capitalization alone counts as agreement. Two blank values count as agreement on that field. An unverified date indicates insufficient evidence to determine agreement.

Field	Populated match	Name case only	Both blank	SQL blank only	Snowflake blank only	Different populated values	Date unverified
Global_Trade_Assessment_ID	7779	0	0	0	0	0	0
Respondent_Status	7754	0	1	0	0	24	0
Assessment_Type	7770	0	6	0	0	3	0
System_Type_Decision_Tree_31	4252	0	3520	0	7	0	0
System_Type_Decision_Tree_38	4220	0	3553	0	6	0	0
System_Type_Decision_Tree_39	4312	0	3459	0	7	1	0
Identifying_Information_11	6714	0	733	0	2	330	0
Authorization_Package (resolved name)	2593	0	4593	5	15	573	0
First_Level_Reviewer_Collaborator_Name	446	122	7195	0	0	16	0
Second_Level_Reviewer_Collaborator_Name	58	70	7649	0	2	0	0
First_Level_Reviewer_Collaborator_UID	584	0	7195	0	0	0	0
Second_Level_Reviewer_Collaborator_UID	128	0	7649	0	2	0	0
Last_Updated	0	0	0	0	0	0	7779
ConfirmedInArcher	0	0	0	0	0	0	7779
Created_Date / First_Published	0	0	0	0	0	0	7779
Approval_Date	0	0	4375	0	0	0	3404

Read-only sample queries

Which assessments appear in each source?
On Nov 24, Content_ID 6207595 was present in both sources and 8591687 was present only in SQL Server. Run the two queries and compare the returned IDs. These examples illustrate the membership check. The totals above cover the full population.

SQL Server
SELECT Content_ID, Global_Trade_Assessment_ID, Respondent_Status
FROM ACP_Connection.dbo.ArcherMetrics_GlobalTrade_PROD
WHERE Content_ID IN ('6207595', '8591687')
ORDER BY Content_ID;

Snowflake
SELECT ARCHER_CONTENT_GLOBAL_TRADE_ASSESSMENT_CONTENT_ID AS Content_ID,
GLOBAL_TRADE_ASSESSMENT_ID, RESPONDENT_STATUS
FROM RTX_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_GLOBAL_TRADE
WHERE ARCHER_CONTENT_GLOBAL_TRADE_ASSESSMENT_CONTENT_ID IN (6207595, 8591687)
ORDER BY Content_ID;

Do the approval date and reviewer collaborator values agree on the same assessment?
The Nov 24 source check found the following approval dates in both systems: 6207595 (2026-05-29), 6597301 (2026-02-26) and 6597303 (2025-11-19). For 6207595, the collaborator names differed only in capitalization and the UIDs matched.

SQL Server
SELECT Content_ID, Approval_Date,
First_Level_Reviewer_Collaborator_Name AS First_Name,
First_Level_Reviewer_Collaborator_UID AS First_UID,
Second_Level_Reviewer_Collaborator_Name AS Second_Name,
Second_Level_Reviewer_Collaborator_UID AS Second_UID
FROM ACP_Connection.dbo.ArcherMetrics_GlobalTrade_PROD
WHERE Content_ID IN ('6207595', '6597301', '6597303')
ORDER BY Content_ID;

Snowflake
SELECT ARCHER_CONTENT_GLOBAL_TRADE_ASSESSMENT_CONTENT_ID AS Content_ID, APPROVAL_DATE,
FIRST_LEVEL_REVIEWER_COLLABORATOR_NAME_ONLY AS First_Name,
FIRST_LEVEL_REVIEWER_COLLABORATOR_ID_ONLY AS First_UID,
SECOND_LEVEL_REVIEWER_COLLABORATOR_ONLY AS Second_Name,
SECOND_LEVEL_REVIEWER_COLLABORATOR_ID_ONLY AS Second_UID
FROM RTX_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_GLOBAL_TRADE
WHERE ARCHER_CONTENT_GLOBAL_TRADE_ASSESSMENT_CONTENT_ID IN (6207595, 6597301, 6597303)
ORDER BY Content_ID;

Which Snowflake assessments have a package reference with no resolved name?
Returns three current examples and the total number of qualifying rows. The Nov 24 total was 13. A reference or name must contain a character other than a space, other whitespace or a semicolon.

Snowflake
SELECT ARCHER_CONTENT_GLOBAL_TRADE_ASSESSMENT_CONTENT_ID AS Content_ID,
AUTHORIZATION_PACKAGE AS Package_Reference, AUTHORIZATION_PACKAGE_NAME AS Package_Name,
COUNT(*) OVER () AS Total_Unresolved
FROM RTX_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_GLOBAL_TRADE
WHERE REGEXP_INSTR(COALESCE(AUTHORIZATION_PACKAGE, ''), '[^;[:space:]]') > 0
AND REGEXP_INSTR(COALESCE(AUTHORIZATION_PACKAGE_NAME, ''), '[^;[:space:]]') = 0
ORDER BY Content_ID
LIMIT 3;
