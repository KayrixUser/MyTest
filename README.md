ATO Overview Dashboard — Metric Definitions & Data Sources
Purpose and Data Sources
The dashboard provides visibility into authorization-package status, ATO expiration dates, POA&M status and due dates, and system-administrator assignments.

Source: Archer authorization-package and POA&M data, provided through the C2S SQL Server reporting model.
Model table	Information used
dimAuthorizationPackage	Authorization-package name, business, entity, ATO dates, operational status, SA/ISSO assignments, information classification, control-set version, and recorded control count.
dimPOAMs	POA&M ID, overall status, target completion date, milestone estimated completion date, and days open.
factCyberEvents	Related SAP and POA&M records used by the report’s counting measures and filters.


Counting units: A SAP represents an authorization package—not a server. SAP KPI totals count distinct, nonblank SAP IDs; POA&M KPI totals count distinct, nonblank POA&M IDs.

Reporting scope: Results reflect applicable business, entity, SAP, SA, page, and visual filters. Report exclusions remove decommissioned, inactive, and stub entries according to the configured filters. The CMMC page also applies IncludeInISSO = True.

Data currency: Refer to Last Refreshed on the dashboard. Displayed counts are not fixed targets and can change after refresh.
1. SAP and ATO Metrics
These definitions apply to the SAP status chart and corresponding SA Info cards.  IMG_ADB547E8-0B24-48B9-8843-E44…  IMG_B53C4800-A98A-4534-BEBA-08E…
Metric / Visual	Definition and Calculation	Source Fields
Total SAPs	Distinct, nonblank SAP IDs within the applicable reporting scope.	factCyberEvents[SAP_ID]
In C2S	SAPs identified as C2S through the configured acronym mapping, including C2S and ACPCCS. C2S classification takes precedence over the expiration-date categories.	Authorization-package Acronym / IsC2S
ATO Current	Non-C2S SAPs with a recorded ATO expiration date after Today.	ATOIATO_Expiration_Date
ATO Expired	Non-C2S SAPs with a recorded ATO expiration date on or before Today.	ATOIATO_Expiration_Date
Pre ATO	Non-C2S SAPs with no ATO expiration date recorded. This is the report’s grouping label; a missing date alone does not confirm a particular workflow stage.	ATOIATO_Expiration_Date / ATO_STATUS
SAP ATO Status	Distribution of SAPs across C2S, Pre ATO, ATO Expired, and ATO Current. Percentages represent each category’s share of the displayed SAP total.	SAP count and ATO-status classification
ATOs Expiring Next 12 Months	SAPs grouped by ATO expiration month within the displayed 12-month window. The current visual also contains an Expired bucket, shown separately from the monthly schedule. Records without an expiration date cannot be assigned to an expiration month.	ATO expiration date and timeline grouping
ATOs Granted Past 12 Months	SAPs grouped by their recorded ATO date during the previous 12 completed calendar months. This shows recorded ATO dates, not necessarily every historical approval or renewal event.	ATOIATO_Date


2. POA&M Metrics — Quick Stats and SA Info
Operational scope: The reconciled Quick Stats POA&M visuals use packages with operational status Operational or Reauthorize. This excludes Under Development from those visuals.

Status distinction: Ongoing is one POA&M status. It is not the same as all non-completed POA&Ms. The Quick Stats status chart displays Ongoing, Not Started, Awaiting Review, and Rejected, while the SA Info card is specifically labeled Ongoing POA&Ms.  IMG_58CC5689-B5E2-48D0-B23B-476…  IMG_B53C4800-A98A-4534-BEBA-08E…
Metric / Visual	Definition and Calculation	Source Fields
POA&Ms Status / Total shown in the chart	Distinct POA&M IDs grouped by Ongoing, Not Started, Awaiting Review, and Rejected. Completed and blank statuses are excluded from this status view.	POAM_ID, Overall_Status
Ongoing POA&Ms — SA Info	Distinct POA&M IDs where Overall_Status = Ongoing, within the applicable operational and report scope.	POAM_ID, Overall_Status
Upcoming POA&M Expirations	Distinct POA&M IDs with a recorded Target Completion Date, grouped into the due-date windows below. The shared measure uses a nonblank status other than Completed; it is not restricted to Ongoing only.	POAM_Target_Completion_Date, Overall_Status
POA&Ms Due — monthly series	Eligible POA&Ms grouped by Target Completion Date within the monthly reporting window.	POAM_Target_Completion_Date
Milestones Due — monthly series	Eligible milestone-dated records grouped by Milestone Estimated Completion Date within the monthly reporting window. This date is separate from the POA&M target completion date.	Milestone_Estimated_Completion_Date
POA&M Expiration Detail Table	Displays the applicable SAP, POA&M ID, dates, and status for the selected expiration window. One SAP can have multiple POA&Ms; therefore, POA&M counts and distinct SAP counts can differ.	SAP and POA&M records


