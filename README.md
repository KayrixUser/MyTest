This document defines the proposed Snowflake source mappings for Matillion ingestion into ACP_Connection and the existing C2S dependencies that must be preserved. It covers authorization packages, hardware, POAMs and Global Trade. Non-Archer ingestion and unrelated notification procedures are outside scope. Archer deviation fields remain included even when their names reference other products.

## 1. Review summary

| Domain | SQL fields compared | PROD Snowflake outputs | Coverage and open findings |
| --- | --- | --- | --- |
| Authorization package | 68; companion PROD table has 60 | 74 | 59 candidates, including a source-expression issue; 9 SQL fields absent. Confirm SAP_ID_RTN and role representation. |
| Hardware | 17 in each SQL table | 19 | 16 candidates; Wireless_Access_Point_Deviation_Approved absent. Validate the HW- identifier prefix and package representation. |
| POAM | 15 in each SQL table | 14 | 11 candidates; 4 SQL fields absent. Final inner joins introduce an owner/package match requirement. |
| Global Trade | 169 | 172 | 163 structural candidates, 1 unconfirmed date candidate and 5 absent SQL fields. Resolve Approval_Date and Created_Date semantics. |

Candidate counts describe structural coverage, not successful data validation. Additional Snowflake columns do not compensate for missing SQL fields. The field tables contain 269 domain-field rows, covering 361 column occurrences across the seven SQL tables.

## 2. Source inventory

All SQL source tables are in ACP_Connection.dbo. All Snowflake objects below are PROD published views. A shared candidate view does not establish equal PROD and PROD_All populations or an approved replacement for both tables.

| ID | SQL Server table | Columns | Current use | Snowflake database.schema.view | Review status |
| --- | --- | --- | --- | --- | --- |
| S1 | ArcherMetrics_AuthorizationPackage_PROD_All | 68 | Main/AD authorization views, BaseTableCopy and baseline consumer | RTX_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_AUTHORIZATION_PACKAGE_ALL | Candidate; coverage and semantic issues open |
| S2 | ArcherMetrics_AuthorizationPackage_PROD | 60 | ACP authorization BaseTable and C2S wrapper | RTX_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_AUTHORIZATION_PACKAGE_ALL | Candidate; S2 population/filter equivalence unverified |
| S3 | ArcherMetrics_Hardware_PROD_All | 17 | Main hardware, BaseTable, CMMC and C2S wrapper paths | RTX_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_HARDWARE_ALL | Candidate; missing field and mapping validation open |
| S4 | ArcherMetrics_Hardware_PROD | 17 | Retained source inventory; active consumer not established | RTX_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_HARDWARE_ALL | Candidate only; consumer and population requirements unverified |
| S5 | ArcherMetrics_POAMs_PROD_All | 15 | Main POAM source view and dimension input | RTX_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_POAM | Candidate; field gaps and join eligibility open |
| S6 | ArcherMetrics_POAMs_PROD | 15 | ACP POAM BaseTable and C2S wrapper | RTX_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_POAM | Candidate; S6 coverage and population equivalence unverified |
| S7 | ArcherMetrics_GlobalTrade_PROD | 169 | Global Trade source view; 12 fields projected | RTX_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_GLOBAL_TRADE | Candidate; dates, package mapping and data validation open |

S1, S3, S5 and S7 feed the main dimension-build views. S2 and S6 also remain relevant through the BaseTable paths. S4 is not classified as unused. The current ACP table writers and the relationship between PROD and PROD_All loads are not established by the view definitions.

## 3. Missing fields and review decisions

The following 19 SQL fields are not exposed by the reviewed Snowflake views. For fields with a current C2S dependency, provide an equivalent source field or agree an explicit C2S dependency change before replacement. Inventory-only gaps require a scope/consumer decision; they are not automatically unused.

| Domain | Missing SQL field | SQL type | Dependency scope |
| --- | --- | --- | --- |
| Authorization package | Alternate_Information_System_Owner_ISO | nvarchar(max) | Main/AD dependency |
| Authorization package | Alternate_Information_System_Owner_ISO_Name | nvarchar(max) | Source inventory; outside main/AD projection |
| Authorization package | Alternate_Information_System_Owner_ISO_UID | nvarchar(max) | Main/AD dependency |
| Authorization package | Count_of_Controls | numeric(18,0) | Main/AD dependency |
| Authorization package | Baseline_Recommendation | nvarchar(max) | Main/AD dependency |
| Authorization package | Control_Set_Version_Number | nvarchar(max) | Main/AD dependency |
| Authorization package | Stub_Entry_RTN | nvarchar(max) | Main/AD dependency |
| Authorization package | SAP_Inactive | nvarchar(max) | Main/AD dependency |
| Authorization package | Authorization_Package_Source | nvarchar(max) | Main/AD dependency |
| Hardware | Wireless_Access_Point_Deviation_Approved | nvarchar(max) | Current C2S source-path dependency |
| POAM | POAM_ID | nvarchar(max) | Current C2S source-path dependency |
| POAM | hRTN_First_Published | nvarchar(max) | Current C2S source-path dependency |
| POAM | POAM_Target_Completion_Date | datetime | Current C2S source-path dependency |
| POAM | Related_Allocated_Controls | nvarchar(max) | Current C2S source-path dependency |
| Global Trade | Second_Level_Reviewer_Collaborator_Name | nvarchar(max) | Source inventory; outside 12-field projection |
| Global Trade | Second_Level_Reviewer_Collaborator_UID | nvarchar(max) | Source inventory; outside 12-field projection |
| Global Trade | First_Level_Reviewer_Collaborator_Name | nvarchar(max) | Source inventory; outside 12-field projection |
| Global Trade | First_Level_Reviewer_Collaborator_UID | nvarchar(max) | Source inventory; outside 12-field projection |
| Global Trade | Approval_Date | datetime | Current 12-field projection |

| Domain / topic | Finding | Review decision required |
| --- | --- | --- |
| Authorization: Additional_Visibility | The view selects AP.ALTERNATE_SYSTEM_ADMINISTRATOR_SA AS ADDITIONAL_VISIBILITY. The enriched visibility fields use ADDITIONAL_VISIBILITY_CTE. | Correct the raw source expression to the intended additional-visibility field and verify raw/derived values. |
| Authorization: SAP_ID_RTN | The output is selected from ARCHER_CONTENT_AUTHORIZATION_PACKAGE_CONTENT_ID. | Confirm equivalence to legacy SAP_ID_RTN before retaining existing joins. |
| Authorization: _Compliant | PCT_COMPLIANT is the proposed counterpart. | Confirm measure meaning, percentage scale and precision. |
| Authorization: roles | Snowflake exposes raw references and UID-before-name ID_NAME strings. The current name parser takes the text before the pipe. | Agree a compatible role representation or explicit transformation; preserve name/UID alignment and current role expansion. |
| Hardware: Hardware_ID | Snowflake applies CONCAT('HW-', HARDWARE_ID). | Compare with existing SQL identifiers; avoid missing or duplicate prefixes. |
| POAM: population | Final inner joins require both a package lookup row and an owner lookup row. The current main SQL view does not require an owner match. | Agree intended eligibility and retain or revise joins explicitly; quantify unmatched records. |
| POAM: date logic | hRTN_First_Published is missing, but drives the historical-date override for POAM_Actual_First_Published and DaysOpen. | Provide the historical source value or approve a documented change to the calculation. Milestone and target completion dates are separate. |
| Global Trade: Created_Date | FIRST_PUBLISHED is only a possible counterpart; the creation event is unconfirmed. | Confirm the correct source event/value. Do not treat FIRST_PUBLISHED as an approved replacement. |
| Hardware, POAM, Global Trade: package relationships | AUTHORIZATION_PACKAGE is raw references; AUTHORIZATION_PACKAGE_NAME is an aggregated resolved-name list. | Confirm the legacy representation and multi-package handling against existing LIKE/equality joins and filters. |
| All domains: source contract | Exact Snowflake type parameters, nullability and row-level equivalence are unverified. | Confirm target conversions, source grain, matching keys, refresh alignment, counts and field-value parity. |
| SQL source populations | A common field set does not explain PROD versus PROD_All loading or filtering. | Confirm the ingestion mapping and required population for each ACP table. |

## 4. Field comparison conventions

- SQL types are retained from the source metadata. All source fields are nullable; observed identity/computed flags are 0. Text collation is SQL_Latin1_General_CP1_CI_AS. This does not establish business-key uniqueness or a complete constraint inventory.
- Snowflake types are recorded at the available family level: VARCHAR, NUMBER and TIMESTAMP_NTZ. Lengths, numeric precision/scale, timestamp precision and nullability require confirmation before target conversions are approved.
- Absent means the field does not exist in that SQL companion table. Not exposed means it is absent from the reviewed Snowflake view. Same-name and renamed fields are mapping candidates; values are not yet validated.
- Source Last_Updated is text in SQL. C2S LastUpdatedDate is a separate datetime2(3) load timestamp. Source timestamps and load timestamps are not interchangeable.
- Main/AD identify the C2S authorization source views; Base identifies the ACP/C2S BaseTable path. Source only means outside the specified main projection, not proven unused elsewhere.

## 5. Authorization package field comparison

SQL: S1 PROD_All and S2 PROD. Snowflake: RTX_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_AUTHORIZATION_PACKAGE_ALL.

