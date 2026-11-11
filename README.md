# RTX C2S Archer SQL Server baseline and Snowflake comparison

Baseline date: November 11, 2026. SQL Server documentation is finalized for the reviewed scope, with the open items explicitly retained below. Snowflake object names, field mappings, types and data equivalence await Grady's views.

This document inventories the existing Archer source tables, SQL views and transformations needed to compare the current C2S inputs with Snowflake views consumed through Matillion. It covers all seven observed ACP Archer tables and the supplied C2S source paths. Source metadata and code establish structure and transformations; they do not establish current row counts, key uniqueness or the upstream loader.

Splunk, Tanium, Verdant, Wiz and LDAP ingestion are outside this review. Archer-sourced deviation fields remain included even when their names reference these products. Existing downstream calculations that consume other systems are retained as model dependencies, not new ingestion scope.

CYBERSQLPL01.UTCAPP.COM is decommissioned per Ranjit's confirmation. Its old FROM clauses in the three C2S BaseTable views are commented out. Investigation of this retired source is closed for this document. The user believes sp_C2S_FailureNotifications is inactive; that execution status is not independently verified and is recorded as a deferred operational item, not an ingestion requirement.

## Source inventory and Snowflake objects

Every SQL table below is in ACP_Connection.dbo. These are physical tables. The C2S-generated tables are listed separately. A PROD suffix does not establish that C2S generated an ACP table. Each Snowflake entry remains TBD until Grady supplies an actual object definition.

| ID | SQL Server table | Columns | Observed role | Snowflake database schema and view | Validation |
| --- | --- | --- | --- | --- | --- |
| S1 | ArcherMetrics_AuthorizationPackage_PROD_All | 68 | Main and AD authorization source views; ACP BaseTableCopy; baseline consumer | TBD | Pending |
| S2 | ArcherMetrics_AuthorizationPackage_PROD | 60 | ACP authorization BaseTable, then the same-named C2S wrapper | TBD | Pending |
| S3 | ArcherMetrics_Hardware_PROD_All | 17 | Main hardware source view; ACP hardware BaseTable and CMMC view; C2S wrapper | TBD | Pending |
| S4 | ArcherMetrics_Hardware_PROD | 17 | Physical table retained in inventory; active consumer not established in reviewed scope | TBD | Pending |
| S5 | ArcherMetrics_POAMs_PROD_All | 15 | Main POAM source view and dimension input | TBD | Pending |
| S6 | ArcherMetrics_POAMs_PROD | 15 | ACP POAM BaseTable, then the same-named C2S wrapper | TBD | Pending |
| S7 | ArcherMetrics_GlobalTrade_PROD | 169 | Global Trade source view; 12 columns projected to the dimension | TBD | Pending |

The four direct dimension-build inputs are S1, S3, S5 and S7. S2 and S6 remain in the wider SQL contract because the C2S BaseTable wrappers read them through ACP views. S4 remains inventoried, without a confirmed active consumer. This establishes four data domains, not a requirement for exactly seven Snowflake views. Separate views or approved derived mappings may cover multiple SQL tables, once their population differences are understood.

The current snapshot contains 361 table-column occurrences: 68 + 60 + 17 + 17 + 15 + 15 + 169. Global Trade's highest column_id is 171, but IDs 122 and 123 are absent from the supplied metadata. Column IDs are not column counts.

## SQL metadata conventions

All fields in the seven supplied table schemas are nullable. The displayed identity and computed flags are 0. Text columns show SQL_Latin1_General_CP1_CI_AS collation. This documents the observed metadata, not an assertion that business identifiers are unique or that every constraint/index has been reviewed.

nvarchar(max) and varchar(max) correspond to Max_Length_Bytes = -1. Hardware_PROD.Hardware_ID is varchar(255). numeric(18,0) has precision 18 and scale 0. A datetime row showing precision 23 and scale 3 remains SQL datetime; it is not datetime2(3). Source Last_Updated columns are text. The generated C2S LastUpdatedDate is a separate datetime2(3) load timestamp.

The paired tables below cover every field in both PROD and PROD_All without repeating identical rows. Absent means that the field is not in that table's captured schema. A common field name or type does not prove equal values or population. Snowflake column and type cells remain TBD, and every Snowflake row has validation status Pending.

## Authorization package field comparison

Left side: S1 PROD_All and S2 PROD. Right side: the corresponding field in Grady's Snowflake object, which is still TBD. Main and AD refer to the two C2S authorization source views; Base refers to the ACP and C2S BaseTable path. All rows are nullable.

