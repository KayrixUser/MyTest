Check	Test ID	Result	Evidence	Details or Action	Comments(Grady)
Required POAM columns	PO01	Pass	Both exports showed all 15 required column headers.	This pass covers column presence.	
Four added columns	PO02	Pass	Grady’s view includes POAM_ID, HRTN_FIRST_PUBLISHED, POAM_TARGET_COMPLETION_DATE and RELATED_ALLOCATED_CONTROLS.	Close the missing-column item. Check values in the rows below.	
Total rows in the two sources	PO03	Fail	Nov 16 database counts: SQL Server 189, Snowflake 1,475. Snowflake returned 1,286 more rows.	Run Q1 in both sources for current totals and record the run time. Confirm whether both should include the same POAMs.	
Same POAMs in both sources	PO04	Needs Review	On Nov 16, the sources had different numbers of unique Content_IDs. The Nov 20 file comparison reported 152 shared, 37 SQL-only and 1,324 Snowflake-only.	Use Q2 for examples. Use Q5 on the same refresh to confirm the full split and whether every expected SQL POAM appears in Snowflake.	
Missing or repeated Content_ID	PO05	Pass (Nov 16)	The database checks found zero blank, missing or repeated Content_IDs on either side.	Both numbers should be 0 in each source. This checks for missing or repeated Content_IDs. SQL Server: SELECT COUNT(*) - COUNT(id) AS Missing, COUNT(id) - COUNT(DISTINCT id) AS ExtraRepeats FROM (SELECT NULLIF(TRIM(Content_ID), '') AS id FROM ACP_Connection.dbo.ArcherMetrics_POAMs_PROD_All) t;  |  Snowflake: SELECT COUNT(*) - COUNT(id) AS Missing, COUNT(id) - COUNT(DISTINCT id) AS ExtraRepeats FROM (SELECT ARCHER_CONTENT_POAMS_CONTENT_ID AS id FROM ABC_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_POAM) t;	
Missing or repeated POAM_ID	PO06	Needs Review	SQL had 189 different POAM_IDs in 189 rows on Nov 16. The Snowflake check that day used SAP_ID. The later SQL file reported only 2 different POAM_IDs.	Run the same check on POAM_ID. Missing counts blank IDs. ExtraRepeats counts rows beyond the first for each populated ID. SQL Server: SELECT COUNT(*) - COUNT(id) AS Missing, COUNT(id) - COUNT(DISTINCT id) AS ExtraRepeats FROM (SELECT NULLIF(TRIM(POAM_ID), '') AS id FROM ACP_Connection.dbo.ArcherMetrics_POAMs_PROD_All) t;  |  Snowflake: SELECT COUNT(*) - COUNT(id) AS Missing, COUNT(id) - COUNT(DISTINCT id) AS ExtraRepeats FROM (SELECT NULLIF(TRIM(TO_VARCHAR(POAM_ID)), '') AS id FROM ABC_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_POAM) t;	
POAM_ID on the same record	PO07	Needs Review	The file flagged differences on all 152 shared records. Its SQL POAM_ID count conflicts with the earlier source check.	Do both sources give this record the same POAM_ID? SQL Server: SELECT Content_ID, POAM_ID FROM ACP_Connection.dbo.ArcherMetrics_POAMs_PROD_All WHERE Content_ID = '8330200';  |  Snowflake: SELECT ARCHER_CONTENT_POAMS_CONTENT_ID AS Content_ID, POAM_ID FROM ABC_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_POAM WHERE ARCHER_CONTENT_POAMS_CONTENT_ID = 8330200;	
POAM name and status	PO08	Needs Review	The file flagged 8 names and recorded matching status on 152 shared records.	Compare the name and status separately. The file flagged the name on this record. SQL Server: SELECT Content_ID, Name, Overall_Status FROM ACP_Connection.dbo.ArcherMetrics_POAMs_PROD_All WHERE Content_ID = '567275';  |  Snowflake: SELECT ARCHER_CONTENT_POAMS_CONTENT_ID AS Content_ID, NAME, OVERALL_STATUS FROM ABC_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_POAM WHERE ARCHER_CONTENT_POAMS_CONTENT_ID = 567275;	
Authorization package	PO09	Needs Review	The file flagged 13 package names.	Compare the package names linked to this POAM. Snowflake exposes the resolved name in AUTHORIZATION_PACKAGE_NAME. SQL Server: SELECT Content_ID, Authorization_Package FROM ACP_Connection.dbo.ArcherMetrics_POAMs_PROD_All WHERE Content_ID = '8588164';  |  Snowflake: SELECT ARCHER_CONTENT_POAMS_CONTENT_ID AS Content_ID, AUTHORIZATION_PACKAGE_NAME FROM ABC_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_POAM WHERE ARCHER_CONTENT_POAMS_CONTENT_ID = 8588164;	
POAM owner	PO10	Needs Review	The file flagged 14 owner UIDs. Raw owner text and display names can differ even when they refer to the same person.	Check the owner UID and name together. If there are several owners, compare the people listed regardless of display order. SQL Server: SELECT Content_ID, POAM_Owner_UID, POAM_Owner_Name FROM ACP_Connection.dbo.ArcherMetrics_POAMs_PROD_All WHERE Content_ID = '8588164';  |  Snowflake: SELECT ARCHER_CONTENT_POAMS_CONTENT_ID AS Content_ID, POAM_OWNER_ID_ONLY, POAM_OWNER_NAME_ONLY FROM ABC_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_POAM WHERE ARCHER_CONTENT_POAMS_CONTENT_ID = 8588164;	
Related allocated controls	PO11	Needs Review	Both sources expose the field. The file flagged controls on 140 shared records.	Check whether both sources list the same control IDs for this POAM. SQL Server: SELECT Content_ID, Related_Allocated_Controls FROM ACP_Connection.dbo.ArcherMetrics_POAMs_PROD_All WHERE Content_ID = '8330200';  |  Snowflake: SELECT ARCHER_CONTENT_POAMS_CONTENT_ID AS Content_ID, RELATED_ALLOCATED_CONTROLS FROM ABC_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_POAM WHERE ARCHER_CONTENT_POAMS_CONTENT_ID = 8330200;	
Latest update in each source	PO12	Needs Review	The files showed Nov 17 for SQL and Nov 12 for Snowflake. These were the latest record updates in those files.	Compare the latest readable record update across each full source. Confirm the refresh time in the load log if the dates differ. SQL Server: SELECT MAX(TRY_CONVERT(datetime2, NULLIF(TRIM(Last_Updated), ''))) AS LatestUpdate FROM ACP_Connection.dbo.ArcherMetrics_POAMs_PROD_All;  |  Snowflake: SELECT MAX(ARCHER_CONTENT_POAMS_LAST_UPDATED) AS LatestUpdate FROM ABC_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_POAM;	
Latest Archer confirmation	PO13	Needs Review	Power Query could not read the SQL confirmation dates. The latest SQL confirmation is still unverified.	Compare the latest Archer confirmation across each full source. SQL Server: SELECT MAX(ConfirmedInArcher) AS LatestConfirmation FROM ACP_Connection.dbo.ArcherMetrics_POAMs_PROD_All;  |  Snowflake: SELECT MAX(ARCHER_CONTENT_POAMS_CONFIRMED_IN_ARCHER) AS LatestConfirmation FROM ABC_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_POAM;	
Update and confirmation on the same record	PO14	Needs Review	The file flagged update differences and could not compare confirmation dates on the shared records.	For the same POAM, compare update with update and confirmation with confirmation. SQL Server: SELECT Content_ID, Last_Updated, ConfirmedInArcher FROM ACP_Connection.dbo.ArcherMetrics_POAMs_PROD_All WHERE Content_ID = '8330200';  |  Snowflake: SELECT ARCHER_CONTENT_POAMS_CONTENT_ID AS Content_ID, ARCHER_CONTENT_POAMS_LAST_UPDATED, ARCHER_CONTENT_POAMS_CONFIRMED_IN_ARCHER FROM ABC_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_POAM WHERE ARCHER_CONTENT_POAMS_CONTENT_ID = 8330200;	
First published and historical publish date	PO15	Needs Review	The file could not compare First_Published on 152 shared records or hRTN_First_Published on 7.	Compare the original and historical publish dates separately. A blank historical date can be valid. SQL Server: SELECT Content_ID, First_Published, hRTN_First_Published FROM ACP_Connection.dbo.ArcherMetrics_POAMs_PROD_All WHERE Content_ID = '8330200';  |  Snowflake: SELECT ARCHER_CONTENT_POAMS_CONTENT_ID AS Content_ID, ARCHER_CONTENT_POAMS_FIRST_PUBLISHED, HRTN_FIRST_PUBLISHED FROM ABC_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_POAM WHERE ARCHER_CONTENT_POAMS_CONTENT_ID = 8330200;	
Milestone and target completion dates	PO16	Needs Review	The file could not compare milestone dates on 152 shared records or POAM target dates on 91.	Compare milestone with milestone and target with target. Each describes a different date. SQL Server: SELECT Content_ID, Milestone_Estimated_Completion_Date, POAM_Target_Completion_Date FROM ACP_Connection.dbo.ArcherMetrics_POAMs_PROD_All WHERE Content_ID = '8330200';  |  Snowflake: SELECT ARCHER_CONTENT_POAMS_CONTENT_ID AS Content_ID, MILESTONE_ESTIMATED_COMPLETION_DATE, POAM_TARGET_COMPLETION_DATE FROM ABC_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_POAM WHERE ARCHER_CONTENT_POAMS_CONTENT_ID = 8330200;	
Effective publish date and DaysOpen	PO17	Needs Review	The file could not verify these calculations for the 152 shared records.	Use the historical publish date when present, otherwise the original. Compare the effective date and DaysOpen at the same cutoff. Nov 20 reproduces the prior UAT cutoff. Use the actual C2S load date when checking the reported DaysOpen. SQL Server: WITH d AS (SELECT CASE WHEN NULLIF(TRIM(hRTN_First_Published), '') IS NULL THEN CAST(First_Published AS date) ELSE TRY_CONVERT(date, hRTN_First_Published) END AS Published FROM ACP_Connection.dbo.ArcherMetrics_POAMs_PROD_All WHERE Content_ID = '8330200') SELECT Published, DATEDIFF(day, Published, CAST('20261120' AS date)) AS DaysOpen FROM d;  |  Snowflake: WITH d AS (SELECT CASE WHEN NULLIF(TRIM(TO_VARCHAR(HRTN_FIRST_PUBLISHED)), '') IS NULL THEN TO_DATE(ARCHER_CONTENT_POAMS_FIRST_PUBLISHED) ELSE TO_DATE(TRY_TO_TIMESTAMP_NTZ(TO_VARCHAR(HRTN_FIRST_PUBLISHED))) END AS Published FROM ABC_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_POAM WHERE ARCHER_CONTENT_POAMS_CONTENT_ID = 8330200) SELECT Published, DATEDIFF(day, Published, DATE '2026-11-20') AS DaysOpen FROM d;	
POAM UAT sign-off	PO18	Needs Review	The source counts differed and the cause is open. The field findings still need direct verification.	Sign-off decision. Record the agreed POAM population and the outcome of the open checks. A sample confirms only the record tested.	