| SQL field | S1 PROD_All type | S2 PROD type | SQL use / mapping note | Snowflake field / candidate | Snowflake type |
| --- | --- | --- | --- | --- | --- |
| Content_ID | nvarchar(max) | varchar(max) | Source identifier; PROD uses varchar and PROD_All uses nvarchar. Renamed content ID; NUMBER to SQL text conversion pending. | ARCHER_CONTENT_AUTHORIZATION_PACKAGE_CONTENT_ID | NUMBER |
| Last_Updated | nvarchar(max) | nvarchar(max) | Base path: direct attribute. Renamed timestamp; SQL stores text. | ARCHER_CONTENT_AUTHORIZATION_PACKAGE_LAST_UPDATED | TIMESTAMP_NTZ |
| SAP_ID | nvarchar(max) | nvarchar(max) | Source field; not selected by main or AD view | SAP_ID | VARCHAR |
| Authorization_Package_Name | nvarchar(max) | nvarchar(max) | Main and AD: direct package attribute. Aggregated matched employee names; validate list behavior. | AUTHORIZATION_PACKAGE_NAME | VARCHAR |
| Acronym | nvarchar(max) | nvarchar(max) | Main and AD: direct package attribute | ACRONYM | VARCHAR |
| Operational_Status | nvarchar(max) | nvarchar(max) | Main and AD: direct package attribute | OPERATIONAL_STATUS | VARCHAR |
| Information_Classification | nvarchar(max) | nvarchar(max) | Main and AD: direct package attribute | INFORMATION_CLASSIFICATION | VARCHAR |
| ATOIATO_Date | datetime | datetime | Main and AD: direct package attribute. Timestamp conversion and values require validation. | ATOIATO_DATE | TIMESTAMP_NTZ |
| ATOIATO_Expiration_Date | datetime | datetime | Main and AD: direct package attribute. Timestamp conversion and values require validation. | ATOIATO_EXPIRATION_DATE | TIMESTAMP_NTZ |
| Connectivity | nvarchar(max) | nvarchar(max) | Main and AD: direct package attribute | CONNECTIVITY | VARCHAR |
| Information_Owner_IO | nvarchar(max) | nvarchar(max) | Source field; not selected by main or AD view. Raw reference string; legacy role format requires validation. | INFORMATION_OWNER_IO | VARCHAR |
| Information_System_Owner_ISO | nvarchar(max) | nvarchar(max) | Main: split into role rows; AD: names aggregated. Raw reference string; legacy role format requires validation. | INFORMATION_SYSTEM_OWNER_ISO | VARCHAR |
| Senior_Information_Systems_Security_Officer_SISSO | nvarchar(max) | nvarchar(max) | Source field; not selected by main or AD view. Raw reference string; legacy role format requires validation. | SENIOR_INFORMATION_SYSTEMS_SECURITY_OFFICER_SISSO | VARCHAR |
| Authorizing_Official_AO | nvarchar(max) | nvarchar(max) | Main: split into role rows; AD: direct value. Raw reference string; legacy role format requires validation. | AUTHORIZING_OFFICIAL_AO | VARCHAR |
| Information_System_Security_Officer_ISSO | nvarchar(max) | nvarchar(max) | Main: split into role rows; AD: names aggregated. Raw reference string; legacy role format requires validation. | INFORMATION_SYSTEM_SECURITY_OFFICER_ISSO | VARCHAR |
| Security_Control_Assessor_SCA | nvarchar(max) | nvarchar(max) | Main: split into role rows; AD: direct value. Raw reference string; legacy role format requires validation. | SECURITY_CONTROL_ASSESSOR_SCA | VARCHAR |
| Authorizing_Official_Designated_Representative_AODR | nvarchar(max) | nvarchar(max) | Source field; not selected by main or AD view. Raw reference string; legacy role format requires validation. | AUTHORIZING_OFFICIAL_DESIGNATED_REPRESENTATIVE_AODR | VARCHAR |
| Package_Acronym | nvarchar(max) | nvarchar(max) | Base path: direct attribute | PACKAGE_ACRONYM | VARCHAR |
| _Compliant | numeric(18,0) | numeric(18,0) | Source field; not selected by main or AD view. PCT_COMPLIANT candidate; confirm business meaning and percentage scale. | PCT_COMPLIANT | NUMBER |
| HRC_Related_Package_Acronyms | nvarchar(max) | nvarchar(max) | Base path: direct attribute | HRC_RELATED_PACKAGE_ACRONYMS | VARCHAR |
| SAP_ID_RTN | nvarchar(max) | nvarchar(max) | Main and AD: direct package attribute. Selected from Archer content ID; legacy identifier equivalence unconfirmed. | SAP_ID_RTN | NUMBER |
| Count_of_Controls_with_Open_POAMs | numeric(18,0) | numeric(18,0) | Source field; not selected by main or AD view. NUMBER precision/scale and values require validation. | COUNT_OF_CONTROLS_WITH_OPEN_POAMS | NUMBER |
| Additional_Visibility | nvarchar(max) | nvarchar(max) | Source field; not selected by main or AD view. Definition issue: selected from ALTERNATE_SYSTEM_ADMINISTRATOR_SA. | ADDITIONAL_VISIBILITY | VARCHAR |
| Alternate_Information_System_Owner_ISO | nvarchar(max) | nvarchar(max) | Main: split into role rows; AD: names aggregated. Missing; source-field gap. | Not exposed | Not available |
| Alternate_Security_Control_Assessor_SCA | nvarchar(max) | nvarchar(max) | Source field; not selected by main or AD view. Raw reference string; legacy role format requires validation. | ALTERNATE_SECURITY_CONTROL_ASSESSOR_SCA | VARCHAR |
| Alternate_Information_System_Security_Officer_ISSO | nvarchar(max) | nvarchar(max) | Main: split into role rows; AD: names aggregated. Raw reference string; legacy role format requires validation. | ALTERNATE_INFORMATION_SYSTEM_SECURITY_OFFICER_ISSO | VARCHAR |
| Alternate_System_Administrator_SA | nvarchar(max) | nvarchar(max) | Main: split into role rows; AD: names aggregated. Raw reference string; legacy role format requires validation. | ALTERNATE_SYSTEM_ADMINISTRATOR_SA | VARCHAR |
| Information_System_Security_Manager_ISSM | nvarchar(max) | nvarchar(max) | Source field; not selected by main or AD view. Raw reference string; legacy role format requires validation. | INFORMATION_SYSTEM_SECURITY_MANAGER_ISSM | VARCHAR |
| System_Administrator_SA | nvarchar(max) | nvarchar(max) | Main: split into role rows; AD: names aggregated. Raw reference string; legacy role format requires validation. | SYSTEM_ADMINISTRATOR_SA | VARCHAR |
| Entity | nvarchar(max) | nvarchar(max) | Main and AD: direct package attribute. Resolved-name candidate; raw references are separate outputs. | ENTITY_NAME | VARCHAR |
| Business | nvarchar(max) | nvarchar(max) | Main and AD: direct package attribute. Resolved-name candidate; raw references are separate outputs. | BUSINESS_NAME | VARCHAR |
| Information_Owner_IO_Name | nvarchar(max) | nvarchar(max) | Source field; not selected by main or AD view. Aggregated matched employee names; validate list behavior. | INFORMATION_OWNER_IO_NAME_ONLY | VARCHAR |
| Information_Owner_IO_UID | nvarchar(max) | nvarchar(max) | Source field; not selected by main or AD view. Aggregated matched UIDs; unmatched tokens may be omitted. | INFORMATION_OWNER_IO_ID_ONLY | VARCHAR |
| Information_System_Owner_ISO_Name | nvarchar(max) | nvarchar(max) | Base path: direct attribute. Aggregated matched employee names; validate list behavior. | INFORMATION_SYSTEM_OWNER_ISO_NAME_ONLY | VARCHAR |
| Information_System_Owner_ISO_UID | nvarchar(max) | nvarchar(max) | AD: retained role UID; selected roles support later email enrichment. Aggregated matched UIDs; unmatched tokens may be omitted. | INFORMATION_SYSTEM_OWNER_ISO_ID_ONLY | VARCHAR |
| Senior_Information_Systems_Security_Officer_SISSO_Name | nvarchar(max) | nvarchar(max) | Source field; not selected by main or AD view. Aggregated matched employee names; validate list behavior. | SENIOR_INFORMATION_SYSTEMS_SECURITY_OFFICER_SISSO_NAME_ONLY | VARCHAR |
| Senior_Information_Systems_Security_Officer_SISSO_UID | nvarchar(max) | nvarchar(max) | Source field; not selected by main or AD view. Aggregated matched UIDs; unmatched tokens may be omitted. | SENIOR_INFORMATION_SYSTEMS_SECURITY_OFFICER_SISSO_ID_ONLY | VARCHAR |
| Authorizing_Official_AO_Name | nvarchar(max) | nvarchar(max) | Base path: direct attribute. Aggregated matched employee names; validate list behavior. | AUTHORIZING_OFFICIAL_AO_NAME_ONLY | VARCHAR |
| Authorizing_Official_AO_UID | nvarchar(max) | nvarchar(max) | AD: retained role UID; selected roles support later email enrichment. Aggregated matched UIDs; unmatched tokens may be omitted. | AUTHORIZING_OFFICIAL_AO_ID_ONLY | VARCHAR |
| Information_System_Security_Officer_ISSO_Name | nvarchar(max) | nvarchar(max) | Base path: direct attribute. Aggregated matched employee names; validate list behavior. | INFORMATION_SYSTEM_SECURITY_OFFICER_ISSO_NAME_ONLY | VARCHAR |
| Information_System_Security_Officer_ISSO_UID | nvarchar(max) | nvarchar(max) | AD: retained role UID; selected roles support later email enrichment. Aggregated matched UIDs; unmatched tokens may be omitted. | INFORMATION_SYSTEM_SECURITY_OFFICER_ISSO_ID_ONLY | VARCHAR |
| Security_Control_Assessor_SCA_Name | nvarchar(max) | nvarchar(max) | Base path: direct attribute. Aggregated matched employee names; validate list behavior. | SECURITY_CONTROL_ASSESSOR_SCA_NAME_ONLY | VARCHAR |
| Security_Control_Assessor_SCA_UID | nvarchar(max) | nvarchar(max) | AD: retained role UID; selected roles support later email enrichment. Aggregated matched UIDs; unmatched tokens may be omitted. | SECURITY_CONTROL_ASSESSOR_SCA_ID_ONLY | VARCHAR |
| Authorizing_Official_Designated_Representative_AODR_Name | nvarchar(max) | nvarchar(max) | Source field; not selected by main or AD view. Aggregated matched employee names; validate list behavior. | AUTHORIZING_OFFICIAL_DESIGNATED_REPRESENTATIVE_AODR_NAME_ONLY | VARCHAR |
| Authorizing_Official_Designated_Representative_AODR_UID | nvarchar(max) | nvarchar(max) | Source field; not selected by main or AD view. Aggregated matched UIDs; unmatched tokens may be omitted. | AUTHORIZING_OFFICIAL_DESIGNATED_REPRESENTATIVE_AODR_ID_ONLY | VARCHAR |
| Additional_Visibility_Name | nvarchar(max) | nvarchar(max) | Source field; not selected by main or AD view. Aggregated matched employee names; validate list behavior. | ADDITIONAL_VISIBILITY_NAME_ONLY | VARCHAR |
| Additional_Visibility_UID | nvarchar(max) | nvarchar(max) | Source field; not selected by main or AD view. Aggregated matched UIDs; unmatched tokens may be omitted. | ADDITIONAL_VISIBILITY_ID_ONLY | VARCHAR |
| Alternate_Information_System_Owner_ISO_Name | nvarchar(max) | nvarchar(max) | Source field; not selected by main or AD view. Missing; source-field gap. | Not exposed | Not available |
| Alternate_Information_System_Owner_ISO_UID | nvarchar(max) | nvarchar(max) | AD: retained role UID; selected roles support later email enrichment. Missing; source-field gap. | Not exposed | Not available |
| Alternate_Security_Control_Assessor_SCA_Name | nvarchar(max) | nvarchar(max) | Source field; not selected by main or AD view. Aggregated matched employee names; validate list behavior. | ALTERNATE_SECURITY_CONTROL_ASSESSOR_SCA_NAME_ONLY | VARCHAR |
| Alternate_Security_Control_Assessor_SCA_UID | nvarchar(max) | nvarchar(max) | Source field; not selected by main or AD view. Aggregated matched UIDs; unmatched tokens may be omitted. | ALTERNATE_SECURITY_CONTROL_ASSESSOR_SCA_ID_ONLY | VARCHAR |
| Alternate_Information_System_Security_Officer_ISSO_Name | nvarchar(max) | nvarchar(max) | Source field; not selected by main or AD view. Aggregated matched employee names; validate list behavior. | ALTERNATE_INFORMATION_SYSTEM_SECURITY_OFFICER_ISSO_NAME_ONLY | VARCHAR |
| Alternate_Information_System_Security_Officer_ISSO_UID | nvarchar(max) | nvarchar(max) | AD: retained role UID; selected roles support later email enrichment. Aggregated matched UIDs; unmatched tokens may be omitted. | ALTERNATE_INFORMATION_SYSTEM_SECURITY_OFFICER_ISSO_ID_ONLY | VARCHAR |
| Alternate_System_Administrator_SA_Name | nvarchar(max) | nvarchar(max) | Source field; not selected by main or AD view. Aggregated matched employee names; validate list behavior. | ALTERNATE_SYSTEM_ADMINISTRATOR_SA_NAME_ONLY | VARCHAR |
| Alternate_System_Administrator_SA_UID | nvarchar(max) | nvarchar(max) | AD: retained role UID; selected roles support later email enrichment. Aggregated matched UIDs; unmatched tokens may be omitted. | ALTERNATE_SYSTEM_ADMINISTRATOR_SA_ID_ONLY | VARCHAR |
| Information_System_Security_Manager_ISSM_Name | nvarchar(max) | nvarchar(max) | Source field; not selected by main or AD view. Aggregated matched employee names; validate list behavior. | INFORMATION_SYSTEM_SECURITY_MANAGER_ISSM_NAME_ONLY | VARCHAR |
| Information_System_Security_Manager_ISSM_UID | nvarchar(max) | nvarchar(max) | Source field; not selected by main or AD view. Aggregated matched UIDs; unmatched tokens may be omitted. | INFORMATION_SYSTEM_SECURITY_MANAGER_ISSM_ID_ONLY | VARCHAR |
| System_Administrator_SA_Name | nvarchar(max) | nvarchar(max) | Base path: direct attribute. Aggregated matched employee names; validate list behavior. | SYSTEM_ADMINISTRATOR_SA_NAME_ONLY | VARCHAR |
| System_Administrator_SA_UID | nvarchar(max) | nvarchar(max) | AD: retained role UID; selected roles support later email enrichment. Aggregated matched UIDs; unmatched tokens may be omitted. | SYSTEM_ADMINISTRATOR_SA_ID_ONLY | VARCHAR |
| ConfirmedInArcher | datetime | datetime | Base path: direct attribute. Renamed timestamp; confirm event meaning and conversion. | ARCHER_CONTENT_AUTHORIZATION_PACKAGE_CONFIRMED_IN_ARCHER | TIMESTAMP_NTZ |
| Count_of_Controls | numeric(18,0) | Absent | Main and AD: direct package attribute. Missing; source-field gap. | Not exposed | Not available |
| Baseline_Recommendation | nvarchar(max) | Absent | Main and AD: direct package attribute. Missing; source-field gap. | Not exposed | Not available |
| Control_Set_Version_Number | nvarchar(max) | Absent | Main and AD: direct package attribute. Missing; source-field gap. | Not exposed | Not available |
| Stub_Entry_RTN | nvarchar(max) | Absent | Main and AD: direct package attribute. Missing; source-field gap. | Not exposed | Not available |
| SAP_Inactive | nvarchar(max) | Absent | Main and AD: direct package attribute. Missing; source-field gap. | Not exposed | Not available |
| In_Continuous_Monitoring | nvarchar(max) | Absent | Main and AD: direct package attribute | IN_CONTINUOUS_MONITORING | VARCHAR |
| Authorization_Package_Source | nvarchar(max) | Absent | Main and AD: direct package attribute. Missing; source-field gap. | Not exposed | Not available |
| First_Published | datetime | Absent | Source date; PROD_All only. Renamed timestamp; confirm event meaning and conversion. | ARCHER_CONTENT_AUTHORIZATION_PACKAGE_FIRST_PUBLISHED | TIMESTAMP_NTZ |

