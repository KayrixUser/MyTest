
What each donut category requests
Displayed category	Exact rule visible in your measure	Accurate interpretation
Legacy C2S	[IsC2S] = TRUE()	Packages identified by the C2S/ACPCCS acronym rule, without an expiration-date condition in this branch.
ATO Expired	[IsC2S] = FALSE(), expiration <> BLANK(), and expiration <= TODAY()	Packages without that acronym match whose recorded expiration meets the date cutoff.
No Expiration Date	[IsC2S] = FALSE() and expiration = BLANK()	Packages without that acronym match whose reporting expiration field compares equal to blank. This does not establish “No ATO issued.”
ATO Current	Calls [KPI SAP - Current ATO]	Its detailed conditions are inside another measure, which is not shown. 


1. SAP ATO Status — chart definition
Total SAPs: Counts each unique, nonblank SAP ID once within the selected reporting population and applicable filters. It is a package count—not a device count. 
Component	What it represents
C2S category	SAPs whose acronym contains C2S or ACPCCS. They are reported here regardless of their expiration date. This naming rule does not independently verify onboarding or authorization.
Pre ATO	SAPs not identified by the C2S acronym rule, with no expiration date recorded. This follows the legacy reporting category; it does not prove that an ATO has never been issued.  IMG_227893EF-2681-4EE3-9B82-CA8…
ATO Expired	SAPs not identified by the C2S acronym rule, with a populated expiration value at or before the calculation’s date cutoff.
ATO Current	SAPs not identified by the C2S acronym rule, with a populated expiration value after the calculation’s date cutoff. 


Date cutoff: The measures use TODAY()—the start of today. A date-only expiration of today is therefore classified as Expired.
Percentages: Each slice’s share of the displayed donut population.
Important: C2S-identified SAPs are excluded from the other three categories. Therefore, ATO Current does not represent every currently authorized SAP, including C2S packages.  IMG_07A571D8-7E25-49E5-9514-864…
2. Responses to the comments — copy and paste
“What is the difference between C2S and Pre ATO?”
C2S is identified from the package acronym containing C2S or ACPCCS. Pre ATO means the package does not match that naming rule and has no expiration date recorded. A missing expiration date alone does not identify a package as C2S.

“Does Pre ATO mean No ATO?”
Pre ATO is the legacy reporting label for this missing-date category. The calculation checks the expiration date, not whether authorization was issued, so we should not describe every record as having no ATO.

“Could C2S and ATO Current overlap?”
A package can be in C2S and have a current ATO in business terms. However, this chart deliberately reports C2S-identified packages separately and excludes them from Current, Expired, and Pre ATO. The category rules are designed not to overlap.

These responses reflect the supplied acronym rule and category measures.  IMG_C121AB27-90E6-41E8-88A5-FFB…  IMG_227893EF-2681-4EE3-9B82-CA8…  IMG_07A571D8-7E25-49E5-9514-864…
3. Info-page text — ready to paste
SAP ATO Status Information
Total SAPs: Unique SAP IDs within the selected report scope. Each SAP is counted once.
C2S: SAPs identified by an acronym containing C2S or ACPCCS, regardless of expiration date.
Pre ATO: SAPs outside that acronym-identified group with no expiration date recorded. This does not independently confirm that an ATO was never issued.
ATO Expired: SAPs outside the C2S-identified group with a populated ATO/IATO expiration value at or before today’s cutoff.
ATO Current: SAPs outside the C2S-identified group with a populated ATO/IATO expiration value after today’s cutoff.
Date cutoff: The calculation compares expiration values against the start of today.
Percentages: Each category’s share of the displayed donut.
Important: C2S-identified SAPs are reported separately and excluded from the three expiration-based categories. Package information reflects the data loaded into the report.


“What does Legacy C2S versus No Expiration Date mean?”
C2S is identified from the package acronym, not from a missing expiration date. A C2S-identified package with no expiration date stays in the C2S group. The No Expiration Date category covers only packages that do not match that acronym rule.

That is the distinction implemented by the measures. 
“Change Legacy C2S to C2S.”
Recommended label: “C2S identified.” Explain in the tooltip that this means an acronym containing C2S or ACPCCS. The formula does not check whether the package is old or “legacy,” so removing that word makes the label clearer without claiming verified onboarding. 
“Change No Expiration Date to No ATO.”
Keep “No Expiration Date.”
We cannot call this No ATO based on the current calculation. We are checking whether an expiration date is recorded, not whether authorization was issued.

Renaming the measure or label does not change what the formula establishes. 
“There will be overlap—for example, ATO Current could also be in C2S.”
That statement needs correcting for this chart.
A package may be in C2S and also have a current ATO in business terms. However, this chart reports C2S-identified packages separately and excludes them from Current, Expired, and No Expiration Date. The measures are not designed to count that package in both groups.