Q1 | Run first: count the rows and IDs in both source objects
SQL Server
SELECT COUNT(*) AS TotalRows,
       COUNT(DISTINCT NULLIF(TRIM(Content_ID), '')) AS ContentIDs,
       COUNT(DISTINCT NULLIF(TRIM(POAM_ID), '')) AS POAMIDs
FROM ACP_Connection.dbo.ArcherMetrics_POAMs_PROD_All;

Snowflake
SELECT COUNT(*) AS TotalRows,
       COUNT(DISTINCT ARCHER_CONTENT_POAMS_CONTENT_ID) AS ContentIDs,
       COUNT(DISTINCT NULLIF(TRIM(TO_VARCHAR(POAM_ID)), '')) AS POAMIDs
FROM ABC_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_POAM;

Run both without a status filter. Record the database results and the source refresh times if available. TotalRows counts every row; ContentIDs and POAMIDs count different populated identifiers. If all three counts are equal within one source, both IDs are populated and unique in that source. Compare TotalRows across sources separately. If the intended validation covers only active C2S POAMs, agree the status rule and refresh cutoff and apply them consistently as a separate test.

Q2 | Direct lookups for PO04 and the main POAM fields
SQL Server
SELECT Content_ID, POAM_ID, Name, Overall_Status, Authorization_Package,
       POAM_Owner_UID, Related_Allocated_Controls