| SQL field | S1 PROD_All type | S2 PROD type | SQL detail | Snowflake field | Snowflake type |
| --- | --- | --- | --- | --- | --- |
| Content_ID | nvarchar(max) | varchar(max) | Source identifier; PROD uses varchar and PROD_All uses nvarchar | TBD | TBD |
| Last_Updated | nvarchar(max) | nvarchar(max) | Base path: direct attribute | TBD | TBD |
| SAP_ID | nvarchar(max) | nvarchar(max) | Source field; not selected by main or AD view | TBD | TBD |
| Authorization_Package_Name | nvarchar(max) | nvarchar(max) | Main and AD: direct package attribute | TBD | TBD |
| Acronym | nvarchar(max) | nvarchar(max) | Main and AD: direct package attribute | TBD | TBD |
| Operational_Status | nvarchar(max) | nvarchar(max) | Main and AD: direct package attribute | TBD | TBD |
| Information_Classification | nvarchar(max) | nvarchar(max) | Main and AD: direct package attribute | TBD | TBD |
| ATOIATO_Date | datetime | datetime | Main and AD: direct package attribute | TBD | TBD |
| ATOIATO_Expiration_Date | datetime | datetime | Main and AD: direct package attribute | TBD | TBD |
| Connectivity | nvarchar(max) | nvarchar(max) | Main and AD: direct package attribute | TBD | TBD |
| Information_Owner_IO | nvarchar(max) | nvarchar(max) | Source field; not selected by main or AD view | TBD | TBD |
| Information_System_Owner_ISO | nvarchar(max) | nvarchar(max) | Main: split into role rows; AD: names aggregated | TBD | TBD |
| Senior_Information_Systems_Security_Officer_SISSO | nvarchar(max) | nvarchar(max) | Source field; not selected by main or AD view | TBD | TBD |
| Authorizing_Official_AO | nvarchar(max) | nvarchar(max) | Main: split into role rows; AD: direct value | TBD | TBD |
| Information_System_Security_Officer_ISSO | nvarchar(max) | nvarchar(max) | Main: split into role rows; AD: names aggregated | TBD | TBD |
| Security_Control_Assessor_SCA | nvarchar(max) | nvarchar(max) | Main: split into role rows; AD: direct value | TBD | TBD |
| Authorizing_Official_Designated_Representative_AODR | nvarchar(max) | nvarchar(max) | Source field; not selected by main or AD view | TBD | TBD |
| Package_Acronym | nvarchar(max) | nvarchar(max) | Base path: direct attribute | TBD | TBD |
| _Compliant | numeric(18,0) | numeric(18,0) | Source field; not selected by main or AD view | TBD | TBD |
| HRC_Related_Package_Acronyms | nvarchar(max) | nvarchar(max) | Base path: direct attribute | TBD | TBD |
| SAP_ID_RTN | nvarchar(max) | nvarchar(max) | Main and AD: direct package attribute; business identifier, uniqueness untested | TBD | TBD |
| Count_of_Controls_with_Open_POAMs | numeric(18,0) | numeric(18,0) | Source field; not selected by main or AD view | TBD | TBD |
| Additional_Visibility | nvarchar(max) | nvarchar(max) | Source field; not selected by main or AD view | TBD | TBD |
| Alternate_Information_System_Owner_ISO | nvarchar(max) | nvarchar(max) | Main: split into role rows; AD: names aggregated | TBD | TBD |
| Alternate_Security_Control_Assessor_SCA | nvarchar(max) | nvarchar(max) | Source field; not selected by main or AD view | TBD | TBD |
| Alternate_Information_System_Security_Officer_ISSO | nvarchar(max) | nvarchar(max) | Main: split into role rows; AD: names aggregated | TBD | TBD |
| Alternate_System_Administrator_SA | nvarchar(max) | nvarchar(max) | Main: split into role rows; AD: names aggregated | TBD | TBD |
| Information_System_Security_Manager_ISSM | nvarchar(max) | nvarchar(max) | Source field; not selected by main or AD view | TBD | TBD |
| System_Administrator_SA | nvarchar(max) | nvarchar(max) | Main: split into role rows; AD: names aggregated | TBD | TBD |
| Entity | nvarchar(max) | nvarchar(max) | Main and AD: direct package attribute | TBD | TBD |
| Business | nvarchar(max) | nvarchar(max) | Main and AD: direct package attribute | TBD | TBD |
| Information_Owner_IO_Name | nvarchar(max) | nvarchar(max) | Source field; not selected by main or AD view | TBD | TBD |
| Information_Owner_IO_UID | nvarchar(max) | nvarchar(max) | Source field; not selected by main or AD view | TBD | TBD |
| Information_System_Owner_ISO_Name | nvarchar(max) | nvarchar(max) | Base path: direct attribute | TBD | TBD |
| Information_System_Owner_ISO_UID | nvarchar(max) | nvarchar(max) | AD: retained role UID; selected roles support later email enrichment | TBD | TBD |
| Senior_Information_Systems_Security_Officer_SISSO_Name | nvarchar(max) | nvarchar(max) | Source field; not selected by main or AD view | TBD | TBD |
| Senior_Information_Systems_Security_Officer_SISSO_UID | nvarchar(max) | nvarchar(max) | Source field; not selected by main or AD view | TBD | TBD |
| Authorizing_Official_AO_Name | nvarchar(max) | nvarchar(max) | Base path: direct attribute | TBD | TBD |
| Authorizing_Official_AO_UID | nvarchar(max) | nvarchar(max) | AD: retained role UID; selected roles support later email enrichment | TBD | TBD |
| Information_System_Security_Officer_ISSO_Name | nvarchar(max) | nvarchar(max) | Base path: direct attribute | TBD | TBD |
| Information_System_Security_Officer_ISSO_UID | nvarchar(max) | nvarchar(max) | AD: retained role UID; selected roles support later email enrichment | TBD | TBD |
| Security_Control_Assessor_SCA_Name | nvarchar(max) | nvarchar(max) | Base path: direct attribute | TBD | TBD |
| Security_Control_Assessor_SCA_UID | nvarchar(max) | nvarchar(max) | AD: retained role UID; selected roles support later email enrichment | TBD | TBD |
| Authorizing_Official_Designated_Representative_AODR_Name | nvarchar(max) | nvarchar(max) | Source field; not selected by main or AD view | TBD | TBD |
| Authorizing_Official_Designated_Representative_AODR_UID | nvarchar(max) | nvarchar(max) | Source field; not selected by main or AD view | TBD | TBD |
| Additional_Visibility_Name | nvarchar(max) | nvarchar(max) | Source field; not selected by main or AD view | TBD | TBD |
| Additional_Visibility_UID | nvarchar(max) | nvarchar(max) | Source field; not selected by main or AD view | TBD | TBD |
| Alternate_Information_System_Owner_ISO_Name | nvarchar(max) | nvarchar(max) | Source field; not selected by main or AD view | TBD | TBD |
| Alternate_Information_System_Owner_ISO_UID | nvarchar(max) | nvarchar(max) | AD: retained role UID; selected roles support later email enrichment | TBD | TBD |
| Alternate_Security_Control_Assessor_SCA_Name | nvarchar(max) | nvarchar(max) | Source field; not selected by main or AD view | TBD | TBD |
| Alternate_Security_Control_Assessor_SCA_UID | nvarchar(max) | nvarchar(max) | Source field; not selected by main or AD view | TBD | TBD |
| Alternate_Information_System_Security_Officer_ISSO_Name | nvarchar(max) | nvarchar(max) | Source field; not selected by main or AD view | TBD | TBD |
| Alternate_Information_System_Security_Officer_ISSO_UID | nvarchar(max) | nvarchar(max) | AD: retained role UID; selected roles support later email enrichment | TBD | TBD |
| Alternate_System_Administrator_SA_Name | nvarchar(max) | nvarchar(max) | Source field; not selected by main or AD view | TBD | TBD |
| Alternate_System_Administrator_SA_UID | nvarchar(max) | nvarchar(max) | AD: retained role UID; selected roles support later email enrichment | TBD | TBD |
| Information_System_Security_Manager_ISSM_Name | nvarchar(max) | nvarchar(max) | Source field; not selected by main or AD view | TBD | TBD |
| Information_System_Security_Manager_ISSM_UID | nvarchar(max) | nvarchar(max) | Source field; not selected by main or AD view | TBD | TBD |
| System_Administrator_SA_Name | nvarchar(max) | nvarchar(max) | Base path: direct attribute | TBD | TBD |
| System_Administrator_SA_UID | nvarchar(max) | nvarchar(max) | AD: retained role UID; selected roles support later email enrichment | TBD | TBD |
| ConfirmedInArcher | datetime | datetime | Base path: direct attribute | TBD | TBD |
| Count_of_Controls | numeric(18,0) | Absent | Main and AD: direct package attribute | TBD | TBD |
| Baseline_Recommendation | nvarchar(max) | Absent | Main and AD: direct package attribute | TBD | TBD |
| Control_Set_Version_Number | nvarchar(max) | Absent | Main and AD: direct package attribute | TBD | TBD |
| Stub_Entry_RTN | nvarchar(max) | Absent | Main and AD: direct package attribute | TBD | TBD |
| SAP_Inactive | nvarchar(max) | Absent | Main and AD: direct package attribute | TBD | TBD |
| In_Continuous_Monitoring | nvarchar(max) | Absent | Main and AD: direct package attribute | TBD | TBD |
| Authorization_Package_Source | nvarchar(max) | Absent | Main and AD: direct package attribute | TBD | TBD |
| First_Published | datetime | Absent | Source date; PROD_All only | TBD | TBD |