The eight S1-only fields are Count_of_Controls, Baseline_Recommendation, Control_Set_Version_Number, Stub_Entry_RTN, SAP_Inactive, In_Continuous_Monitoring, Authorization_Package_Source and First_Published. The first seven participate in the main/AD source projections; First_Published is part of the source inventory.

## 6. Hardware field comparison

SQL: S3 PROD_All and S4 PROD. Snowflake: RTX_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_HARDWARE_ALL.

| SQL field | S3 PROD_All type | S4 PROD type | SQL use / mapping note | Snowflake field / candidate | Snowflake type |
| --- | --- | --- | --- | --- | --- |
| Content_ID | nvarchar(max) | varchar(max) | Raw identifier; text-to-number mapping. Confirm key values and target conversion. | ARCHER_CONTENT_HARDWARE_CONTENT_ID | NUMBER |
| Last_Updated | nvarchar(max) | nvarchar(max) | BaseTable paths; SQL text versus Snowflake timestamp. Confirm format, precision and update-event meaning. | ARCHER_CONTENT_HARDWARE_LAST_UPDATED | TIMESTAMP_NTZ |
| Hardware_Name | nvarchar(max) | nvarchar(max) | Main and BaseTable paths; direct column correspondence. | HARDWARE_NAME | VARCHAR |
| Host_Name | nvarchar(max) | nvarchar(max) | Direct host value. Retain SQL-derived Archer_Asset_Name and HostNameFirstPartHW separately. | HOST_NAME | VARCHAR |
| Internal_IP_Address | nvarchar(max) | nvarchar(max) | Main and BaseTable paths; direct column correspondence. | INTERNAL_IP_ADDRESS | VARCHAR |
| MAC_Address | nvarchar(max) | nvarchar(max) | Main and BaseTable paths; direct column correspondence. | MAC_ADDRESS | VARCHAR |
| Hardware_ID | nvarchar(max) | varchar(255) | Snowflake applies CONCAT('HW-', HARDWARE_ID). Confirm existing SQL prefix/values and target length. | HARDWARE_ID | VARCHAR |
| Authorization_Package | nvarchar(max) | nvarchar(max) | Candidate: resolved package names. Snowflake AUTHORIZATION_PACKAGE is the raw reference list. Compare legacy SQL values before accepting the name mapping. | AUTHORIZATION_PACKAGE_NAME | VARCHAR |
| Splunk_Deviation_Approved | nvarchar(max) | nvarchar(max) | Main and BaseTable paths; direct column correspondence. | SPLUNK_DEVIATION_APPROVED | VARCHAR |
| Tanium_Deviation_Approved | nvarchar(max) | nvarchar(max) | Main and BaseTable paths; direct column correspondence. | TANIUM_DEVIATION_APPROVED | VARCHAR |
| Access_Management_Deviation_Approved | nvarchar(max) | nvarchar(max) | Main and BaseTable paths; direct column correspondence. | ACCESS_MANAGEMENT_DEVIATION_APPROVED | VARCHAR |
| Privileged_Access_Management_Deviation_Approved | nvarchar(max) | nvarchar(max) | Main and BaseTable paths; direct column correspondence. | PRIVILEGED_ACCESS_MANAGEMENT_DEVIATION_APPROVED | VARCHAR |
| Password_Management_Deviation_Approved | nvarchar(max) | nvarchar(max) | Main and BaseTable paths; direct column correspondence. | PASSWORD_MANAGEMENT_DEVIATION_APPROVED | VARCHAR |
| Session_Control_Deviation_Approved | nvarchar(max) | nvarchar(max) | Main and BaseTable paths; direct column correspondence. | SESSION_CONTROL_DEVIATION_APPROVED | VARCHAR |
| ConfirmedInArcher | datetime | datetime | BaseTable paths; renamed timestamp. Confirm datetime conversion, precision and event meaning. | ARCHER_CONTENT_HARDWARE_CONFIRMED_IN_ARCHER | TIMESTAMP_NTZ |
| Type | nvarchar(max) | nvarchar(max) | Main and BaseTable paths; direct column correspondence. | TYPE | VARCHAR |
| Wireless_Access_Point_Deviation_Approved | nvarchar(max) | nvarchar(max) | Missing from the 19-column view. Used by the documented main hardware source view; not selected by the BaseTable/CMMC paths. | Not exposed | Not available |

Archer_Asset_Name and HostNameFirstPartHW remain SQL-derived outputs of ParseStringValue(Host_Name), returning varchar(max). They are not additional raw ACP columns. The BaseTableCMMC view has no CMMC-specific filter.

## 7. POAM field comparison