FROM ACP_Connection.dbo.ArcherMetrics_POAMs_PROD_All
WHERE Content_ID IN ('8330200', '566781', '8277840');

Snowflake
SELECT ARCHER_CONTENT_POAMS_CONTENT_ID AS Content_ID, POAM_ID, NAME, OVERALL_STATUS,
       AUTHORIZATION_PACKAGE_NAME, POAM_OWNER_ID_ONLY, RELATED_ALLOCATED_CONTROLS
FROM ABC_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_POAM
WHERE ARCHER_CONTENT_POAMS_CONTENT_ID IN (8330200, 566781, 8277840);

The Nov 20 files identified 8330200 as shared, 566781 as SQL-only and 8277840 as Snowflake-only. Use the lookup results for the snapshot being tested. Swap in another flagged Content_ID when checking a specific field. For owner display names, the pairing is POAM_Owner_Name in SQL and POAM_OWNER_NAME_ONLY in Snowflake. A match for one example verifies that example only.

Q3 | Latest update and latest confirmation dates, across each full source
SQL Server
SELECT MAX(TRY_CONVERT(datetime2, NULLIF(TRIM(Last_Updated), ''))) AS LatestUpdate,
       MAX(ConfirmedInArcher) AS LatestConfirmation
FROM ACP_Connection.dbo.ArcherMetrics_POAMs_PROD_All;

