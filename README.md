Authorization Package UAT	Prepared 17-Nov-2026
SQL source	ACP_Connection.dbo.ArcherMetrics_AuthorizationPackage_PROD_All
Snowflake source	RTX_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_AUTHORIZATION_PACKAGE_ALL
Record match	Content_ID to ARCHER_CONTENT_AUTHORIZATION_PACKAGE_CONTENT_ID
Business identifier	SAP_ID_RTN; SAP_ID is a separate field
Existing result date	16-Nov-2026; reuse completed results

ID	Validation	Acceptance criteria	Recorded result / evidence needed	Status
AP01	Refresh alignment	Compare the same source cutoff, or account for timing differences.	Load times and source cutoff not yet confirmed.	Pending
AP02	Required fields	Resolve every required mapping and all missing-field gaps.	68 SQL fields; 74 SF outputs; 59 candidates; 9 fields missing (note below).	Open
AP03	Population counts	Explain and reconcile every population difference.	Rows: SQL 4,126 / SF 4,872. Distinct Content_ID and SAP_ID_RTN, each: SQL 4,125 / SF 4,872.	Open
AP04	Missing identifiers	Zero null or blank Content_ID and SAP_ID_RTN values.	Both fields: SQL and SF null = 0; blank = 0.	Pass
AP05	Duplicate identifiers	One record per source identifier; investigate repeated keys.	Content_ID and SAP_ID_RTN each have SQL excess duplicates = 1; SF = 0. Inspect affected records.	Open
AP06	Exact record matching	Reconcile SQL-only and SF-only IDs; verify Content_ID-to-SAP_ID_RTN mapping.	ID sets and key associations not compared. Equal counts alone are insufficient.	Pending
AP07	Field-value comparison	Compare all mapped fields across all matched unique records; explain every value/null difference.	Include package name, acronym, status, classification, connectivity, business, entity and flags. Field-level results pending.	Pending
AP08	Roles and lookups	Preserve correct name/UID pairs, list members and unmatched-reference behavior.	SF role format is UID before name; SQL name parser expects name first. Role/lookup compatibility unresolved.	Open
AP09	Dates and conversions	Preserve date meaning, precision and timezone; no conversion errors or text truncation.	Validate Last_Updated, ConfirmedInArcher, First_Published, ATOIATO_Date and ATOIATO_Expiration_Date; check target data types.	Pending
AP10	Numeric measures	Preserve control counts and compliance meaning, scale and precision.	Validate _Compliant versus PCT_COMPLIANT and control-count fields; required missing fields remain open.	Pending
AP11	Additional_Visibility	Raw and enriched visibility values represent the intended visibility source.	Reviewed SF raw expression uses ALTERNATE_SYSTEM_ADMINISTRATOR_SA. Correction and value verification remain open.	Open
AP12	C2S behavior	Preserve status filters, role expansion and affected package relationships.	Validate main/AD outputs: exclude Decommissioned, retain NULL statuses; verify role rows and package joins.	Pending
AP13	Controlled UAT refresh	Load the mapped data and refresh affected C2S outputs without unexplained loss, duplication or value changes.	Run after mapping fixes in a controlled UAT environment; reconcile loaded rows and affected dimension/fact outputs.	Pending
AP14	Final acceptance	All applicable checks pass; required gaps and unexplained discrepancies are resolved.	Initial profiling recorded. Full data validation and migration acceptance remain open.	Open

Missing SF fields	Alternate ISO raw/name/UID (3); Count_of_Controls; Baseline_Recommendation; Control_Set_Version_Number; Stub_Entry_RTN; SAP_Inactive; Authorization_Package_Source.
Status definitions	Pass = criterion met in recorded evidence. Open = known finding. Pending = execution/evidence outstanding.
Overall status	OPEN - further evidence and fixes required; no production cutover approved.