The Current ATO measure now explicitly confirms that exclusion. 
Recommended final legend:
C2S identified · ATO Expired · No Expiration Date · ATO Current
Recommended definition beneath the chart or on the Info page:
C2S is identified by the package acronym containing C2S or ACPCCS. All other packages are grouped by their recorded ATO/IATO expiration date. Missing expiration information does not necessarily mean no ATO was issued.


----------POAM

Included POA&Ms	Unique, nonblank POA&M IDs in the selected reporting population, excluding Completed and blank statuses.	73
Ongoing	POA&Ms whose recorded status is Ongoing. They may be within their deadline or already overdue.	39
Not Started	POA&Ms whose recorded status is Not Started. This does not mean their deadline is still in the future.	24
Awaiting Review	POA&Ms whose recorded status is Awaiting Review. The chart does not assume they are completed or on time.	9
Rejected	POA&Ms whose recorded status is Rejected. They are included because the current measure excludes only Completed and blank statuses. Inclusion does not mean they are approved or progressing.	1



“Does this include expired POA&Ms?”
Yes, within the selected Operational/Reauthorize SAP population, past-due POA&Ms remain included unless their status is Completed or blank. For example, a POA&M marked Ongoing stays under Ongoing after its completion date has passed. This chart shows its recorded status; it does not separately flag it as expired.

“We need to call out expired POA&Ms, particularly for CMMC.”
Understood. The current chart can include those records, but it does not show which ones are overdue. We should keep the status breakdown and agree with Linda on a separate overdue count and supporting details, including which deadline field to use. Overdue would be a subset of the existing POA&Ms, not an additional status slice in this donut.





POA&M Status Information
Purpose: Shows the recorded status of POA&Ms associated with SAPs marked Operational or Reauthorize, within the selected reporting population.
Included POA&Ms: Counts each unique, nonblank POA&M ID once. Completed and blank/empty statuses are excluded.
Ongoing: POA&Ms recorded as Ongoing. This does not indicate whether their completion deadline has passed.
Not Started: POA&Ms recorded as Not Started.
Awaiting Review: POA&Ms recorded as Awaiting Review.
Rejected: POA&Ms recorded as Rejected. These are included under the current reporting rule; inclusion does not indicate approval.
Percentages: Show each status category’s share of the POA&Ms displayed in the donut.
Past-due and missing dates: Completion dates are not evaluated by this chart. Past-due POA&Ms remain under their recorded status. A missing completion date does not itself exclude a record or establish that it is on time.
Data timing: Statuses reflect the data loaded into the report, not necessarily changes made in Archer after that load.




# M9 Weekly Automation — Demo Speaking Notes

## 1. Show the source scorecard

Hi everyone. I’ll quickly walk through how we capture the M9 data and preserve the weekly history.

This is the source scorecard that the business updates. Our agreed capture time is every Thursday at 1:00 PM Eastern, so the updated file needs to be completed and saved before that time.

The flow reads this workbook without changing the working source. The reporting date and week in the heading identify which period we are capturing.

## 2. Show the Power Automate flow

First, the flow copies the source workbook into the M9 Processing folder on our C2S SharePoint site. All our processing happens on this temporary copy.

It reads the six Business Unit records: Collins, PW, Raytheon, Corp, Unknown, and Total. It converts the supported missing-value entries, such as a dash or TBD, into blank/null values in the history data. Genuine zeros and valid numeric values are preserved; we are not recalculating the business’s scores.

The flow creates a separate M9_History tab with consistent column names. It also records the source reporting title and the UTC capture timestamp. The copied Scorecard tab retains the original information for reference.

## 3. Explain the archive and notification decisions

Next, the flow reads the reporting title and builds the archive filename using that report’s date and week—not today’s capture date.

It compares the source reporting date with the latest completed archive. When the source has a newer reporting date, the flow proceeds to save the completed workbook in the M9 Data folder.

An existing archive does not automatically mean an email is required. If we rerun a report already captured during the current weekly cycle, the flow skips the duplicate.

If the source still has the same reporting date and the previous final capture was before this Thursday’s noon cutoff, it sends the source-update notification and stops final publication. It also alerts if the source shows an older reporting date.

This check uses the previous final archive’s capture timestamp—not the new temporary copy’s timestamp. The email means a newer reporting period was not detected; it is not a general notification for every technical flow failure.

## 4. Show the completed archive

Here is the final workbook in C2S SharePoint. For example:

20261001_M9SurgeDetailReportWeek39_ESCyber.xlsx

The Scorecard tab keeps the captured source presentation, and M9_History contains the clean reporting data.

Each new reporting period gets a separate archive. Power BI combines the M9HistoryData table from these completed workbooks. Two weekly files with six records each give us twelve historical rows after refresh. Previous weeks are retained, not replaced.

## 5. Explain what must remain consistent in the source

The main requirement is a consistent spreadsheet structure.