SQL: S5 PROD_All and S6 PROD. Snowflake: RTX_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_POAM.

| SQL field | S5 PROD_All type | S6 PROD type | SQL use / mapping note | Snowflake field / candidate | Snowflake type |
| --- | --- | --- | --- | --- | --- |
| Content_ID | nvarchar(max) | varchar(max) | Renamed source identifier; SQL text versus Snowflake NUMBER. Confirm key values and conversion. Do not substitute this for POAM_ID without evidence. | ARCHER_CONTENT_POAMS_CONTENT_ID | NUMBER |
| Last_Updated | nvarchar(max) | nvarchar(max) | BaseTable path. SQL stores text; confirm timestamp meaning, formatting and precision. | ARCHER_CONTENT_POAMS_LAST_UPDATED | TIMESTAMP_NTZ |
| POAM_ID | nvarchar(max) | nvarchar(max) | Missing. Required by the main source view and BaseTable path. Content_ID is a different field; an equivalent POAM identifier must be provided or explicitly validated. | Not exposed | Not available |
| First_Published | datetime | datetime | Renamed timestamp used by main and BaseTable paths. Validate conversion; it does not establish equivalence to hRTN_First_Published. | ARCHER_CONTENT_POAMS_FIRST_PUBLISHED | TIMESTAMP_NTZ |
| Name | nvarchar(max) | nvarchar(max) | Main and BaseTable paths; direct column correspondence. | NAME | VARCHAR |
| Milestone_Estimated_Completion_Date | datetime | datetime | Main and BaseTable paths; datetime conversion and values pending. This is distinct from POAM_Target_Completion_Date. | MILESTONE_ESTIMATED_COMPLETION_DATE | TIMESTAMP_NTZ |
| Overall_Status | nvarchar(max) | nvarchar(max) | Main and BaseTable paths; existing main SQL view excludes Completed and NULL statuses. | OVERALL_STATUS | VARCHAR |
| POAM_Owner | nvarchar(max) | nvarchar(max) | BaseTable path; raw Snowflake owner references. Validate representation against the legacy role string; do not assume name/UID formatting matches. | POAM_OWNER | VARCHAR |
| Authorization_Package | nvarchar(max) | nvarchar(max) | Resolved-name candidate. Snowflake AUTHORIZATION_PACKAGE contains raw references. Validate names and multi-package representation against the current SQL package-name equality join. | AUTHORIZATION_PACKAGE_NAME | VARCHAR |
| hRTN_First_Published | nvarchar(max) | nvarchar(max) | Missing. Used by POAM_Actual_First_Published and DaysOpen when populated. Do not silently replace it with First_Published. | Not exposed | Not available |
| POAM_Owner_Name | nvarchar(max) | nvarchar(max) | Candidate aggregated employee names from Workday LDAP matches. Validate list delimiters, ordering, duplicates and missing-user behavior. | POAM_OWNER_NAME_ONLY | VARCHAR |
| POAM_Owner_UID | nvarchar(max) | nvarchar(max) | Candidate aggregated owner IDs after matching Workday LDAP UID. Unmatched tokens are excluded from the enriched list. | POAM_OWNER_ID_ONLY | VARCHAR |
| ConfirmedInArcher | datetime | datetime | BaseTable path; renamed timestamp. Validate event meaning, datetime conversion and precision. | ARCHER_CONTENT_POAMS_CONFIRMED_IN_ARCHER | TIMESTAMP_NTZ |
| POAM_Target_Completion_Date | datetime | datetime | Missing. Selected by the main POAM source view; milestone completion date is not an established substitute. | Not exposed | Not available |
| Related_Allocated_Controls | nvarchar(max) | nvarchar(max) | Missing. Selected by the main POAM source view; supply the source field or resolve its current dependency. | Not exposed | Not available |

POAM_ID is distinct from content ID; no substitute POAM identifier is approved. First_Published is not an established replacement for hRTN_First_Published. Milestone_Estimated_Completion_Date is not an established replacement for POAM_Target_Completion_Date.

## 8. Global Trade field comparison

SQL: S7 ArcherMetrics_GlobalTrade_PROD. Snowflake: RTX_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_GLOBAL_TRADE. No GlobalTrade_PROD_All table is in the source inventory.

The table retains SQL column IDs, including gaps 122 and 123: 169 fields, not 171. Twelve are projected by the current C2S view; the other 157 remain in the wider source inventory. Created_Date has an unconfirmed date candidate and Approval_Date is missing.

