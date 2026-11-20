Check	Test ID	Result	Evidence	Details or Action	Comments(Grady)
Required POAM fields	PO01	Pass	All 15 required columns are in both files.	Field values are checked below.	
Four previously missing fields	PO02	Pass	Both files contain POAM_ID, hRTN_First_Published, POAM_Target_Completion_Date and Related_Allocated_Controls.	Close the missing-column finding.	
Total rows	PO03	Pass	SQL Server and Snowflake each exported 1,476 rows.	Count covers both full files before the C2S status filter.	
Same POAMs in both files	PO04	Needs Review	152 Content_IDs appear in both files, 37 only in SQL Server and 1,324 only in Snowflake.	Run Q1 first. Use Q2 to look up one example from each group.	
Missing or repeated Content_ID	PO05	Needs Review	SQL has 189 populated IDs and 1,287 rows with a blank ID. Snowflake has 1,476 populated IDs. Neither file repeats a populated Content_ID.	Run Q1 to check the SQL source against its export.	
Missing or repeated POAM_ID	PO06	Needs Review	SQL has 1,287 blanks. The other 189 rows contain only 2 different POAM_IDs. Snowflake has 1,476 different POAM_IDs and no blanks.	Check the POAM_ID count in Q1.	
POAM_ID on the same record	PO07	Needs Review	POAM_ID differs on all 152 shared records.	Use Q2 to compare POAM_ID for Content_ID 8330200.	
POAM name and status	PO08	Needs Review	8 names differ. Overall_Status matches on all 152 shared records.	Review the names. The status check passed for these records.	
Authorization package	PO09	Needs Review	The package name differs on 13 shared records.	Compare SQL Authorization_Package with Snowflake AUTHORIZATION_PACKAGE_NAME.	
POAM owner	PO10	Needs Review	The owner field and owner name each differ on 152 records. The owner UID differs on 14.	Review the owner UID first, then the displayed name.	
Related allocated controls	PO11	Needs Review	The controls differ on 140 of the 152 shared records.	Use Q2 to compare the controls for Content_ID 8330200.	
Latest update in each file	PO12	Needs Review	Latest Last_Updated was Nov 17, 2026 in SQL and Nov 12, 2026 in Snowflake.	Confirm which source refresh each export includes.	
Latest Archer confirmation	PO13	Needs Review	SQL had 189 populated confirmation dates the query could not read. Snowflake's latest was Nov 16, 2026.	Check one original SQL date against the exported value.	
Update and confirmation on the same record	PO14	Needs Review	Last_Updated differs on all 152 shared records. Confirmation dates could not be compared on those 152.	Recheck after confirming the refresh dates and SQL date format.	
First published and historical publish date	PO15	Needs Review	The query could not compare First_Published on 152 records or hRTN_First_Published on 7.	Check the two date fields separately.	
Milestone and target completion dates	PO16	Needs Review	The query could not compare the milestone date on 152 records or the POAM target date on 91.	Check the source date values and how they appear in the CSV.	
Effective publish date and DaysOpen	PO17	Needs Review	The query could not verify either result for the 152 shared records.	Retest with readable publish dates and the same calculation date.	
POAM UAT sign-off	PO18	Needs Review	The export comparison is complete. Only 152 records could be matched for field testing.	Verify the SQL export first. Rerun the comparison, then review any remaining differences and downstream C2S results.	


SELECT COUNT(*) AS TotalRows,
       COUNT(DISTINCT NULLIF(LTRIM(RTRIM(Content_ID)), '')) AS ContentIDs,
       COUNT(DISTINCT NULLIF(LTRIM(RTRIM(POAM_ID)), '')) AS POAMIDs
FROM ACP_Connection.dbo.ArcherMetrics_POAMs_PROD_All;

SELECT Content_ID, POAM_ID, Authorization_Package, Related_Allocated_Controls
FROM ACP_Connection.dbo.ArcherMetrics_POAMs_PROD_All
WHERE Content_ID IN ('8330200', '566781', '8277840');


SELECT ARCHER_CONTENT_POAMS_CONTENT_ID AS Content_ID, POAM_ID,
       AUTHORIZATION_PACKAGE_NAME, RELATED_ALLOCATED_CONTROLS
FROM _ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_POAM
WHERE ARCHER_CONTENT_POAMS_CONTENT_ID IN (8330200, 566781, 8277840);