The eight fields present only in S1 are Count_of_Controls, Baseline_Recommendation, Control_Set_Version_Number, Stub_Entry_RTN, SAP_Inactive, In_Continuous_Monitoring, Authorization_Package_Source and First_Published. The first seven are projected by the current main/AD source views. First_Published is retained in the raw source inventory.

## Hardware field comparison

Left side: S3 PROD_All and S4 PROD. Right side: Grady's corresponding Snowflake field. All rows are nullable. The main source view uses S3; the BaseTable and CMMC paths also ultimately use S3.

| SQL field | S3 PROD_All type | S4 PROD type | SQL detail | Snowflake field | Snowflake type |
| --- | --- | --- | --- | --- | --- |
| Content_ID | nvarchar(max) | varchar(max) | Raw source field; Unicode type differs | TBD | TBD |
| Last_Updated | nvarchar(max) | nvarchar(max) | BaseTable paths; not projected in main hardware source view | TBD | TBD |
| Hardware_Name | nvarchar(max) | nvarchar(max) | Main view and BaseTable paths: direct field | TBD | TBD |
| Host_Name | nvarchar(max) | nvarchar(max) | Main: original plus Archer_Asset_Name; wrapper: original plus HostNameFirstPartHW | TBD | TBD |
| Internal_IP_Address | nvarchar(max) | nvarchar(max) | Main view and BaseTable paths: direct field | TBD | TBD |
| MAC_Address | nvarchar(max) | nvarchar(max) | Main view and BaseTable paths: direct field | TBD | TBD |
| Hardware_ID | nvarchar(max) | varchar(255) | Source identifier; declared types differ between PROD and PROD_All | TBD | TBD |
| Authorization_Package | nvarchar(max) | nvarchar(max) | Main view and BaseTable paths: direct field | TBD | TBD |
| Splunk_Deviation_Approved | nvarchar(max) | nvarchar(max) | Main view and BaseTable paths: direct field | TBD | TBD |
| Tanium_Deviation_Approved | nvarchar(max) | nvarchar(max) | Main view and BaseTable paths: direct field | TBD | TBD |
| Access_Management_Deviation_Approved | nvarchar(max) | nvarchar(max) | Main view and BaseTable paths: direct field | TBD | TBD |
| Privileged_Access_Management_Deviation_Approved | nvarchar(max) | nvarchar(max) | Main view and BaseTable paths: direct field | TBD | TBD |
| Password_Management_Deviation_Approved | nvarchar(max) | nvarchar(max) | Main view and BaseTable paths: direct field | TBD | TBD |
| Session_Control_Deviation_Approved | nvarchar(max) | nvarchar(max) | Main view and BaseTable paths: direct field | TBD | TBD |
| ConfirmedInArcher | datetime | datetime | BaseTable paths; not projected in main hardware source view | TBD | TBD |
| Type | nvarchar(max) | nvarchar(max) | Main view and BaseTable paths: direct field | TBD | TBD |
| Wireless_Access_Point_Deviation_Approved | nvarchar(max) | nvarchar(max) | Main view only; not selected by the supplied BaseTable/CMMC views | TBD | TBD |

Archer_Asset_Name and HostNameFirstPartHW are derived SQL view outputs, not additional raw ACP fields. Both apply C2S.dbo.ParseStringValue to Host_Name and return varchar(max). The BaseTableCMMC definition contains no CMMC-specific filter.

## POAM field comparison

Left side: S5 PROD_All and S6 PROD. Right side: Grady's corresponding Snowflake field. All rows are nullable. The main POAM source view uses S5; the BaseTable wrapper path ultimately uses S6.