Snowflake
SELECT MAX(ARCHER_CONTENT_POAMS_LAST_UPDATED) AS LatestUpdate,
       MAX(ARCHER_CONTENT_POAMS_CONFIRMED_IN_ARCHER) AS LatestConfirmation
FROM ABC_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_POAM;

SQL stores Last_Updated as text. The query takes the latest readable update. A maximum record date is separate from the time a refresh finished.

Q4 | Source dates for one shared POAM
SQL Server
SELECT Content_ID, Last_Updated, ConfirmedInArcher, First_Published,
       hRTN_First_Published, Milestone_Estimated_Completion_Date, POAM_Target_Completion_Date
FROM ACP_Connection.dbo.ArcherMetrics_POAMs_PROD_All
WHERE Content_ID = '8330200';

Snowflake
SELECT ARCHER_CONTENT_POAMS_CONTENT_ID AS Content_ID, ARCHER_CONTENT_POAMS_LAST_UPDATED,
       ARCHER_CONTENT_POAMS_CONFIRMED_IN_ARCHER, ARCHER_CONTENT_POAMS_FIRST_PUBLISHED,
       HRTN_FIRST_PUBLISHED, MILESTONE_ESTIMATED_COMPLETION_DATE, POAM_TARGET_COMPLETION_DATE
FROM ABC_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_POAM
WHERE ARCHER_CONTENT_POAMS_CONTENT_ID = 8330200;

Q5 | Complete lists when Grady needs the exact overlap
SQL Server
SELECT DISTINCT TRIM(Content_ID) AS Content_ID
FROM ACP_Connection.dbo.ArcherMetrics_POAMs_PROD_All
WHERE NULLIF(TRIM(Content_ID), '') IS NOT NULL;

Snowflake
SELECT DISTINCT TO_VARCHAR(ARCHER_CONTENT_POAMS_CONTENT_ID) AS Content_ID
FROM ABC_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_POAM
WHERE ARCHER_CONTENT_POAMS_CONTENT_ID IS NOT NULL;

Run at comparable source cutoffs. Compare the two result lists by Content_ID using a full outer join in the approved comparison tool. Count the IDs found in both lists, only in SQL, and only in Snowflake. Separate total counts and a few spot checks cannot give the exact overlap. If expected SQL IDs are missing in Snowflake, check the owner and authorization-package lookup matches for those IDs.