| Column ID | SQL field | SQL type | C2S output / mapping note | Snowflake field / candidate | Snowflake type |
| --- | --- | --- | --- | --- | --- |
| 1 | Content_ID | nvarchar(max) | Source only; renamed content ID. NUMBER to SQL text conversion pending; distinct from the assessment ID. | ARCHER_CONTENT_GLOBAL_TRADE_ASSESSMENT_CONTENT_ID | NUMBER |
| 2 | Last_Updated | nvarchar(max) | Last_Updated_GT; renamed timestamp. SQL stores text; confirm format and event meaning. | ARCHER_CONTENT_GLOBAL_TRADE_ASSESSMENT_LAST_UPDATED | TIMESTAMP_NTZ |
| 3 | Created_Date | datetime | Created_Date_GT; UNCONFIRMED DATE CANDIDATE. FIRST_PUBLISHED is not an established equivalent. | ARCHER_CONTENT_GLOBAL_TRADE_ASSESSMENT_FIRST_PUBLISHED | TIMESTAMP_NTZ |
| 4 | Global_Trade_Assessment_ID | nvarchar(max) | Global_Trade_Assessment_ID; direct source identifier. Validate NUMBER to SQL text conversion. | GLOBAL_TRADE_ASSESSMENT_ID | NUMBER |
| 5 | Respondent_Status | nvarchar(max) | Respondent_Status | RESPONDENT_STATUS | VARCHAR |
| 6 | Workflow_Current_Node | nvarchar(max) | Source only | WORKFLOW_CURRENT_NODE | VARCHAR |
| 7 | Workflow_Process_Version | numeric(18,0) | Source only; validate NUMBER precision/scale against numeric(18,0). | WORKFLOW_PROCESS_VERSION | NUMBER |
| 8 | Workflow_Job_Status | nvarchar(max) | Source only | WORKFLOW_JOB_STATUS | VARCHAR |
| 9 | Background_Information_and_Prior_Global_Trade_Reviews_23 | nvarchar(max) | Source only | BACKGROUND_INFORMATION_AND_PRIOR_GLOBAL_TRADE_REVIEWS_23 | VARCHAR |
| 10 | Background_Information_and_Prior_Global_Trade_Reviews_24 | nvarchar(max) | Source only | BACKGROUND_INFORMATION_AND_PRIOR_GLOBAL_TRADE_REVIEWS_24 | VARCHAR |
| 11 | Background_Information_and_Prior_Global_Trade_Reviews_25 | nvarchar(max) | Source only | BACKGROUND_INFORMATION_AND_PRIOR_GLOBAL_TRADE_REVIEWS_25 | VARCHAR |
| 12 | Background_Information_and_Prior_Global_Trade_Reviews_26 | nvarchar(max) | Source only | BACKGROUND_INFORMATION_AND_PRIOR_GLOBAL_TRADE_REVIEWS_26 | VARCHAR |
| 13 | Background_Information_and_Prior_Global_Trade_Reviews_21 | nvarchar(max) | Source only | BACKGROUND_INFORMATION_AND_PRIOR_GLOBAL_TRADE_REVIEWS_21 | VARCHAR |
| 14 | Background_Information_and_Prior_Global_Trade_Reviews_22 | nvarchar(max) | Source only | BACKGROUND_INFORMATION_AND_PRIOR_GLOBAL_TRADE_REVIEWS_22 | VARCHAR |
| 15 | System_Type_Decision_Tree_31 | nvarchar(max) | Technical_data | SYSTEM_TYPE_DECISION_TREE_31 | VARCHAR |
| 16 | System_Type_Decision_Tree_32 | nvarchar(max) | Source only | SYSTEM_TYPE_DECISION_TREE_32 | VARCHAR |
| 17 | System_Type_Decision_Tree_34 | nvarchar(max) | Source only | SYSTEM_TYPE_DECISION_TREE_34 | VARCHAR |
| 18 | System_Type_Decision_Tree_35 | nvarchar(max) | Source only | SYSTEM_TYPE_DECISION_TREE_35 | VARCHAR |
| 19 | System_Type_Decision_Tree_36 | nvarchar(max) | Source only | SYSTEM_TYPE_DECISION_TREE_36 | VARCHAR |
| 20 | System_Type_Decision_Tree_37 | nvarchar(max) | Source only | SYSTEM_TYPE_DECISION_TREE_37 | VARCHAR |
| 21 | System_Type_Decision_Tree_38 | nvarchar(max) | Data_location | SYSTEM_TYPE_DECISION_TREE_38 | VARCHAR |
| 22 | System_Type_Decision_Tree_39 | nvarchar(max) | Information_Type | SYSTEM_TYPE_DECISION_TREE_39 | VARCHAR |
| 23 | Type_A_Controls_Questions_41 | nvarchar(max) | Source only | TYPE_A_CONTROLS_QUESTIONS_41 | VARCHAR |
| 24 | Type_A_Controls_Questions_42 | nvarchar(max) | Source only | TYPE_A_CONTROLS_QUESTIONS_42 | VARCHAR |
| 25 | Type_A_Controls_Questions_43 | nvarchar(max) | Source only | TYPE_A_CONTROLS_QUESTIONS_43 | VARCHAR |
| 26 | Type_A_Controls_Questions_44 | nvarchar(max) | Source only | TYPE_A_CONTROLS_QUESTIONS_44 | VARCHAR |
| 27 | Type_A_Controls_Questions_45 | nvarchar(max) | Source only | TYPE_A_CONTROLS_QUESTIONS_45 | VARCHAR |
| 28 | Type_A_Controls_Questions_46 | nvarchar(max) | Source only | TYPE_A_CONTROLS_QUESTIONS_46 | VARCHAR |
| 29 | Type_A_Controls_Questions_47 | nvarchar(max) | Source only | TYPE_A_CONTROLS_QUESTIONS_47 | VARCHAR |
| 30 | Type_A_Controls_Questions_48 | nvarchar(max) | Source only | TYPE_A_CONTROLS_QUESTIONS_48 | VARCHAR |
| 31 | Type_B_Controls_Questions_51 | nvarchar(max) | Source only | TYPE_B_CONTROLS_QUESTIONS_51 | VARCHAR |
| 32 | Type_B_Controls_Questions_52 | nvarchar(max) | Source only | TYPE_B_CONTROLS_QUESTIONS_52 | VARCHAR |
| 33 | Type_B_Controls_Questions_53 | nvarchar(max) | Source only | TYPE_B_CONTROLS_QUESTIONS_53 | VARCHAR |
| 34 | Type_B_Controls_Questions_54 | nvarchar(max) | Source only | TYPE_B_CONTROLS_QUESTIONS_54 | VARCHAR |
| 35 | Type_B_Controls_Questions_55 | nvarchar(max) | Source only | TYPE_B_CONTROLS_QUESTIONS_55 | VARCHAR |
| 36 | Type_B_Controls_Questions_56 | nvarchar(max) | Source only | TYPE_B_CONTROLS_QUESTIONS_56 | VARCHAR |
| 37 | Type_B_Controls_Questions_57 | nvarchar(max) | Source only | TYPE_B_CONTROLS_QUESTIONS_57 | VARCHAR |
| 38 | Type_B_Controls_Questions_58 | nvarchar(max) | Source only | TYPE_B_CONTROLS_QUESTIONS_58 | VARCHAR |
| 39 | Type_C_Controls_Questions_61 | nvarchar(max) | Source only | TYPE_C_CONTROLS_QUESTIONS_61 | VARCHAR |
| 40 | Type_C_Controls_Questions_62 | nvarchar(max) | Source only | TYPE_C_CONTROLS_QUESTIONS_62 | VARCHAR |
| 41 | Type_C_Controls_Questions_63 | nvarchar(max) | Source only | TYPE_C_CONTROLS_QUESTIONS_63 | VARCHAR |
| 42 | Type_C_Controls_Questions_64 | nvarchar(max) | Source only | TYPE_C_CONTROLS_QUESTIONS_64 | VARCHAR |
| 43 | Type_C_Controls_Questions_65 | nvarchar(max) | Source only | TYPE_C_CONTROLS_QUESTIONS_65 | VARCHAR |
| 44 | Type_C_Controls_Questions_66 | nvarchar(max) | Source only | TYPE_C_CONTROLS_QUESTIONS_66 | VARCHAR |
| 45 | Type_C_Controls_Questions_67 | nvarchar(max) | Source only | TYPE_C_CONTROLS_QUESTIONS_67 | VARCHAR |
| 46 | Type_C_Controls_Questions_68 | nvarchar(max) | Source only | TYPE_C_CONTROLS_QUESTIONS_68 | VARCHAR |
| 47 | Type_C_Controls_Questions_69 | nvarchar(max) | Source only | TYPE_C_CONTROLS_QUESTIONS_69 | VARCHAR |
| 48 | Type_D_Controls_Questions_71 | nvarchar(max) | Source only | TYPE_D_CONTROLS_QUESTIONS_71 | VARCHAR |
| 49 | Type_D_Controls_Questions_72 | nvarchar(max) | Source only | TYPE_D_CONTROLS_QUESTIONS_72 | VARCHAR |
| 50 | Type_D_Controls_Questions_73 | nvarchar(max) | Source only | TYPE_D_CONTROLS_QUESTIONS_73 | VARCHAR |
| 51 | Type_D_Controls_Questions_74 | nvarchar(max) | Source only | TYPE_D_CONTROLS_QUESTIONS_74 | VARCHAR |
| 52 | Type_D_Controls_Questions_75 | nvarchar(max) | Source only | TYPE_D_CONTROLS_QUESTIONS_75 | VARCHAR |
| 53 | Type_D_Controls_Questions_76 | nvarchar(max) | Source only | TYPE_D_CONTROLS_QUESTIONS_76 | VARCHAR |
| 54 | Type_D_Controls_Questions_77 | nvarchar(max) | Source only | TYPE_D_CONTROLS_QUESTIONS_77 | VARCHAR |
| 55 | Type_D_Controls_Questions_78 | nvarchar(max) | Source only | TYPE_D_CONTROLS_QUESTIONS_78 | VARCHAR |
| 56 | Type_D_Controls_Questions_710 | nvarchar(max) | Source only | TYPE_D_CONTROLS_QUESTIONS_710 | VARCHAR |
| 57 | Type_D_Controls_Questions_711 | nvarchar(max) | Source only | TYPE_D_CONTROLS_QUESTIONS_711 | VARCHAR |
| 58 | Type_D_Controls_Questions_712 | nvarchar(max) | Source only | TYPE_D_CONTROLS_QUESTIONS_712 | VARCHAR |
| 59 | Type_D_Controls_Questions_713 | nvarchar(max) | Source only | TYPE_D_CONTROLS_QUESTIONS_713 | VARCHAR |
| 60 | Type_D_Controls_Questions_714 | nvarchar(max) | Source only | TYPE_D_CONTROLS_QUESTIONS_714 | VARCHAR |
| 61 | Type_D_Controls_Questions_715 | nvarchar(max) | Source only | TYPE_D_CONTROLS_QUESTIONS_715 | VARCHAR |
| 62 | Type_D_Controls_Questions_716 | nvarchar(max) | Source only | TYPE_D_CONTROLS_QUESTIONS_716 | VARCHAR |
| 63 | Type_E_Controls_Questions_81 | nvarchar(max) | Source only | TYPE_E_CONTROLS_QUESTIONS_81 | VARCHAR |
| 64 | Type_E_Controls_Questions_82 | nvarchar(max) | Source only | TYPE_E_CONTROLS_QUESTIONS_82 | VARCHAR |
| 65 | Type_E_Controls_Questions_83 | nvarchar(max) | Source only | TYPE_E_CONTROLS_QUESTIONS_83 | VARCHAR |
| 66 | Type_E_Controls_Questions_84 | nvarchar(max) | Source only | TYPE_E_CONTROLS_QUESTIONS_84 | VARCHAR |
| 67 | Type_E_Controls_Questions_85 | nvarchar(max) | Source only | TYPE_E_CONTROLS_QUESTIONS_85 | VARCHAR |
| 68 | Type_E_Controls_Questions_86 | nvarchar(max) | Source only | TYPE_E_CONTROLS_QUESTIONS_86 | VARCHAR |
| 69 | Type_E_Controls_Questions_87 | nvarchar(max) | Source only | TYPE_E_CONTROLS_QUESTIONS_87 | VARCHAR |
| 70 | Type_E_Controls_Questions_88 | nvarchar(max) | Source only | TYPE_E_CONTROLS_QUESTIONS_88 | VARCHAR |
| 71 | Type_E_Controls_Questions_89 | nvarchar(max) | Source only | TYPE_E_CONTROLS_QUESTIONS_89 | VARCHAR |
| 72 | Type_E_Controls_Questions_810 | nvarchar(max) | Source only | TYPE_E_CONTROLS_QUESTIONS_810 | VARCHAR |
| 73 | Type_E_Controls_Questions_811 | nvarchar(max) | Source only | TYPE_E_CONTROLS_QUESTIONS_811 | VARCHAR |
| 74 | Type_E_Controls_Questions_812 | nvarchar(max) | Source only | TYPE_E_CONTROLS_QUESTIONS_812 | VARCHAR |
| 75 | Type_E_Controls_Questions_813 | nvarchar(max) | Source only | TYPE_E_CONTROLS_QUESTIONS_813 | VARCHAR |
| 76 | Type_F_Controls_Questions_91 | nvarchar(max) | Source only | TYPE_F_CONTROLS_QUESTIONS_91 | VARCHAR |
| 77 | Type_F_Controls_Questions_92 | nvarchar(max) | Source only | TYPE_F_CONTROLS_QUESTIONS_92 | VARCHAR |
| 78 | Type_F_Controls_Questions_93 | nvarchar(max) | Source only | TYPE_F_CONTROLS_QUESTIONS_93 | VARCHAR |
| 79 | System_Type_Decision_Tree_33 | nvarchar(max) | Source only | SYSTEM_TYPE_DECISION_TREE_33 | VARCHAR |
| 80 | Type_D_Controls_Questions_79 | nvarchar(max) | Source only | TYPE_D_CONTROLS_QUESTIONS_79 | VARCHAR |
| 81 | System_Type_Decision_Tree_33_Selection_d_Supporting_Information | nvarchar(max) | Source only | SYSTEM_TYPE_DECISION_TREE_33_SELECTION_D_SUPPORTING_INFORMATION | VARCHAR |
| 82 | System_Type_Decision_Tree_33_Selection_e_Supporting_Information | nvarchar(max) | Source only | SYSTEM_TYPE_DECISION_TREE_33_SELECTION_E_SUPPORTING_INFORMATION | VARCHAR |
| 83 | System_Type_Decision_Tree_38_Supporting_Information_RTX_Managed | nvarchar(max) | Source only | SYSTEM_TYPE_DECISION_TREE_38_SUPPORTING_INFORMATION_RTX_MANAGED | VARCHAR |
| 84 | System_Type_Decision_Tree_38_Supporting_Information_Third_Party_Manager | nvarchar(max) | Source only | SYSTEM_TYPE_DECISION_TREE_38_SUPPORTING_INFORMATION_THIRD_PARTY_MANAGER | VARCHAR |
| 85 | System_Type_Decision_Tree_38_Supporting_Information_Cloud_Service_Provider | nvarchar(max) | Source only | SYSTEM_TYPE_DECISION_TREE_38_SUPPORTING_INFORMATION_CLOUD_SERVICE_PROVIDER | VARCHAR |
| 86 | System_Type_Decision_Tree_38_Supporting_Information_Cloud_Type | nvarchar(max) | Source only | SYSTEM_TYPE_DECISION_TREE_38_SUPPORTING_INFORMATION_CLOUD_TYPE | VARCHAR |
| 87 | System_Type_Decision_Tree_38_Supporting_Information_SaaS_Provider__Third_Party_Name | nvarchar(max) | Source only; SQL double underscore maps to a single underscore in the Snowflake name. | SYSTEM_TYPE_DECISION_TREE_38_SUPPORTING_INFORMATION_SAAS_PROVIDER_THIRD_PARTY_NAME | VARCHAR |
| 88 | System_Type_Decision_Tree_38_Supporting_Information_Countries | nvarchar(max) | Source only | SYSTEM_TYPE_DECISION_TREE_38_SUPPORTING_INFORMATION_COUNTRIES | VARCHAR |
| 89 | HELPER_2nd_Level_Review_Required | nvarchar(max) | Source only | HELPER_2ND_LEVEL_REVIEW_REQUIRED | VARCHAR |
| 90 | First_Level_Reviewer_Comments | nvarchar(max) | Source only | FIRST_LEVEL_REVIEWER_COMMENTS | VARCHAR |
| 91 | Second_Level_Reviewer_Comments | nvarchar(max) | Source only | SECOND_LEVEL_REVIEWER_COMMENTS | VARCHAR |
| 92 | System_Type_Decision_Tree_34_Selection_c_Supporting_Information | nvarchar(max) | Source only | SYSTEM_TYPE_DECISION_TREE_34_SELECTION_C_SUPPORTING_INFORMATION | VARCHAR |
| 93 | Type_A_Controls_Questions_46_Selection_Other_describe_Supporting_Information | nvarchar(max) | Source only | TYPE_A_CONTROLS_QUESTIONS_46_SELECTION_OTHER_DESCRIBE_SUPPORTING_INFORMATION | VARCHAR |
| 94 | Type_B_Controls_Questions_56_Selection_Other_describe_Supporting_Information | nvarchar(max) | Source only | TYPE_B_CONTROLS_QUESTIONS_56_SELECTION_OTHER_DESCRIBE_SUPPORTING_INFORMATION | VARCHAR |
| 95 | Type_C_Controls_Questions_67_Selection_Other_describe_Supporting_Information | nvarchar(max) | Source only | TYPE_C_CONTROLS_QUESTIONS_67_SELECTION_OTHER_DESCRIBE_SUPPORTING_INFORMATION | VARCHAR |
| 96 | Type_D_Controls_Questions_76_Selection_Other_describe_Supporting_Information | nvarchar(max) | Source only | TYPE_D_CONTROLS_QUESTIONS_76_SELECTION_OTHER_DESCRIBE_SUPPORTING_INFORMATION | VARCHAR |
| 97 | Type_E_Controls_Questions_811_Selection_Other_describe_Supporting_Information | nvarchar(max) | Source only | TYPE_E_CONTROLS_QUESTIONS_811_SELECTION_OTHER_DESCRIBE_SUPPORTING_INFORMATION | VARCHAR |
| 98 | Type_F_Controls_Questions_93_Selection_Other_describe_Supporting_Information | nvarchar(max) | Source only | TYPE_F_CONTROLS_QUESTIONS_93_SELECTION_OTHER_DESCRIBE_SUPPORTING_INFORMATION | VARCHAR |
| 99 | Identifying_Information_11 | nvarchar(max) | Application_Name | IDENTIFYING_INFORMATION_11 | VARCHAR |
| 100 | Identifying_Information_14 | nvarchar(max) | Source only | IDENTIFYING_INFORMATION_14 | VARCHAR |
| 101 | Baseline_System_1 | nvarchar(max) | Source only | BASELINE_SYSTEM_1 | VARCHAR |
| 102 | Baseline_System_2 | nvarchar(max) | Source only | BASELINE_SYSTEM_2 | VARCHAR |
| 103 | Baseline_System_3 | nvarchar(max) | Source only | BASELINE_SYSTEM_3 | VARCHAR |
| 104 | Baseline_System_4 | nvarchar(max) | Source only | BASELINE_SYSTEM_4 | VARCHAR |
| 105 | Baseline_System_5 | nvarchar(max) | Source only | BASELINE_SYSTEM_5 | VARCHAR |
| 106 | Baseline_System_6 | nvarchar(max) | Source only | BASELINE_SYSTEM_6 | VARCHAR |
| 107 | Baseline_System_7 | nvarchar(max) | Source only | BASELINE_SYSTEM_7 | VARCHAR |
| 108 | Baseline_System_8 | nvarchar(max) | Source only | BASELINE_SYSTEM_8 | VARCHAR |
| 109 | Baseline_System_9 | nvarchar(max) | Source only | BASELINE_SYSTEM_9 | VARCHAR |
| 110 | Baseline_System_10 | nvarchar(max) | Source only | BASELINE_SYSTEM_10 | VARCHAR |
| 111 | Baseline_System_1_Supporting_Information_Cloud_Service_Provider | nvarchar(max) | Source only | BASELINE_SYSTEM_1_SUPPORTING_INFORMATION_CLOUD_SERVICE_PROVIDER | VARCHAR |
| 112 | Baseline_System_1_Supporting_Information_Cloud_Type | nvarchar(max) | Source only | BASELINE_SYSTEM_1_SUPPORTING_INFORMATION_CLOUD_TYPE | VARCHAR |
| 113 | Baseline_System_1_Supporting_Information_Countries | nvarchar(max) | Source only | BASELINE_SYSTEM_1_SUPPORTING_INFORMATION_COUNTRIES | VARCHAR |
| 114 | Baseline_System_1_Supporting_Information_RTX_Managed | nvarchar(max) | Source only | BASELINE_SYSTEM_1_SUPPORTING_INFORMATION_RTX_MANAGED | VARCHAR |
| 115 | Baseline_System_1_Supporting_Information_SaaS_Provider__Third_Party_Name | nvarchar(max) | Source only; SQL double underscore maps to a single underscore in the Snowflake name. | BASELINE_SYSTEM_1_SUPPORTING_INFORMATION_SAAS_PROVIDER_THIRD_PARTY_NAME | VARCHAR |
| 116 | Baseline_System_1_Supporting_Information_Third_Party_Manager | nvarchar(max) | Source only | BASELINE_SYSTEM_1_SUPPORTING_INFORMATION_THIRD_PARTY_MANAGER | VARCHAR |
| 117 | Baseline_System_3_Supporting_Information_Nature_or_Purpose | nvarchar(max) | Source only | BASELINE_SYSTEM_3_SUPPORTING_INFORMATION_NATURE_OR_PURPOSE | VARCHAR |
| 118 | Baseline_System_4_Supporting_Information_Countries | nvarchar(max) | Source only | BASELINE_SYSTEM_4_SUPPORTING_INFORMATION_COUNTRIES | VARCHAR |
| 119 | Baseline_System_5_Supporting_Information_Nature_or_Purpose | nvarchar(max) | Source only | BASELINE_SYSTEM_5_SUPPORTING_INFORMATION_NATURE_OR_PURPOSE | VARCHAR |
| 120 | Baseline_System_9_Supporting_Information_Access_Limitations | nvarchar(max) | Source only | BASELINE_SYSTEM_9_SUPPORTING_INFORMATION_ACCESS_LIMITATIONS | VARCHAR |
| 121 | HELPER_Baseline_System_Assessment | nvarchar(max) | Source only | HELPER_BASELINE_SYSTEM_ASSESSMENT | VARCHAR |
| 124 | First_Level_Reviewer_Denial_Justification | nvarchar(max) | Source only | FIRST_LEVEL_REVIEWER_DENIAL_JUSTIFICATION | VARCHAR |
| 125 | Second_Level_Reviewer_Denial_Justification | nvarchar(max) | Source only | SECOND_LEVEL_REVIEWER_DENIAL_JUSTIFICATION | VARCHAR |
| 126 | First_Level_Reviewer_Status | nvarchar(max) | Source only | FIRST_LEVEL_REVIEWER_STATUS | VARCHAR |
| 127 | Second_Level_Reviewer_Status | nvarchar(max) | Source only | SECOND_LEVEL_REVIEWER_STATUS | VARCHAR |
| 128 | Assessment_Type | nvarchar(max) | Assessment_Type | ASSESSMENT_TYPE | VARCHAR |
| 129 | Baseline_System_1_Supporting_Information_Customer | nvarchar(max) | Source only | BASELINE_SYSTEM_1_SUPPORTING_INFORMATION_CUSTOMER | VARCHAR |
| 130 | Baseline_System_1_Supporting_Information_Hosting_Location | nvarchar(max) | Source only | BASELINE_SYSTEM_1_SUPPORTING_INFORMATION_HOSTING_LOCATION | VARCHAR |
| 131 | Baseline_System_3_Supporting_Information_Description_of_Control | nvarchar(max) | Source only | BASELINE_SYSTEM_3_SUPPORTING_INFORMATION_DESCRIPTION_OF_CONTROL | VARCHAR |
| 132 | Baseline_System_5_Supporting_Information_Description_of_Control | nvarchar(max) | Source only | BASELINE_SYSTEM_5_SUPPORTING_INFORMATION_DESCRIPTION_OF_CONTROL | VARCHAR |
| 133 | System_Type_Decision_Tree_38_Supporting_Information_Description_of_Other | nvarchar(max) | Source only | SYSTEM_TYPE_DECISION_TREE_38_SUPPORTING_INFORMATION_DESCRIPTION_OF_OTHER | VARCHAR |
| 134 | Baseline_System_1_Supporting_Information_Description_of_Other | nvarchar(max) | Source only | BASELINE_SYSTEM_1_SUPPORTING_INFORMATION_DESCRIPTION_OF_OTHER | VARCHAR |
| 135 | System_Type_Decision_Tree_38_Supporting_Information_Customer_Name | nvarchar(max) | Source only | SYSTEM_TYPE_DECISION_TREE_38_SUPPORTING_INFORMATION_CUSTOMER_NAME | VARCHAR |
| 136 | System_Type_Decision_Tree_38_Supporting_Information_Hosting_Location | nvarchar(max) | Source only | SYSTEM_TYPE_DECISION_TREE_38_SUPPORTING_INFORMATION_HOSTING_LOCATION | VARCHAR |
| 137 | HELPER_Assessment_Required | nvarchar(max) | Source only | HELPER_ASSESSMENT_REQUIRED | VARCHAR |
| 138 | Identifying_Information_13 | nvarchar(max) | Source only | IDENTIFYING_INFORMATION_13 | VARCHAR |
| 139 | Second_Level_Reviewer_Collaborator | nvarchar(max) | Source only; raw reference string. Confirm legacy role format. | SECOND_LEVEL_REVIEWER_COLLABORATOR | VARCHAR |
| 140 | Respondent | nvarchar(max) | Source only; raw reference string. Confirm legacy role format. | RESPONDENT | VARCHAR |
| 141 | First_Level_Reviewer_Collaborator | nvarchar(max) | Source only; raw reference string. Confirm legacy role format. | FIRST_LEVEL_REVIEWER_COLLABORATOR | VARCHAR |
| 142 | Identifying_Information_12 | nvarchar(max) | Source only; raw reference string. Confirm legacy role format. | IDENTIFYING_INFORMATION_12 | VARCHAR |
| 143 | First_Level_Reviewer | nvarchar(max) | Source only; raw reference string. Confirm legacy role format. | FIRST_LEVEL_REVIEWER | VARCHAR |
| 144 | Second_Level_Reviewer | nvarchar(max) | Source only; raw reference string. Confirm legacy role format. | SECOND_LEVEL_REVIEWER | VARCHAR |
| 145 | Target_Contacts | nvarchar(max) | Source only | TARGET_CONTACTS | VARCHAR |
| 146 | Business | nvarchar(max) | Source only; resolved business-name candidate. Confirm legacy representation; raw BUSINESS is separate. | BUSINESS_NAME | VARCHAR |
| 147 | Authorization_Package | nvarchar(max) | Authorization_Package; resolved-name candidate. Validate multi-package lists, IS NOT NULL eligibility and package-name equality. | AUTHORIZATION_PACKAGE_NAME | VARCHAR |
| 148 | Identifying_Information_13_Collins_Supporting_Information | nvarchar(max) | Source only | IDENTIFYING_INFORMATION_13_COLLINS_SUPPORTING_INFORMATION | VARCHAR |
| 149 | HELPER_Assessment_Complete | nvarchar(max) | Source only | HELPER_ASSESSMENT_COMPLETE | VARCHAR |
| 150 | HELPER_Evidence_Required | nvarchar(max) | Source only | HELPER_EVIDENCE_REQUIRED | VARCHAR |
| 151 | HELPER_Background_Previous_Assessment_Information_Complete | nvarchar(max) | Source only | HELPER_BACKGROUND_PREVIOUS_ASSESSMENT_INFORMATION_COMPLETE | VARCHAR |
| 152 | HELPER_Evidence_Attached | nvarchar(max) | Source only | HELPER_EVIDENCE_ATTACHED | VARCHAR |
| 153 | HELPER_System_Type_Decision_Tree_Complete | nvarchar(max) | Source only | HELPER_SYSTEM_TYPE_DECISION_TREE_COMPLETE | VARCHAR |
| 154 | Identifying_Information_16 | nvarchar(max) | Source only | IDENTIFYING_INFORMATION_16 | VARCHAR |
| 155 | Identifying_Information_15 | nvarchar(max) | Source only | IDENTIFYING_INFORMATION_15 | VARCHAR |
| 156 | Second_Level_Reviewer_Collaborator_Name | nvarchar(max) | Source only; MISSING resolved collaborator name/UID. Raw collaborator references are not a substitute. | Not exposed | Not available |
| 157 | Second_Level_Reviewer_Collaborator_UID | nvarchar(max) | Source only; MISSING resolved collaborator name/UID. Raw collaborator references are not a substitute. | Not exposed | Not available |
| 158 | Respondent_Name | nvarchar(max) | Source only; aggregated matched employee names. Validate list behavior. | RESPONDENT_NAME_ONLY | VARCHAR |
| 159 | Respondent_UID | nvarchar(max) | Source only; aggregated matched UIDs. Unmatched tokens may be omitted. | RESPONDENT_ID_ONLY | VARCHAR |
| 160 | First_Level_Reviewer_Collaborator_Name | nvarchar(max) | Source only; MISSING resolved collaborator name/UID. Raw collaborator references are not a substitute. | Not exposed | Not available |
| 161 | First_Level_Reviewer_Collaborator_UID | nvarchar(max) | Source only; MISSING resolved collaborator name/UID. Raw collaborator references are not a substitute. | Not exposed | Not available |
| 162 | Identifying_Information_12_Name | nvarchar(max) | Source only; aggregated matched employee names. Validate list behavior. | IDENTIFYING_INFORMATION_12_NAME_ONLY | VARCHAR |
| 163 | Identifying_Information_12_UID | nvarchar(max) | Source only; aggregated matched UIDs. Unmatched tokens may be omitted. | IDENTIFYING_INFORMATION_12_ID_ONLY | VARCHAR |
| 164 | First_Level_Reviewer_Name | nvarchar(max) | Source only; aggregated matched employee names. Validate list behavior. | FIRST_LEVEL_REVIEWER_NAME_ONLY | VARCHAR |
| 165 | First_Level_Reviewer_UID | nvarchar(max) | Source only; aggregated matched UIDs. Unmatched tokens may be omitted. | FIRST_LEVEL_REVIEWER_ID_ONLY | VARCHAR |
| 166 | Second_Level_Reviewer_Name | nvarchar(max) | Source only; aggregated matched employee names. Validate list behavior. | SECOND_LEVEL_REVIEWER_NAME_ONLY | VARCHAR |
| 167 | Second_Level_Reviewer_UID | nvarchar(max) | Source only; aggregated matched UIDs. Unmatched tokens may be omitted. | SECOND_LEVEL_REVIEWER_ID_ONLY | VARCHAR |
| 168 | ConfirmedInArcher | datetime | ConfirmedInArcher; renamed timestamp. Confirm event meaning and datetime conversion. | ARCHER_CONTENT_GLOBAL_TRADE_ASSESSMENT_CONFIRMED_IN_ARCHER | TIMESTAMP_NTZ |
| 169 | Approval_Date | datetime | Approval_date; MISSING. Required by the current C2S projection. | Not exposed | Not available |
| 170 | First_Level_Reviewer_Approval_Comments | nvarchar(max) | Source only | FIRST_LEVEL_REVIEWER_APPROVAL_COMMENTS | VARCHAR |
| 171 | Second_Level_Reviewer_Approval_Comments | nvarchar(max) | Source only | SECOND_LEVEL_REVIEWER_APPROVAL_COMMENTS | VARCHAR |