| SQL field | S5 PROD_All type | S6 PROD type | SQL detail | Snowflake field | Snowflake type |
| --- | --- | --- | --- | --- | --- |
| Content_ID | nvarchar(max) | varchar(max) | Raw source identifier; Unicode type differs | TBD | TBD |
| Last_Updated | nvarchar(max) | nvarchar(max) | BaseTable path; not projected by main POAM source view | TBD | TBD |
| POAM_ID | nvarchar(max) | nvarchar(max) | Main view and BaseTable path: direct field | TBD | TBD |
| First_Published | datetime | datetime | Main view and BaseTable path: direct field | TBD | TBD |
| Name | nvarchar(max) | nvarchar(max) | Main view and BaseTable path: direct field | TBD | TBD |
| Milestone_Estimated_Completion_Date | datetime | datetime | Main view and BaseTable path: direct field | TBD | TBD |
| Overall_Status | nvarchar(max) | nvarchar(max) | Main view and BaseTable path: direct field | TBD | TBD |
| POAM_Owner | nvarchar(max) | nvarchar(max) | BaseTable path; not projected by main POAM source view | TBD | TBD |
| Authorization_Package | nvarchar(max) | nvarchar(max) | Main view and BaseTable path: direct field | TBD | TBD |
| hRTN_First_Published | nvarchar(max) | nvarchar(max) | Text source; participates in generated first-published date and age | TBD | TBD |
| POAM_Owner_Name | nvarchar(max) | nvarchar(max) | BaseTable path; not projected by main POAM source view | TBD | TBD |
| POAM_Owner_UID | nvarchar(max) | nvarchar(max) | BaseTable path; not projected by main POAM source view | TBD | TBD |
| ConfirmedInArcher | datetime | datetime | BaseTable path; not projected by main POAM source view | TBD | TBD |
| POAM_Target_Completion_Date | datetime | datetime | Main view; not projected by the supplied POAM BaseTable views | TBD | TBD |
| Related_Allocated_Controls | nvarchar(max) | nvarchar(max) | Main view; not projected by the supplied POAM BaseTable views | TBD | TBD |

## Global Trade field comparison

Source S7 is ACP_Connection.dbo.ArcherMetrics_GlobalTrade_PROD. There is no GlobalTrade_PROD_All table in the supplied Archer inventory. This table lists every field present in the metadata, including fields beyond the 12-field C2S projection. All rows are nullable. The column ID is the metadata column_id, not a dense ordinal.

In the SQL detail column, an output name identifies one of the 12 current C2S view projections. Source only means that the field is present in S7 but is not selected by the supplied Global Trade source view. Those fields remain available for the wider source-equivalence comparison; they are not automatically required by that 12-field view.