Paste into E55 | Details or Action only | Copy the next 14 lines, from PO05 through PO18
Both numbers should be 0 in each source. This checks for missing or repeated Content_IDs. SQL Server: SELECT COUNT(*) - COUNT(id) AS Missing, COUNT(id) - COUNT(DISTINCT id) AS ExtraRepeats FROM (SELECT NULLIF(TRIM(Content_ID), '') AS id FROM ACP_Connection.dbo.ArcherMetrics_POAMs_PROD_All) t;  |  Snowflake: SELECT COUNT(*) - COUNT(id) AS Missing, COUNT(id) - COUNT(DISTINCT id) AS ExtraRepeats FROM (SELECT ARCHER_CONTENT_POAMS_CONTENT_ID AS id FROM ABC_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_POAM) t;
Run the same check on POAM_ID. Missing counts blank IDs. ExtraRepeats counts rows beyond the first for each populated ID. SQL Server: SELECT COUNT(*) - COUNT(id) AS Missing, COUNT(id) - COUNT(DISTINCT id) AS ExtraRepeats FROM (SELECT NULLIF(TRIM(POAM_ID), '') AS id FROM ACP_Connection.dbo.ArcherMetrics_POAMs_PROD_All) t;  |  Snowflake: SELECT COUNT(*) - COUNT(id) AS Missing, COUNT(id) - COUNT(DISTINCT id) AS ExtraRepeats FROM (SELECT NULLIF(TRIM(TO_VARCHAR(POAM_ID)), '') AS id FROM ABC_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_POAM) t;
Do both sources give this record the same POAM_ID? SQL Server: SELECT Content_ID, POAM_ID FROM ACP_Connection.dbo.ArcherMetrics_POAMs_PROD_All WHERE Content_ID = '8330200';  |  Snowflake: SELECT ARCHER_CONTENT_POAMS_CONTENT_ID AS Content_ID, POAM_ID FROM ABC_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_POAM WHERE ARCHER_CONTENT_POAMS_CONTENT_ID = 8330200;
Compare the name and status separately. The file flagged the name on this record. SQL Server: SELECT Content_ID, Name, Overall_Status FROM ACP_Connection.dbo.ArcherMetrics_POAMs_PROD_All WHERE Content_ID = '567275';  |  Snowflake: SELECT ARCHER_CONTENT_POAMS_CONTENT_ID AS Content_ID, NAME, OVERALL_STATUS FROM ABC_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_POAM WHERE ARCHER_CONTENT_POAMS_CONTENT_ID = 567275;
Compare the package names linked to this POAM. Snowflake exposes the resolved name in AUTHORIZATION_PACKAGE_NAME. SQL Server: SELECT Content_ID, Authorization_Package FROM ACP_Connection.dbo.ArcherMetrics_POAMs_PROD_All WHERE Content_ID = '8588164';  |  Snowflake: SELECT ARCHER_CONTENT_POAMS_CONTENT_ID AS Content_ID, AUTHORIZATION_PACKAGE_NAME FROM ABC_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_POAM WHERE ARCHER_CONTENT_POAMS_CONTENT_ID = 8588164;
Check the owner UID and name together. If there are several owners, compare the people listed regardless of display order. SQL Server: SELECT Content_ID, POAM_Owner_UID, POAM_Owner_Name FROM ACP_Connection.dbo.ArcherMetrics_POAMs_PROD_All WHERE Content_ID = '8588164';  |  Snowflake: SELECT ARCHER_CONTENT_POAMS_CONTENT_ID AS Content_ID, POAM_OWNER_ID_ONLY, POAM_OWNER_NAME_ONLY FROM ABC_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_POAM WHERE ARCHER_CONTENT_POAMS_CONTENT_ID = 8588164;
Check whether both sources list the same control IDs for this POAM. SQL Server: SELECT Content_ID, Related_Allocated_Controls FROM ACP_Connection.dbo.ArcherMetrics_POAMs_PROD_All WHERE Content_ID = '8330200';  |  Snowflake: SELECT ARCHER_CONTENT_POAMS_CONTENT_ID AS Content_ID, RELATED_ALLOCATED_CONTROLS FROM ABC_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_POAM WHERE ARCHER_CONTENT_POAMS_CONTENT_ID = 8330200;
Compare the latest readable record update across each full source. Confirm the refresh time in the load log if the dates differ. SQL Server: SELECT MAX(TRY_CONVERT(datetime2, NULLIF(TRIM(Last_Updated), ''))) AS LatestUpdate FROM ACP_Connection.dbo.ArcherMetrics_POAMs_PROD_All;  |  Snowflake: SELECT MAX(ARCHER_CONTENT_POAMS_LAST_UPDATED) AS LatestUpdate FROM ABC_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_POAM;
Compare the latest Archer confirmation across each full source. SQL Server: SELECT MAX(ConfirmedInArcher) AS LatestConfirmation FROM ACP_Connection.dbo.ArcherMetrics_POAMs_PROD_All;  |  Snowflake: SELECT MAX(ARCHER_CONTENT_POAMS_CONFIRMED_IN_ARCHER) AS LatestConfirmation FROM ABC_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_POAM;
For the same POAM, compare update with update and confirmation with confirmation. SQL Server: SELECT Content_ID, Last_Updated, ConfirmedInArcher FROM ACP_Connection.dbo.ArcherMetrics_POAMs_PROD_All WHERE Content_ID = '8330200';  |  Snowflake: SELECT ARCHER_CONTENT_POAMS_CONTENT_ID AS Content_ID, ARCHER_CONTENT_POAMS_LAST_UPDATED, ARCHER_CONTENT_POAMS_CONFIRMED_IN_ARCHER FROM ABC_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_POAM WHERE ARCHER_CONTENT_POAMS_CONTENT_ID = 8330200;
Compare the original and historical publish dates separately. A blank historical date can be valid. SQL Server: SELECT Content_ID, First_Published, hRTN_First_Published FROM ACP_Connection.dbo.ArcherMetrics_POAMs_PROD_All WHERE Content_ID = '8330200';  |  Snowflake: SELECT ARCHER_CONTENT_POAMS_CONTENT_ID AS Content_ID, ARCHER_CONTENT_POAMS_FIRST_PUBLISHED, HRTN_FIRST_PUBLISHED FROM ABC_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_POAM WHERE ARCHER_CONTENT_POAMS_CONTENT_ID = 8330200;
Compare milestone with milestone and target with target. Each describes a different date. SQL Server: SELECT Content_ID, Milestone_Estimated_Completion_Date, POAM_Target_Completion_Date FROM ACP_Connection.dbo.ArcherMetrics_POAMs_PROD_All WHERE Content_ID = '8330200';  |  Snowflake: SELECT ARCHER_CONTENT_POAMS_CONTENT_ID AS Content_ID, MILESTONE_ESTIMATED_COMPLETION_DATE, POAM_TARGET_COMPLETION_DATE FROM ABC_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_POAM WHERE ARCHER_CONTENT_POAMS_CONTENT_ID = 8330200;
Use the historical publish date when present, otherwise the original. Compare the effective date and DaysOpen at the same cutoff. Nov 20 reproduces the prior UAT cutoff. Use the actual C2S load date when checking the reported DaysOpen. SQL Server: WITH d AS (SELECT CASE WHEN NULLIF(TRIM(hRTN_First_Published), '') IS NULL THEN CAST(First_Published AS date) ELSE TRY_CONVERT(date, hRTN_First_Published) END AS Published FROM ACP_Connection.dbo.ArcherMetrics_POAMs_PROD_All WHERE Content_ID = '8330200') SELECT Published, DATEDIFF(day, Published, CAST('20261120' AS date)) AS DaysOpen FROM d;  |  Snowflake: WITH d AS (SELECT CASE WHEN NULLIF(TRIM(TO_VARCHAR(HRTN_FIRST_PUBLISHED)), '') IS NULL THEN TO_DATE(ARCHER_CONTENT_POAMS_FIRST_PUBLISHED) ELSE TO_DATE(TRY_TO_TIMESTAMP_NTZ(TO_VARCHAR(HRTN_FIRST_PUBLISHED))) END AS Published FROM ABC_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_POAM WHERE ARCHER_CONTENT_POAMS_CONTENT_ID = 8330200) SELECT Published, DATEDIFF(day, Published, DATE '2026-11-20') AS DaysOpen FROM d;
Sign-off decision. Record the agreed POAM population and the outcome of the open checks. A sample 