## 9. Additional Snowflake outputs

These outputs are additional to the selected SQL mapping candidates, including the unconfirmed Global Trade FIRST_PUBLISHED candidate. They do not resolve missing SQL fields. Every output below is VARCHAR except the hardware first-published timestamp.

| Domain | Additional Snowflake output | Type | Purpose |
| --- | --- | --- | --- |
| Authorization package | BUSINESS | VARCHAR | Raw lookup references |
| Authorization package | ENTITY | VARCHAR | Raw lookup references |
| Authorization package | INFORMATION_SYSTEM_OWNER_ISO_ID_NAME | VARCHAR | Aggregated UID-before-name role helper |
| Authorization package | SYSTEM_ADMINISTRATOR_SA_ID_NAME | VARCHAR | Aggregated UID-before-name role helper |
| Authorization package | INFORMATION_SYSTEM_SECURITY_OFFICER_ISSO_ID_NAME | VARCHAR | Aggregated UID-before-name role helper |
| Authorization package | SECURITY_CONTROL_ASSESSOR_SCA_ID_NAME | VARCHAR | Aggregated UID-before-name role helper |
| Authorization package | AUTHORIZING_OFFICIAL_AO_ID_NAME | VARCHAR | Aggregated UID-before-name role helper |
| Authorization package | INFORMATION_SYSTEM_SECURITY_MANAGER_ISSM_ID_NAME | VARCHAR | Aggregated UID-before-name role helper |
| Authorization package | SENIOR_INFORMATION_SYSTEMS_SECURITY_OFFICER_SISSO_ID_NAME | VARCHAR | Aggregated UID-before-name role helper |
| Authorization package | INFORMATION_OWNER_IO_ID_NAME | VARCHAR | Aggregated UID-before-name role helper |
| Authorization package | ADDITIONAL_VISIBILITY_ID_NAME | VARCHAR | Aggregated UID-before-name role helper |
| Authorization package | ALTERNATE_SYSTEM_ADMINISTRATOR_SA_ID_NAME | VARCHAR | Aggregated UID-before-name role helper |
| Authorization package | ALTERNATE_INFORMATION_SYSTEM_SECURITY_OFFICER_ISSO_ID_NAME | VARCHAR | Aggregated UID-before-name role helper |
| Authorization package | ALTERNATE_SECURITY_CONTROL_ASSESSOR_SCA_ID_NAME | VARCHAR | Aggregated UID-before-name role helper |
| Authorization package | AUTHORIZING_OFFICIAL_DESIGNATED_REPRESENTATIVE_AODR_ID_NAME | VARCHAR | Aggregated UID-before-name role helper |
| Hardware | ARCHER_CONTENT_HARDWARE_FIRST_PUBLISHED | TIMESTAMP_NTZ | No corresponding SQL hardware source field |
| Hardware | SAP_ID | VARCHAR | Resolved package identifier list |
| Hardware | AUTHORIZATION_PACKAGE | VARCHAR | Raw package references |
| POAM | POAM_OWNER_ID_NAME | VARCHAR | Aggregated UID-before-name owner helper |
| POAM | AUTHORIZATION_PACKAGE | VARCHAR | Raw package references |
| POAM | SAP_ID | VARCHAR | Resolved package identifier list |
| Global Trade | AUTHORIZATION_PACKAGE | VARCHAR | Raw lookup references |
| Global Trade | SAP_ID | VARCHAR | Resolved package identifier list |
| Global Trade | BUSINESS | VARCHAR | Raw lookup references |
| Global Trade | FIRST_LEVEL_REVIEWER_ID_NAME | VARCHAR | Aggregated UID-before-name role helper |
| Global Trade | IDENTIFYING_INFORMATION_12_ID_NAME | VARCHAR | Aggregated UID-before-name role helper |
| Global Trade | IDENTIFYING_INFORMATION_13_RAYTHEON_SUPPORTING_INFORMATION | VARCHAR | No corresponding field in the SQL inventory |
| Global Trade | RESPONDENT_ID_NAME | VARCHAR | Aggregated UID-before-name role helper |
| Global Trade | SECOND_LEVEL_REVIEWER_ID_NAME | VARCHAR | Aggregated UID-before-name role helper |

