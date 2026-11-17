**Authorization Package UAT**

SQL source: `ACP_Connection.dbo.ArcherMetrics_AuthorizationPackage_PROD_All`  
Snowflake source: `RTX_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_AUTHORIZATION_PACKAGE_ALL`  
Existing test results: November 16, 2026.

| Test ID | Validation | Expected result | Current finding | Status |
|---|---|---|---|---|
| AP01 | Refresh timing | Compare equivalent source snapshots and explain timing differences. | Source cutoff and load times need confirmation. | Pending |
| AP02 | Field mapping | All required fields have validated mappings. | 68 SQL fields, 74 SF outputs, 59 mapping candidates and 9 missing fields. | Open |
| AP03 | Record counts | Reconcile differences in total records and distinct IDs. | Rows: SQL 4,126; SF 4,872. Distinct Content_ID and SAP_ID_RTN, each: SQL 4,125; SF 4,872. | Open |
| AP04 | Null and blank IDs | No null or blank Content_ID or SAP_ID_RTN values. | Both fields have zero null and blank values in SQL and SF. | Pass |
| AP05 | Duplicate IDs | One source record per identifier. | Content_ID and SAP_ID_RTN each have one excess duplicate row in SQL and none in SF. Affected records need inspection. | Open |
| AP06 | Record matching | Identify records present on only one side and verify the Content_ID-to-SAP_ID_RTN relationship. | Actual ID sets and key relationships have not been compared. | Pending |
| AP07 | Field values | Compare all mapped fields for every matched unique record and explain value or null differences. | Package details, status, classification, connectivity, business, entity and other mapped values need comparison. | Pending |
| AP08 | Roles and lookups | Preserve names, UIDs, list members and handling of unmatched references. | SF role strings place UID before name; the SQL name parser expects name first. Compatibility needs resolution. | Open |
| AP09 | Dates and data types | Preserve values without conversion errors, truncation or unintended date/time changes. | Validate source update dates, confirmation dates, first-published dates, ATO dates and target data types. | Pending |
| AP10 | Numeric values | Preserve control counts and compliance meaning, scale and precision. | Validate control-count fields and the _Compliant-to-PCT_COMPLIANT mapping. | Pending |
| AP11 | Additional_Visibility | Raw and enriched values use the intended visibility source. | The reviewed SF raw expression uses ALTERNATE_SYSTEM_ADMINISTRATOR_SA. Correction and validation remain open. | Open |
| AP12 | C2S rules and relationships | Preserve status filtering, role expansion and package relationships. | Verify main/AD outputs, Decommissioned exclusion, NULL-status retention and affected package joins. | Pending |
| AP13 | UAT load and refresh | Load the mapped data and refresh affected C2S outputs without unexplained loss, duplication or value changes. | Controlled UAT execution and output reconciliation are outstanding. | Pending |
| AP14 | Final acceptance | Resolve required-field gaps and unexplained differences; complete all applicable checks. | Initial profiling is complete. Full data validation remains open. | Open |