| Column ID | SQL field | SQL type | C2S output or usage | Snowflake field | Snowflake type |
| --- | --- | --- | --- | --- | --- |
| 1 | Content_ID | nvarchar(max) | Source only | TBD | TBD |
| 2 | Last_Updated | nvarchar(max) | Last_Updated_GT | TBD | TBD |
| 3 | Created_Date | datetime | Created_Date_GT | TBD | TBD |
| 4 | Global_Trade_Assessment_ID | nvarchar(max) | Global_Trade_Assessment_ID | TBD | TBD |
| 5 | Respondent_Status | nvarchar(max) | Respondent_Status | TBD | TBD |
| 6 | Workflow_Current_Node | nvarchar(max) | Source only | TBD | TBD |
| 7 | Workflow_Process_Version | numeric(18,0) | Source only | TBD | TBD |
| 8 | Workflow_Job_Status | nvarchar(max) | Source only | TBD | TBD |
| 9 | Background_Information_and_Prior_Global_Trade_Reviews_23 | nvarchar(max) | Source only | TBD | TBD |
| 10 | Background_Information_and_Prior_Global_Trade_Reviews_24 | nvarchar(max) | Source only | TBD | TBD |
| 11 | Background_Information_and_Prior_Global_Trade_Reviews_25 | nvarchar(max) | Source only | TBD | TBD |
| 12 | Background_Information_and_Prior_Global_Trade_Reviews_26 | nvarchar(max) | Source only | TBD | TBD |
| 13 | Background_Information_and_Prior_Global_Trade_Reviews_21 | nvarchar(max) | Source only | TBD | TBD |
| 14 | Background_Information_and_Prior_Global_Trade_Reviews_22 | nvarchar(max) | Source only | TBD | TBD |
| 15 | System_Type_Decision_Tree_31 | nvarchar(max) | Technical_data | TBD | TBD |
| 16 | System_Type_Decision_Tree_32 | nvarchar(max) | Source only | TBD | TBD |
| 17 | System_Type_Decision_Tree_34 | nvarchar(max) | Source only | TBD | TBD |
| 18 | System_Type_Decision_Tree_35 | nvarchar(max) | Source only | TBD | TBD |
| 19 | System_Type_Decision_Tree_36 | nvarchar(max) | Source only | TBD | TBD |
| 20 | System_Type_Decision_Tree_37 | nvarchar(max) | Source only | TBD | TBD |
| 21 | System_Type_Decision_Tree_38 | nvarchar(max) | Data_location | TBD | TBD |
| 22 | System_Type_Decision_Tree_39 | nvarchar(max) | Information_Type | TBD | TBD |
| 23 | Type_A_Controls_Questions_41 | nvarchar(max) | Source only | TBD | TBD |
| 24 | Type_A_Controls_Questions_42 | nvarchar(max) | Source only | TBD | TBD |
| 25 | Type_A_Controls_Questions_43 | nvarchar(max) | Source only | TBD | TBD |
| 26 | Type_A_Controls_Questions_44 | nvarchar(max) | Source only | TBD | TBD |
| 27 | Type_A_Controls_Questions_45 | nvarchar(max) | Source only | TBD | TBD |
| 28 | Type_A_Controls_Questions_46 | nvarchar(max) | Source only | TBD | TBD |
| 29 | Type_A_Controls_Questions_47 | nvarchar(max) | Source only | TBD | TBD |
| 30 | Type_A_Controls_Questions_48 | nvarchar(max) | Source only | TBD | TBD |
| 31 | Type_B_Controls_Questions_51 | nvarchar(max) | Source only | TBD | TBD |
| 32 | Type_B_Controls_Questions_52 | nvarchar(max) | Source only | TBD | TBD |
| 33 | Type_B_Controls_Questions_53 | nvarchar(max) | Source only | TBD | TBD |
| 34 | Type_B_Controls_Questions_54 | nvarchar(max) | Source only | TBD | TBD |
| 35 | Type_B_Controls_Questions_55 | nvarchar(max) | Source only | TBD | TBD |
| 36 | Type_B_Controls_Questions_56 | nvarchar(max) | Source only | TBD | TBD |
| 37 | Type_B_Controls_Questions_57 | nvarchar(max) | Source only | TBD | TBD |
| 38 | Type_B_Controls_Questions_58 | nvarchar(max) | Source only | TBD | TBD |
| 39 | Type_C_Controls_Questions_61 | nvarchar(max) | Source only | TBD | TBD |
| 40 | Type_C_Controls_Questions_62 | nvarchar(max) | Source only | TBD | TBD |
| 41 | Type_C_Controls_Questions_63 | nvarchar(max) | Source only | TBD | TBD |
| 42 | Type_C_Controls_Questions_64 | nvarchar(max) | Source only | TBD | TBD |
| 43 | Type_C_Controls_Questions_65 | nvarchar(max) | Source only | TBD | TBD |
| 44 | Type_C_Controls_Questions_66 | nvarchar(max) | Source only | TBD | TBD |
| 45 | Type_C_Controls_Questions_67 | nvarchar(max) | Source only | TBD | TBD |
| 46 | Type_C_Controls_Questions_68 | nvarchar(max) | Source only | TBD | TBD |
| 47 | Type_C_Controls_Questions_69 | nvarchar(max) | Source only | TBD | TBD |
| 48 | Type_D_Controls_Questions_71 | nvarchar(max) | Source only | TBD | TBD |
| 49 | Type_D_Controls_Questions_72 | nvarchar(max) | Source only | TBD | TBD |
| 50 | Type_D_Controls_Questions_73 | nvarchar(max) | Source only | TBD | TBD |
| 51 | Type_D_Controls_Questions_74 | nvarchar(max) | Source only | TBD | TBD |
| 52 | Type_D_Controls_Questions_75 | nvarchar(max) | Source only | TBD | TBD |
| 53 | Type_D_Controls_Questions_76 | nvarchar(max) | Source only | TBD | TBD |
| 54 | Type_D_Controls_Questions_77 | nvarchar(max) | Source only | TBD | TBD |
| 55 | Type_D_Controls_Questions_78 | nvarchar(max) | Source only | TBD | TBD |
| 56 | Type_D_Controls_Questions_710 | nvarchar(max) | Source only | TBD | TBD |
| 57 | Type_D_Controls_Questions_711 | nvarchar(max) | Source only | TBD | TBD |
| 58 | Type_D_Controls_Questions_712 | nvarchar(max) | Source only | TBD | TBD |
| 59 | Type_D_Controls_Questions_713 | nvarchar(max) | Source only | TBD | TBD |
| 60 | Type_D_Controls_Questions_714 | nvarchar(max) | Source only | TBD | TBD |
| 61 | Type_D_Controls_Questions_715 | nvarchar(max) | Source only | TBD | TBD |
| 62 | Type_D_Controls_Questions_716 | nvarchar(max) | Source only | TBD | TBD |
| 63 | Type_E_Controls_Questions_81 | nvarchar(max) | Source only | TBD | TBD |
| 64 | Type_E_Controls_Questions_82 | nvarchar(max) | Source only | TBD | TBD |
| 65 | Type_E_Controls_Questions_83 | nvarchar(max) | Source only | TBD | TBD |
| 66 | Type_E_Controls_Questions_84 | nvarchar(max) | Source only | TBD | TBD |
| 67 | Type_E_Controls_Questions_85 | nvarchar(max) | Source only | TBD | TBD |
| 68 | Type_E_Controls_Questions_86 | nvarchar(max) | Source only | TBD | TBD |
| 69 | Type_E_Controls_Questions_87 | nvarchar(max) | Source only | TBD | TBD |
| 70 | Type_E_Controls_Questions_88 | nvarchar(max) | Source only | TBD | TBD |
| 71 | Type_E_Controls_Questions_89 | nvarchar(max) | Source only | TBD | TBD |
| 72 | Type_E_Controls_Questions_810 | nvarchar(max) | Source only | TBD | TBD |
| 73 | Type_E_Controls_Questions_811 | nvarchar(max) | Source only | TBD | TBD |
| 74 | Type_E_Controls_Questions_812 | nvarchar(max) | Source only | TBD | TBD |
| 75 | Type_E_Controls_Questions_813 | nvarchar(max) | Source only | TBD | TBD |
| 76 | Type_F_Controls_Questions_91 | nvarchar(max) | Source only | TBD | TBD |
| 77 | Type_F_Controls_Questions_92 | nvarchar(max) | Source only | TBD | TBD |
| 78 | Type_F_Controls_Questions_93 | nvarchar(max) | Source only | TBD | TBD |
| 79 | System_Type_Decision_Tree_33 | nvarchar(max) | Source only | TBD | TBD |
| 80 | Type_D_Controls_Questions_79 | nvarchar(max) | Source only | TBD | TBD |
| 81 | System_Type_Decision_Tree_33_Selection_d_Supporting_Information | nvarchar(max) | Source only | TBD | TBD |
| 82 | System_Type_Decision_Tree_33_Selection_e_Supporting_Information | nvarchar(max) | Source only | TBD | TBD |
| 83 | System_Type_Decision_Tree_38_Supporting_Information_RTX_Managed | nvarchar(max) | Source only | TBD | TBD |
| 84 | System_Type_Decision_Tree_38_Supporting_Information_Third_Party_Manager | nvarchar(max) | Source only | TBD | TBD |
| 85 | System_Type_Decision_Tree_38_Supporting_Information_Cloud_Service_Provider | nvarchar(max) | Source only | TBD | TBD |
| 86 | System_Type_Decision_Tree_38_Supporting_Information_Cloud_Type | nvarchar(max) | Source only | TBD | TBD |
| 87 | System_Type_Decision_Tree_38_Supporting_Information_SaaS_Provider__Third_Party_Name | nvarchar(max) | Source only | TBD | TBD |
| 88 | System_Type_Decision_Tree_38_Supporting_Information_Countries | nvarchar(max) | Source only | TBD | TBD |
| 89 | HELPER_2nd_Level_Review_Required | nvarchar(max) | Source only | TBD | TBD |
| 90 | First_Level_Reviewer_Comments | nvarchar(max) | Source only | TBD | TBD |
| 91 | Second_Level_Reviewer_Comments | nvarchar(max) | Source only | TBD | TBD |
| 92 | System_Type_Decision_Tree_34_Selection_c_Supporting_Information | nvarchar(max) | Source only | TBD | TBD |
| 93 | Type_A_Controls_Questions_46_Selection_Other_describe_Supporting_Information | nvarchar(max) | Source only | TBD | TBD |
| 94 | Type_B_Controls_Questions_56_Selection_Other_describe_Supporting_Information | nvarchar(max) | Source only | TBD | TBD |
| 95 | Type_C_Controls_Questions_67_Selection_Other_describe_Supporting_Information | nvarchar(max) | Source only | TBD | TBD |
| 96 | Type_D_Controls_Questions_76_Selection_Other_describe_Supporting_Information | nvarchar(max) | Source only | TBD | TBD |
| 97 | Type_E_Controls_Questions_811_Selection_Other_describe_Supporting_Information | nvarchar(max) | Source only | TBD | TBD |
| 98 | Type_F_Controls_Questions_93_Selection_Other_describe_Supporting_Information | nvarchar(max) | Source only | TBD | TBD |
| 99 | Identifying_Information_11 | nvarchar(max) | Application_Name | TBD | TBD |
| 100 | Identifying_Information_14 | nvarchar(max) | Source only | TBD | TBD |
| 101 | Baseline_System_1 | nvarchar(max) | Source only | TBD | TBD |
| 102 | Baseline_System_2 | nvarchar(max) | Source only | TBD | TBD |
| 103 | Baseline_System_3 | nvarchar(max) | Source only | TBD | TBD |
| 104 | Baseline_System_4 | nvarchar(max) | Source only | TBD | TBD |
| 105 | Baseline_System_5 | nvarchar(max) | Source only | TBD | TBD |
| 106 | Baseline_System_6 | nvarchar(max) | Source only | TBD | TBD |
| 107 | Baseline_System_7 | nvarchar(max) | Source only | TBD | TBD |
| 108 | Baseline_System_8 | nvarchar(max) | Source only | TBD | TBD |
| 109 | Baseline_System_9 | nvarchar(max) | Source only | TBD | TBD |
| 110 | Baseline_System_10 | nvarchar(max) | Source only | TBD | TBD |
| 111 | Baseline_System_1_Supporting_Information_Cloud_Service_Provider | nvarchar(max) | Source only | TBD | TBD |
| 112 | Baseline_System_1_Supporting_Information_Cloud_Type | nvarchar(max) | Source only | TBD | TBD |
| 113 | Baseline_System_1_Supporting_Information_Countries | nvarchar(max) | Source only | TBD | TBD |
| 114 | Baseline_System_1_Supporting_Information_RTX_Managed | nvarchar(max) | Source only | TBD | TBD |
| 115 | Baseline_System_1_Supporting_Information_SaaS_Provider__Third_Party_Name | nvarchar(max) | Source only | TBD | TBD |
| 116 | Baseline_System_1_Supporting_Information_Third_Party_Manager | nvarchar(max) | Source only | TBD | TBD |
| 117 | Baseline_System_3_Supporting_Information_Nature_or_Purpose | nvarchar(max) | Source only | TBD | TBD |
| 118 | Baseline_System_4_Supporting_Information_Countries | nvarchar(max) | Source only | TBD | TBD |
| 119 | Baseline_System_5_Supporting_Information_Nature_or_Purpose | nvarchar(max) | Source only | TBD | TBD |
| 120 | Baseline_System_9_Supporting_Information_Access_Limitations | nvarchar(max) | Source only | TBD | TBD |
| 121 | HELPER_Baseline_System_Assessment | nvarchar(max) | Source only | TBD | TBD |
| 124 | First_Level_Reviewer_Denial_Justification | nvarchar(max) | Source only | TBD | TBD |
| 125 | Second_Level_Reviewer_Denial_Justification | nvarchar(max) | Source only | TBD | TBD |
| 126 | First_Level_Reviewer_Status | nvarchar(max) | Source only | TBD | TBD |
| 127 | Second_Level_Reviewer_Status | nvarchar(max) | Source only | TBD | TBD |
| 128 | Assessment_Type | nvarchar(max) | Assessment_Type | TBD | TBD |
| 129 | Baseline_System_1_Supporting_Information_Customer | nvarchar(max) | Source only | TBD | TBD |
| 130 | Baseline_System_1_Supporting_Information_Hosting_Location | nvarchar(max) | Source only | TBD | TBD |
| 131 | Baseline_System_3_Supporting_Information_Description_of_Control | nvarchar(max) | Source only | TBD | TBD |
| 132 | Baseline_System_5_Supporting_Information_Description_of_Control | nvarchar(max) | Source only | TBD | TBD |
| 133 | System_Type_Decision_Tree_38_Supporting_Information_Description_of_Other | nvarchar(max) | Source only | TBD | TBD |
| 134 | Baseline_System_1_Supporting_Information_Description_of_Other | nvarchar(max) | Source only | TBD | TBD |
| 135 | System_Type_Decision_Tree_38_Supporting_Information_Customer_Name | nvarchar(max) | Source only | TBD | TBD |
| 136 | System_Type_Decision_Tree_38_Supporting_Information_Hosting_Location | nvarchar(max) | Source only | TBD | TBD |
| 137 | HELPER_Assessment_Required | nvarchar(max) | Source only | TBD | TBD |
| 138 | Identifying_Information_13 | nvarchar(max) | Source only | TBD | TBD |
| 139 | Second_Level_Reviewer_Collaborator | nvarchar(max) | Source only | TBD | TBD |
| 140 | Respondent | nvarchar(max) | Source only | TBD | TBD |
| 141 | First_Level_Reviewer_Collaborator | nvarchar(max) | Source only | TBD | TBD |
| 142 | Identifying_Information_12 | nvarchar(max) | Source only | TBD | TBD |
| 143 | First_Level_Reviewer | nvarchar(max) | Source only | TBD | TBD |
| 144 | Second_Level_Reviewer | nvarchar(max) | Source only | TBD | TBD |
| 145 | Target_Contacts | nvarchar(max) | Source only | TBD | TBD |
| 146 | Business | nvarchar(max) | Source only | TBD | TBD |
| 147 | Authorization_Package | nvarchar(max) | Authorization_Package | TBD | TBD |
| 148 | Identifying_Information_13_Collins_Supporting_Information | nvarchar(max) | Source only | TBD | TBD |
| 149 | HELPER_Assessment_Complete | nvarchar(max) | Source only | TBD | TBD |
| 150 | HELPER_Evidence_Required | nvarchar(max) | Source only | TBD | TBD |
| 151 | HELPER_Background_Previous_Assessment_Information_Complete | nvarchar(max) | Source only | TBD | TBD |
| 152 | HELPER_Evidence_Attached | nvarchar(max) | Source only | TBD | TBD |
| 153 | HELPER_System_Type_Decision_Tree_Complete | nvarchar(max) | Source only | TBD | TBD |
| 154 | Identifying_Information_16 | nvarchar(max) | Source only | TBD | TBD |
| 155 | Identifying_Information_15 | nvarchar(max) | Source only | TBD | TBD |
| 156 | Second_Level_Reviewer_Collaborator_Name | nvarchar(max) | Source only | TBD | TBD |
| 157 | Second_Level_Reviewer_Collaborator_UID | nvarchar(max) | Source only | TBD | TBD |
| 158 | Respondent_Name | nvarchar(max) | Source only | TBD | TBD |
| 159 | Respondent_UID | nvarchar(max) | Source only | TBD | TBD |
| 160 | First_Level_Reviewer_Collaborator_Name | nvarchar(max) | Source only | TBD | TBD |
| 161 | First_Level_Reviewer_Collaborator_UID | nvarchar(max) | Source only | TBD | TBD |
| 162 | Identifying_Information_12_Name | nvarchar(max) | Source only | TBD | TBD |
| 163 | Identifying_Information_12_UID | nvarchar(max) | Source only | TBD | TBD |
| 164 | First_Level_Reviewer_Name | nvarchar(max) | Source only | TBD | TBD |
| 165 | First_Level_Reviewer_UID | nvarchar(max) | Source only | TBD | TBD |
| 166 | Second_Level_Reviewer_Name | nvarchar(max) | Source only | TBD | TBD |
| 167 | Second_Level_Reviewer_UID | nvarchar(max) | Source only | TBD | TBD |
| 168 | ConfirmedInArcher | datetime | ConfirmedInArcher | TBD | TBD |
| 169 | Approval_Date | datetime | Approval_date | TBD | TBD |
| 170 | First_Level_Reviewer_Approval_Comments | nvarchar(max) | Source only | TBD | TBD |
| 171 | Second_Level_Reviewer_Approval_Comments | nvarchar(max) | Source only | TBD | TBD |