## 10. Snowflake view behavior and dependencies

All curated objects in this section are in RTX_ENTERPRISESERVICES.ES_ESC_GRC_CURATED.

| Published view | Main curated source | Final join/filter behavior | Review implication |
| --- | --- | --- | --- |
| C2S_AUTHORIZATION_PACKAGE_ALL | ARCHER_CONTENT_AUTHORIZATION_PACKAGE | 16 LEFT JOINs; no WHERE | Retains base rows without lookup matches. Preserve the C2S operational-status rule and role semantics. |
| C2S_HARDWARE_ALL | ARCHER_CONTENT_HARDWARE | LEFT JOIN to aggregated package CTE; no WHERE | Missing package matches leave resolved fields NULL; hardware is retained. |
| C2S_POAM | ARCHER_CONTENT_POAMS | INNER JOIN to package CTE and owner CTE; no WHERE | No package match or no owner match excludes the base POAM row. |
| C2S_GLOBAL_TRADE | ARCHER_CONTENT_GLOBAL_TRADE_ASSESSMENT | Six LEFT JOINs; no WHERE or final DISTINCT | Missing enrichment alone does not remove the base row. Preserve the C2S package eligibility rule. |

Lookup processing splits semicolon-delimited references, trims tokens, joins to lookup keys and aggregates resolved values by source content ID. User-role lookups use ARCHER_WORKDAY_LDAP_USERS. Package, business and entity lookups use ARCHER_CONTENT_AUTHORIZATION_PACKAGE, ARCHER_CONTENT_BUSINESS and ARCHER_CONTENT_ENTITY as applicable. The authorization view also joins a POAM CTE from ARCHER_CONTENT_POAMS without exposing POAM outputs.

