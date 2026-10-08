
Policy Level 3 validation: Archer → Snowflake → Enterprise Dashboard
Status: A missing policy association has been confirmed. The exact extraction or loading issue is still under investigation. Full Policy Level 3 coverage is not yet signed off.
Issue in plain language
The Finding exists in Snowflake. The policy exists in Snowflake. What we have not found in the tested Snowflake fields is the connection between them that Archer displays.
For example, Archer shows FND-3528 associated with DT-DM-001, even though the Finding’s Control Standards field is blank.  IMG_753B5D28-FF15-4318-8F0A-F11…
The exact example
Item	Value	What it means
Finding displayed to users	FND-3528	The business-facing Finding number
Finding Content ID	8391422	The internal identifier used to match this Finding between tables
Overall Status	Open	This Finding belongs in the Open population
Archer field label	Policy – Level 3	The field displaying the policy association
Archer Field ID	32061	Identifies the field—not the Finding or policy record
Policy displayed in Archer	DT-DM-001	The policy associated with this Finding
Policy master Content ID found for this code	7250316	The policy record identified in our Snowflake lookup


These identifiers were established through the Archer record, field metadata, and Snowflake record checks. The original extraction should still be checked for the actual referenced policy ID.  IMG_2FF6C4E5-00D0-4F29-B2CA-E62…  IMG_753B5D28-FF15-4318-8F0A-F11…  IMG_CB840D6A-3E80-4C6B-A703-F2E…
Exact Snowflake table and column to investigate
Database for the objects below: RTX_ENTERPRISESERVICES
Findings source table:
ES_ESC_GRC_CURATED.ARCHER_CONTENT_FINDINGS

Record identifier:
ARCHER_CONTENT_FINDINGS_CONTENT_ID = 8391422

Column corresponding to Archer field 32061:
POLICY_LEVEL_3

Observed value for FND-3528:
NULL

Policy master table:
ES_ESC_GRC_CURATED.ARCHER_CONTENT_POLICIES_POLICY_LEVEL_3

Record identifier:
ARCHER_CONTENT_POLICIES_POLICY_LEVEL_3_CONTENT_ID = 7250316

Policy reference:
POLICY_LEVEL_3_NAME = DT-DM-001

The policy record exists; the inspected Finding-side policy field is blank.  IMG_2FF6C4E5-00D0-4F29-B2CA-E62…  IMG_CB840D6A-3E80-4C6B-A703-F2E…
Important: do not confuse these fields
Archer Field ID	Archer field	Curated Snowflake column
32061	Policy – Level 3	POLICY_LEVEL_3
31713	Policies – Calculated Level 3	POLICIES_CALCULATED_LEVEL_3


These are separate fields, not two names for the same column. The field currently being traced is 32061 → POLICY_LEVEL_3.  IMG_2FF6C4E5-00D0-4F29-B2CA-E62…
How the existing views fit together
All three views are in ES_ESC_GRC_PUBLISHED.
View	Purpose and relevant finding
ENTERPRISE_FINDINGS	Shared Findings source used by Adrianne. It exposes Control Standard IDs and names, but those fields are blank for FND-3528.
C2S_ENTERPRISE_FINDINGS_DATA	Main dashboard Findings view, loaded as SF_FactFindings. It currently exposes the calculated source field as POLICIES_LEVEL_3, plural. It does not expose the separate source POLICY_LEVEL_3 field.
C2S_ENTERPRISE_FINDINGS_POLICY_DATA	Dashboard policy mapping, loaded as SF_FactFindingPolicy. It already reuses the shared ENTERPRISE_FINDINGS source and follows Control Standards to their parent policies.


The main view’s field omission and the mapping view’s Control Standard requirement are separate from the missing value in the curated source.  IMG_07DE942A-829A-404A-B384-2F4…  IMG_CB840D6A-3E80-4C6B-A703-F2E…   update_existing_findings_policy…
Current chart route:
Open Finding
    → recorded Control Standard ID
    → Control Standard’s recorded Policy Level 3 ID
    → policy master reference

That route can work where Control Standards are recorded. It cannot recover FND-3528’s displayed policy association when its Control Standard link is blank.   update_existing_findings_policy…
Latest population check
Among the 509 Open Findings tested, none had a nonblank source POLICY_LEVEL_3 value. This does not mean all 509 should have a policy in Archer; it means the tested Snowflake field currently supplies no direct-policy values for that population.  IMG_F89BB63D-111D-4437-8D51-529…
Adding this column to the main view now would expose blank values—it would not restore the missing associations.
Required next action
Trace Archer field 32061 for Finding Content ID 8391422 through the original extraction and into the curated column.
If the extracted record contains the policy reference but curated does not, investigate its parsing/loading. If the extracted record omits the field or value, investigate field selection, extracting-account access, and the response being loaded.
The precise cause has not been established. Do not attribute it to Matillion, Grady’s view, or Power BI without that comparison.
Current decision: Keep the existing views unchanged. Do not manually insert the missing association or claim complete policy coverage. Once the source relationship is available, validate the intended reporting rule and update the existing mapping as needed.
2. Plain-language message to Kamal / Grady
Hi Kamal / Grady, I’ve narrowed down the Policy Level 3 discrepancy with a specific example.
For FND-3528, Content ID 8391422, I opened the actual Archer record and confirmed that “Policy – Level 3” shows DT-DM-001. Its Control Standards field is blank.
In Snowflake, the same Finding exists in:
RTX_ENTERPRISESERVICES.ES_ESC_GRC_CURATED.ARCHER_CONTENT_FINDINGS
However, its POLICY_LEVEL_3 column is NULL. The metadata maps this column to Archer field ID 32061 — “Policy – Level 3.” This is separate from field 31713, the calculated-policy field.
The DT-DM-001 policy record also exists in the policy master, with Content ID 7250316. So we have both records, but we have not found their association in the Snowflake fields checked.
I also checked the current Open population: none of the 509 Open Findings had a populated POLICY_LEVEL_3 value. That does not mean every Finding needs a policy, but it confirms that this field is currently unavailable for direct-policy reporting.
Our current chart already reuses the shared ENTERPRISE_FINDINGS view and maps policies through Control Standards. That cannot recover this particular example because its Control Standard link is blank.
Could we trace field 32061 for record 8391422 in the original extracted data and compare it with the curated column? That should show whether the value is absent from the extraction or lost during loading. I’m keeping the current views unchanged until we identify the cause.