## Local SQL view paths

All active references below are local SQL Server database/schema/object references. These are existing SQL views, not evidence of Grady-published Snowflake views or of who originally created the SQL objects.

| Database and view | Active source | Projection or behavior |
| --- | --- | --- |
| ACP_Connection.dbo.ArcherMetricsAuthorizationPackageBaseTable | S2 | 28 direct fields; no filter or join |
| ACP_Connection.dbo.ArcherMetricsAuthorizationPackageBaseTableCopy | S1 | 37 fields including alternate-role details; no filter or join |
| ACP_Connection.dbo.ArcherMetricsHardwareBaseTable | S3 | 15 direct fields; no filter or join |
| ACP_Connection.dbo.ArcherMetricsHardwareBaseTableCMMC | S3 | Same 15-field projection; no CMMC filter |
| ACP_Connection.dbo.ArcherMetricsPOAMsBaseTable | S6 | 12 direct fields; no filter or join |
| C2S.dbo.ArcherMetricsAuthorizationPackageBaseTable | Same-named ACP view, ultimately S2 | 28 direct fields; old remote FROM commented |
| C2S.dbo.ArcherMetricsHardwareBaseTable | Same-named ACP view, ultimately S3 | 15 source fields plus HostNameFirstPartHW; old remote FROM commented |
| C2S.dbo.ArcherMetricsPOAMsBaseTable | Same-named ACP view, ultimately S6 | 12 direct fields; old remote FROM commented |