Please keep the source filename and location, the Scorecard tab name, and the title in cell B2. The title should continue using the format “M9 Cloud YYYYMMDD Week NN.”

Our extraction currently uses B3:Q9: the headers and six Business Unit records. Column names, including their spaces and line breaks, also need to remain consistent.

Moving the title, renaming headers, adding rows or columns, or introducing unsupported text into numeric fields can cause errors or incomplete capture. Please coordinate those changes before updating the template.

Weekly values and the reporting date/week should change normally. Corrections to a period already archived need a controlled reprocess because we do not automatically overwrite its final file.

So, the overall purpose is simple: preserve each weekly scorecard, provide clean historical data, and notify the team when the source has not advanced to a newer reporting period.



Source change	What it affects	Expected risk
Rename or move M9 Cloud Scorecard.xlsx	Get file content using path	The configured path no longer identifies the source. Replacing the workbook at the same path/name is different from renaming it. Microsoft Learn
Rename or delete the Scorecard worksheet	Scorecard!B3:Q9 and the title formula	Table creation or the title reference fails because the expected worksheet is missing.
Move the reporting title away from B2	SourceReportTitle, filename generation, freshness comparison	Wrong title is read. This is already visible in the new layout.
Change the title format from M9 Cloud YYYYMMDD Week NN	Build M9 Archive Filename	Our current string parsing may fail or generate an incorrect name. A different date format, missing Week, or extra text after the week number needs a corresponding parsing change.
Rename column headers—even spaces, punctuation, or embedded line breaks	Add a row into a table → Row	The expression can fail with “property does not exist.” We already encountered this with M9.14. Connector responses can also encode punctuation in header names. Microsoft Learn
Insert/delete rows or columns, or add another BU outside the captured range	Fixed extraction range B3:Q9	The flow may omit records or columns, include unintended rows, or fail. It does not automatically expand to a changed source layout. Microsoft Learn
Introduce unsupported text in numeric fields	Normalization and numeric conversion	Exact "-" and uppercase "TBD" are handled. Values such as N/A, Pending, lowercase tbd, or text such as "91%" can fail the current json() conversion. Microsoft Learn
Change the numeric scale—for example, store 91.2 instead of 0.912	Archived metrics and Power BI	The flow may succeed but store an incorrect percentage scale. A green run does not validate the business meaning of numbers.
Add an existing table over the extraction range, or reuse M9_History, M9SnapshotData, or M9HistoryData in the source	The copied workbook’s worksheet/table creation	The flow can encounter overlapping-table or duplicate-name conflicts. Keep the source separate from our generated archive structure.
Restrict permissions, protect/encrypt the workbook, or apply access-restricting policies	Source read or processing of its copy	Connector access or workbook operations can fail. Do not change classifications merely to make the flow run.



Source change	Impact on our current implementation
Rename BU values, such as Collins → COL	Usually does not fail the flow. The changed labels are copied, but historical reporting needs consistent BU mapping.
Move or clear the title in B2	SourceReportTitle reads the wrong cell. Final naming and source-update detection can fail or give the wrong result.
Change the title pattern from M9 Cloud YYYYMMDD Week NN	Our filename expression relies on M9 Cloud, Week, and numeric date/week text. Different wording, separators, or missing values can break parsing. Microsoft Learn
Rename column headers, including actual spaces, capitalization, or manually inserted line breaks	The Row expression uses exact returned property names. Changes can cause “property does not exist” errors, as happened with M9.14. Connector responses can also encode punctuation in those names. Microsoft Learn
Move the scorecard or insert/delete rows or columns within it	Extraction is fixed to Scorecard!B3:Q9. New data outside that range can be omitted; shifted headers or values can cause errors or incomplete capture. The range does not automatically expand. Microsoft Learn
Rename/delete the Scorecard worksheet	The fixed table range and title reference no longer point to the required worksheet.
Introduce new nonnumeric values, such as N/A, Pending, lowercase tbd, or text percentages	Current normalization explicitly handles "-" and uppercase "TBD". Other strings—and empty strings returned by the connector—are not covered and can fail the numeric json() conversion. Microsoft Learn
Change numeric scale, such as storing 91.2 instead of 0.912 for 91.2%	The flow may succeed but save an incorrectly scaled metric. It copies/converts values; it does not independently validate their business meaning.
Rename or move M9 Cloud Scorecard.xlsx, or publish only a differently named file	The initial Get file content using path still reads the configured filename/path. It can fail or continue reading the old file. Microsoft Learn
Supply a workbook already containing M9_History, M9SnapshotData, or M9HistoryData	Our creation actions can encounter existing-name or overlapping-table conflicts. Do not use a completed archive as the new source template without reviewing this.
Change access, encryption, protection, or policy restrictions	Reading or processing the workbook can fail even when the layout is correct. Do not bypass those restrictions.