Inner joins inside lookup CTEs can omit unmatched tokens from enriched lists even when the final view retains the base row. The reviewed LISTAGG expressions do not specify WITHIN GROUP ordering. Do not assume fixed list order, name/UID positional alignment, or exact legacy string equivalence. Validate duplicates, NULLs, unmatched references and multi-value records.

The output declarations and SELECT ordering reviewed for these PROD views are aligned. Column alignment does not resolve the authorization Additional_Visibility source-expression issue. Base-source key uniqueness and data parity remain untested.

## 11. Existing SQL view paths

| Database and view | Active source | Projection / behavior |
| --- | --- | --- |
| ACP_Connection.dbo.ArcherMetricsAuthorizationPackageBaseTable | S2 | 28 direct fields; no filter or join |
| ACP_Connection.dbo.ArcherMetricsAuthorizationPackageBaseTableCopy | S1 | 37 fields including alternate-role details; no filter or join |
| ACP_Connection.dbo.ArcherMetricsHardwareBaseTable | S3 | 15 direct fields; no filter or join |
| ACP_Connection.dbo.ArcherMetricsHardwareBaseTableCMMC | S3 | Same 15-field projection; no CMMC filter |
| ACP_Connection.dbo.ArcherMetricsPOAMsBaseTable | S6 | 12 direct fields; no filter or join |
| C2S.dbo.ArcherMetricsAuthorizationPackageBaseTable | Same-named ACP view, ultimately S2 | 28 direct fields |
| C2S.dbo.ArcherMetricsHardwareBaseTable | Same-named ACP view, ultimately S3 | 15 source fields plus HostNameFirstPartHW |
| C2S.dbo.ArcherMetricsPOAMsBaseTable | Same-named ACP view, ultimately S6 | 12 direct fields |

All active BaseTable paths above use local ACP_Connection views. Their source tables remain distinct from C2S-generated dimension tables.

All main source views below are in C2S.dbo. Preserve these transformations at one agreed layer; applying them twice can change row counts and values.

| C2S source view | Source | Current behavior |
| --- | --- | --- |
| view_c2sreport_tbl_RIS_ArcherMetrics_AuthorizationPackage_PROD_LS | S1 | Retains rows where Operational_Status differs from Decommissioned or is NULL. Eight independent CROSS APPLY dbo.split_string calls expand role combinations. No active acronym filter. |
| view_c2sreport_tbl_RIS_ArcherMetrics_AuthorizationPackage_PROD_LS_AD | S1 | Same status filter. Six role fields use GetNamesFromRole_Scalar; AO and SCA remain direct. Retains eight role UID fields. The downstream name Unfiltered does not remove this status filter. |
| view_c2sreport_tbl_RIS_ArcherMetrics_Hardware_PROD_LS | S3 | 14 direct source fields plus Archer_Asset_Name from Host_Name. No WHERE, JOIN or DISTINCT in the view. |
| view_c2sreport_tbl_RIS_ArcherMetrics_POAMs_PROD_LS | S5 plus main authorization view | Nine selected POAM fields. INNER JOIN by Authorization_Package = Authorization_Package_Name. Overall_Status != Completed excludes Completed and NULL status. No active owner projection or owner CROSS APPLY. |
| view_c2sreport_tbl_GlobalTrade | S7 | 12 projected fields with documented aliases. WHERE Authorization_Package IS NOT NULL; no other filter, JOIN or DISTINCT present. |

The Global Trade IS NOT NULL filter does not exclude empty package strings or require a lookup match. POAM rows can multiply through the authorization join before the import applies DISTINCT to the selected fields. Multiple assessments per package are not automatically duplicate records.

## 12. SQL functions and generated outputs

| Function | Input / output | Existing behavior to preserve |
| --- | --- | --- |
| C2S.dbo.ParseStringValue | varchar(max) to varchar(max) | Returns everything before the first period. With no period returns the original input. No trim or case conversion. NULL remains NULL; a leading period produces an empty string. |
| C2S.dbo.GetNamesFromRole_Scalar | nvarchar(max) to nvarchar(max) | Built-in STRING_SPLIT on semicolon, TRIM each item, retain only items containing a pipe, take text before the first pipe, STRING_AGG with comma-space. No DISTINCT or ordering clause. Name order is not guaranteed; duplicate names can remain. No qualifying items produces NULL. |
| C2S.dbo.split_string | varchar(max) to table(value varchar(max)) | Custom loop, not built-in STRING_SPLIT. Splits semicolon-delimited input, trimming each emitted token. With no chosen delimiter, inserts the original value as one row, including NULL or empty input. Retains duplicates and can emit empty interior tokens; a trailing delimiter does not add a trailing empty token. |

The custom split_string definition tests semicolons in both WHEN conditions; its second branch assigns a period but is unreachable. Preserve the actual behavior unless a change is explicitly agreed. NULL/empty inputs produce one row, while multiple role values can create combinations across the eight independent CROSS APPLY calls. ParseStringValue and split_string use varchar even where source columns are nvarchar; validate Unicode conversion.

ImportDataIntoDimensionTables builds the following C2S.dbo objects. Each reviewed branch uses SELECT DISTINCT over its source view, creates an INT IDENTITY key and primary key, and exposes a dimension view.

| Generated C2S table | Source view suffix / name | Dimension view | Generated key |
| --- | --- | --- | --- |
| tbl_C2Sreport_RIS_ArcherMetrics_AuthorizationPackage_PROD | AuthorizationPackage_PROD_LS | dimAuthorizationPackage | SAPID |
| tbl_C2Sreport_RIS_ArcherMetrics_AuthorizationPackage_PROD_Unfiltered | AuthorizationPackage_PROD_LS_AD | dimAuthorizationPackageUnfiltered | AUID |
| tbl_C2Sreport_RIS_ArcherMetrics_Hardware_PROD | Hardware_PROD_LS | dimHardware | HWID |
| tbl_C2Sreport_RIS_ArcherMetrics_POAMs_PROD | POAMs_PROD_LS | dimPOAMs | POID |
| tbl_C2Sreport_GlobalTrade | view_c2sreport_tbl_GlobalTrade | dimGlobalTrade | GTID |

- SAPID, AUID, HWID, POID and GTID are generated model keys, not Archer business identifiers. Do not use them to match source records across platforms. DISTINCT across all selected columns does not establish business-key uniqueness.
- LastUpdatedDate is added as datetime2(3) and set from GETDATE(); it represents the C2S load, not the raw source update event.
- POAM_Actual_First_Published uses hRTN_First_Published unless NULL or empty, otherwise First_Published. DaysOpen uses DATEDIFF(DAY, that date, GETDATE()) at load time. The historical source field is text; invalid date strings require validation.
- CreateFactCyberEvents relates hardware to packages with a package-name LIKE predicate and POAM/Global Trade with package-name equality. The unfiltered package branch relates through hardware package text. Preserve the intended one-to-many relationships and selected generated IDs.
- Existing classification/connectivity points, cloud flags, role-email enrichment and other calculation procedures remain downstream C2S logic. Their full calculation-column definitions/types are outside this raw-source comparison and must not be inferred from source metadata.

## 13. Review completion criteria

The review can close when the following decisions and validations are recorded:

1. Resolve each missing field: provide an equivalent source, agree a derivation with matching meaning, or approve a scoped dependency change. Do not substitute arbitrary values or NULLs and classify them as equivalent.
2. Resolve the Additional_Visibility expression, SAP_ID_RTN meaning, _Compliant scale, hardware ID prefix, Global Trade creation date and role/package representations.
3. Confirm each ACP target table's required population and the layer responsible for filtering, splitting and enrichment. Resolve POAM owner-match eligibility and multi-package relationships.
4. Confirm Snowflake lengths, precision/scale, timestamp semantics and nullability; specify compatible SQL target conversions without truncation or unintended formatting changes.
5. Compare aligned refresh snapshots using validated source identifiers. Check key uniqueness at the intended grain, row counts, missing/extra records, NULL rates and field values. Then compare transformed C2S outputs separately, including role combinations, dates and package joins.
6. Record the agreed Matillion-to-ACP mappings and deployment/test plan after the above items are resolved.

This document is ready for source-mapping review. It does not assert that production migration testing has passed or that database/ETL changes have been approved.