The C2S wrapper definitions now explain the linked-server text matches: CYBERSQLPL01.UTCAPP.COM appears in commented legacy FROM clauses in these three views. Their active FROM clauses select ACP views. Separate dependency rows previously reported for sp_C2S_FailureNotifications remain historical/operational evidence; they are not proof of an ACP loading process.

## Main source view behavior

All views in this section are in C2S.dbo. Their row structure must be compared at the same processing layer as the Snowflake equivalent. A source-level view should not apply C2S transformations a second time.

| View | Source | Current behavior |
| --- | --- | --- |
| view_c2sreport_tbl_RIS_ArcherMetrics_AuthorizationPackage_PROD_LS | S1 | Retains Operational_Status != Decommissioned OR NULL. Eight independent CROSS APPLY dbo.split_string calls expand role combinations. Acronym filters are commented. |
| view_c2sreport_tbl_RIS_ArcherMetrics_AuthorizationPackage_PROD_LS_AD | S1 | Same status filter. Six role fields use GetNamesFromRole_Scalar; AO and SCA remain direct. Retains eight role UID fields. The downstream name Unfiltered does not remove this status filter. |
| view_c2sreport_tbl_RIS_ArcherMetrics_Hardware_PROD_LS | S3 | 14 direct source fields plus Archer_Asset_Name from Host_Name. No WHERE, JOIN or DISTINCT in the supplied view. |
| view_c2sreport_tbl_RIS_ArcherMetrics_POAMs_PROD_LS | S5 plus main authorization view | Nine selected POAM fields. INNER JOIN by Authorization_Package = Authorization_Package_Name. Overall_Status != Completed excludes Completed and NULL status. Owner projection and owner CROSS APPLY are commented. |
| view_c2sreport_tbl_GlobalTrade | S7 | 12 projected fields with documented aliases. WHERE Authorization_Package IS NOT NULL; no other filter, JOIN or DISTINCT shown. |

Global Trade does not exclude empty package strings by its IS NOT NULL condition, and the shown view does not require a matching authorization-package record. Multiple assessments for one package are not automatically duplicates. POAM rows can multiply through the authorization join before the import's DISTINCT over selected fields.

## Custom function behavior

The function definitions have now been captured. The following describes code behavior; edge cases have not been run against SQL Server as part of this documentation work.

| Function | Input and output | Confirmed logic and migration detail |
| --- | --- | --- |
| C2S.dbo.ParseStringValue | varchar(max) to varchar(max) | Returns everything before the first period. With no period returns the original input. No trim or case conversion. NULL remains NULL; a leading period produces an empty string. |
| C2S.dbo.GetNamesFromRole_Scalar | nvarchar(max) to nvarchar(max) | Built-in STRING_SPLIT on semicolon, TRIM each item, retain only items containing a pipe, take text before the first pipe, STRING_AGG with comma-space. No DISTINCT or ordering clause. Name order is not guaranteed; duplicate names can remain. No qualifying items produces NULL. |
| C2S.dbo.split_string | varchar(max) to table(value varchar(max)) | Custom loop, not built-in STRING_SPLIT. Splits semicolon-delimited input, trimming each emitted token. With no chosen delimiter, inserts the original value as one row, including NULL or empty input. Retains duplicates and can emit empty interior tokens; a trailing delimiter does not add a trailing empty token. |