POA&M Due-Date Windows
All windows below use POA&M Target Completion Date, not the milestone date.
Window	Date Rule
1–30 Days	Today + 1 through Today + 30, inclusive.
31–60 Days	Today + 31 through Today + 60, inclusive.
61–90 Days	Today + 61 through Today + 90, inclusive.
90+ Days	Later than Today + 90.
Expired	Before Today.


Due today is neither expired nor in the 1–30-day window. Records with missing target dates are not assigned to these date windows.

POA&M versus milestone dates: A POA&M can have a future target completion date and an already-past milestone date. The two monthly series are therefore not expected to have matching counts.
3. SA Info
Metric / Visual	Definition	Source
SAP status cards	Uses the SAP and ATO definitions above, within the current selection.	SAP records and authorization-package status
Ongoing POA&Ms card	Counts POA&Ms with Overall_Status = Ongoing. It should not be interpreted as the total across all POA&M statuses.	POA&M records
SBU Alignment by SA	Shows SAP distribution by the recorded System Administrator and Entity.	Authorization-package SA and Entity fields
SAP Info table	Displays package details and associated ATO/POA&M information for the current selection.	Authorization-package and POA&M records


4. CMMC Page
The CMMC page provides package classifications, recorded control information, and POA&M workload views. These are reporting summaries—not a CMMC certification determination.  IMG_2A2D4DA7-8E76-4979-A227-4F6…
Metric / Visual	Definition and Calculation	Source Fields
SSP Count by CMMC Asset Category	SAP counts grouped by the category derived from the Authorization Package Name, using the report’s category-mapping rules. These are package counts, not physical asset counts.	Authorization_Package_Name, CMMC_Asset_Category
SSP Count by IS Type	SAP counts grouped by the IS type derived from the package-name mapping, such as INT, APP, NSG, or CLD.	Authorization_Package_Name, IS_Type
SSP Count by IS Subtype	SAP counts for the mapped subtypes LFA and CCP. Packages without either subtype are not represented in these two bars.	Package-name subtype mapping
Open POA&M by CMMC Asset Category	Distinct Ongoing POA&Ms grouped by their associated package’s CMMC asset category. Uses Operational or Reauthorize packages and ATO statuses C2S, Current, or Expired.	POAM_ID, Overall_Status, package category and statuses
POA&M Age Range	Ongoing POA&Ms grouped by Days Open, using the same operational and ATO-status scope as the category chart. Age describes how long the POA&M has been open—not how long until its due date.	dimPOAMs[DaysOpen], age grouping
POA&M Milestone Due Timeline by ISSO	POA&M counts grouped by ISSO and Milestone Estimated Completion Date. Uses the timeline’s Ongoing and Rejected status selection and Operational or Reauthorize package scope. Dates before Today appear as Overdue; blank dates are excluded. Future months follow the visual’s configured cutoff.	POAM_ID, ISSO, Overall_Status, milestone estimated completion date
Information Classification	Each evaluated record receives one group: CUI first; otherwise EXIM; otherwise Other. A record containing both CUI and EXIM is grouped as CUI, regardless of their order in the source text. The shared formula also places blank classifications in Other.	Information_Classification, InfoClass
800-171 Control Counts	SAPs grouped by their recorded number of controls, within the applicable 800-171 control-set and ATO-status filters. A category labeled 110 means the SAP’s recorded control count is 110—not that 110 controls passed.	Control_Set_Version_Number, Count_of_Controls
800-53 Control Counts	SAPs grouped by their recorded number of controls, within the applicable 800-53 control-set and ATO-status filters. This does not report controls passed or failed.	Control_Set_Version_Number, Count_of_Controls


Control-Mapping Statement
No approved metric-to-control mapping is assigned in this document. The dashboard summarizes recorded authorization, classification, control-count, and POA&M information. The SOX and IRAMP thresholds from other dashboards do not apply automatically to these metrics.