In the supplied split_string screenshot, both WHEN predicates test for a semicolon; the second THEN assigns a period. As written, the first matching branch takes precedence and the period branch is unreachable. This is recorded as existing behavior, not silently corrected. A migration should not introduce comma/period splitting on the assumption that the function name implies it.

NULL or empty role values in this custom split_string produce one row, so those values alone do not remove packages through the main view's CROSS APPLY calls. This resolves the earlier uncertainty about NULL role suppression. Multiple role values still produce combinations across the eight independent calls. For example, two ISO values and three ISSO values, with one row from every other role, produce six combinations before downstream DISTINCT.

ParseStringValue and custom split_string use varchar even though their source columns are nvarchar. Preserve or explicitly validate that conversion when comparing Unicode names/host values. GetNamesFromRole_Scalar uses nvarchar. Its whole-item trim does not explicitly trim the extracted name again before the pipe.

Function edge cases for later equivalence tests: NULL, empty input, whitespace-only input, a leading/trailing period, semicolon-separated roles, consecutive/trailing semicolons, pipe-free role items, duplicate names, and Unicode characters. These tests are pending data validation, not a reason to reopen source discovery.

Microsoft documentation supports the built-in behavior used in this interpretation: STRING_SPLIT output ordering is not guaranteed, STRING_AGG offers ordering through WITHIN GROUP, and LEN excludes trailing spaces. The custom loop's use of LEN therefore matters for its end condition.

- https://learn.microsoft.com/en-us/sql/t-sql/functions/string-split-transact-sql
- https://learn.microsoft.com/en-us/sql/t-sql/functions/string-agg-transact-sql
- https://learn.microsoft.com/en-us/sql/t-sql/functions/len-transact-sql

## C2S generated outputs

ImportDataIntoDimensionTables builds the following C2S tables from the main source views. They are different objects from the ACP tables listed above. Each displayed branch uses SELECT DISTINCT over the source view, assigns an INT IDENTITY key, adds its primary key, and exposes a dimension view.

| C2S.dbo generated table | Source view suffix or name | Dimension view | Generated key |
| --- | --- | --- | --- |
| tbl_C2Sreport_RIS_ArcherMetrics_AuthorizationPackage_PROD | AuthorizationPackage_PROD_LS | dimAuthorizationPackage | SAPID |
| tbl_C2Sreport_RIS_ArcherMetrics_AuthorizationPackage_PROD_Unfiltered | AuthorizationPackage_PROD_LS_AD | dimAuthorizationPackageUnfiltered | AUID |
| tbl_C2Sreport_RIS_ArcherMetrics_Hardware_PROD | Hardware_PROD_LS | dimHardware | HWID |
| tbl_C2Sreport_RIS_ArcherMetrics_POAMs_PROD | POAMs_PROD_LS | dimPOAMs | POID |
| tbl_C2Sreport_GlobalTrade | view_c2sreport_tbl_GlobalTrade | dimGlobalTrade | GTID |

SAPID, AUID, HWID, POID and GTID are generated model keys, not Archer source business IDs. They may be regenerated during rebuilds and must not be used as cross-platform source matching keys. SELECT DISTINCT over all projected fields does not enforce uniqueness of SAP_ID_RTN, Hardware_ID or POAM_ID.

The import adds LastUpdatedDate as datetime2(3) and updates it from GETDATE(). This represents the SQL load/update, separately from raw Last_Updated or ConfirmedInArcher.

POAM_Actual_First_Published is calculated as datetime from hRTN_First_Published unless that field is NULL or empty, in which case First_Published is used. DaysOpen is an int DATEDIFF(DAY, same date choice, GETDATE()) calculated at load time. hRTN_First_Published is text in the source; no TRY_CONVERT is shown in this calculation. Invalid source date text is a data-validation item.

Existing downstream package enrichment includes information-classification points, ordered connectivity points, cloud flags and role email enrichment. These belong to the C2S model, not the raw Archer input contract. Definitions for every added calculation column and their final output types have not all been supplied; they are retained as a separate downstream-model gap rather than guessed as source fields.

CreateFactCyberEvents joins hardware to authorization packages with a package-name LIKE predicate, POAMs and Global Trade by package-name equality, and the unfiltered package branch through hardware's package text. Its output selects generated IDs. These relationships and possible one-to-many combinations must remain distinct from raw source-grain comparisons.

## Validation status and remaining items

The document is complete as the current SQL source baseline. Open items below remain part of the handoff; this is not a declaration that production migration testing has passed.

| Item | Status | Treatment |
| --- | --- | --- |
| Seven ACP object types and field/type inventories | Captured | 68/60 authorization, 17/17 hardware, 15/15 POAM, 169 Global Trade fields. Nullable/type differences retained. |
| Five ACP views and three C2S wrappers | Captured | Active local sources distinguished from commented remote references. |
| Five main C2S source views and three functions | Captured | Filters, aliases, role expansion and host parsing documented. |
| Retired CYBERSQLPL01 source | Closed for this scope | Decommissioned per user confirmation; do not pursue further as an active source. |
| sp_C2S_FailureNotifications current execution | Deferred and unverified | User believes inactive. No claim of independently verified disablement and no change performed. |
| ACP table writers and upstream feeds | Unverified | Actual job/ETL mappings not established. No assumption that PROD derives from PROD_All or that C2S creates ACP PROD tables. |
| Hardware_PROD active consumer | Unverified | Retain table in inventory; do not mark it unused based on absence from captured views. |
| Row grain, uniqueness, counts and freshness | Pending validation | No current data samples or aligned count/value tests performed. |
| All downstream generated calculation-column types | Outside completed raw-source inventory | Do not invent types of fields added by unsupplied AddColumns procedures. |
| Snowflake mapping and Matillion implementation | Pending | Await exact Snowflake objects, metadata, population/filter rules and approved mapping to ACP inputs. |



