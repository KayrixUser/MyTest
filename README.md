
# Surge M9 Weekly Archive

This flow preserves the weekly M9 scorecard without changing the source workbook. It reads `M9 Cloud Scorecard.xlsx` from the original SharePoint site, creates a working copy on the C2S SharePoint site, and builds a clean history table. Missing `-` values become blank/null values, genuine zeros are preserved, and all six Business Unit records are retained: Collins, PW, Raytheon, Corp, Unknown, and Total.

Once processing is complete, the flow saves a separate final archive using the reporting date and week from the scorecard’s title. Each archive also records when the source was captured. Power BI combines the history tables from all completed archives, keeping previous weeks alongside new weeks rather than replacing historical data.

## Run timing and locations

| Item | Configuration |
|---|---|
| Current trigger | Manual testing |
| Agreed production schedule | Thursday at 12:00 noon U.S. Eastern, once Recurrence is enabled |
| Source | `RTXVMCampaignManagement-CORP` → Documents → Surge 2026 M9 Cloud → `M9 Cloud Scorecard.xlsx` |
| Processing destination | `ContinuousComplianceService_C2S-CORP` → Documents → General → CMS → Surge Metrics → **M9 Processing** |
| Final archive destination | Same C2S site → Documents → General → CMS → Surge Metrics → **M9 Data** |
| Final filename example | `20260925_M9SurgeDetailReportWeek38_ESCyber.xlsx` |

The source must be finalized and saved before the scheduled capture. The flow checks the version it reads at the start; later source updates do not change that run’s copy.

## Flow cheat sheet

| Step / action | What it does |
|---|---|
| **Manually trigger a flow** | Starts the test run. Recurrence will replace this for production scheduling. |
| **Get file content using path** | Reads the working source workbook without editing it. |
| **Create file** | Creates this run’s temporary workbook in **M9 Processing**. |
| **Create table** | Creates `M9SnapshotData` over the copied Scorecard’s headers and six records. |
| **List rows present in a table** | Reads the six copied scorecard records. |
| **Normalize M9 Missing Values** | Converts `-` values to actual nulls in the flow’s data. |
| **Create worksheet** | Adds the `M9_History` sheet to the temporary workbook. |
| **Create M9 History Table** | Creates `M9HistoryData` with consistent column names and capture metadata fields. |
| **Write M9 History Rows** | Processes the six normalized records sequentially. |
| **Add a row into a table** | Writes each clean record, its `CapturedAtUTC`, and `SourceReportTitle` into the history table. |
| **Read M9 Report Title** | Reads the reporting title from this run’s processed copy. |
| **Build M9 Archive Filename** | Uses the title’s reporting date and week to calculate the final filename. |
| **Get Latest M9 Archive** | Finds the final archive with the newest reporting date in **M9 Data**. |
| **Compare M9 Reporting Period** | Compares the current source’s reporting date with the latest archive’s reporting date. |
| **Previous M9 Archive Found** | Runs the previous-capture check only when an archived baseline exists. |
| **Read Previous M9 Capture** | Reads `CapturedAtUTC` from the previous final archive—not today’s temporary copy. |
| **M9 Cycle Start UTC** | Calculates the most recent Thursday-noon Eastern cutoff, converted to UTC. |
| **M9 Source Email Required** | Decides whether the source-update notification is needed. |
| **Condition 1** | Sends the notification only when the calculated result is true. |
| **Send an email (V2)** | Sends one source-update message per alerting run from the team mailbox. |
| **Stop after source-update alert** | Ends the run after the email, preventing final archive publication. |
| **Check M9 Archive Exists** | Checks whether the exact final filename already exists in **M9 Data**. |
| **Final archive Condition** | Continues publication only when that filename is absent. |
| **Delay** | Provides the configured seven-minute wait before reading the completed workbook for publication. |
| **Get Completed M9 Workbook** | Reads the processed temporary workbook, including its populated history table. |
| **Create Final M9 Archive** | Saves those completed contents in **M9 Data** under the reporting-date/week filename. |

## When the source-update email is sent

The decision uses the **source reporting date** and the **previous final archive’s capture timestamp**. It does not require the metric values to change.

| Scenario | Email | Final archive behavior |
|---|---|---|
| Source reporting date is newer than the latest archive | No | Continue normal archive creation and duplicate checks. |
| Same reporting date, and previous final capture was before the current Thursday-noon cutoff | Yes | Stop; do not publish another final archive. |
| Same reporting date, already captured during the current weekly cycle | No | Skip the existing final filename. |
| Source reporting date is older than the latest archive | Yes | Stop; flag the older source report. |
| No previous archive exists | No source-update email | Establish the initial archive; freshness cannot be verified against a missing baseline. |

**Example:** On Thursday after noon, the source still shows September 25 / Week 38, and the previous final archive was captured on September 30. The flow sends an email because that report has not advanced for the new capture cycle.

The new temporary workbook’s timestamp does not make an old report current. The comparison reads the timestamp already stored in the previous final archive.

## How Power BI uses the archives

Power BI’s **M9 Weekly History** query reads completed archive files from **M9 Data** and combines the named **M9HistoryData** table inside each workbook’s **M9_History** sheet. It excludes temporary processing files.

- One six-record archive gives **6 historical rows**.
- Two six-record archives give **12 historical rows**.
- Refreshing those same two files again still gives **12 rows**, not another duplicated set.

`ReportingDate`, derived from the filename, identifies each reporting period. `CapturedAtUTC` records the capture time. A Power BI refresh must occur after the final archive is available to include the new records.

## Important boundaries

The notification means **“a newer declared reporting period was not detected,”** not “the source system’s refresh job failed.” Identical weekly scores are acceptable. Corrections under an already-archived filename are not automatically published over the existing archive.

This email branch does not monitor a flow that never starts or fails before reaching the source-update check. Repeated runs while the source remains stale can send repeated emails. Temporary files remain in **M9 Processing** for troubleshooting.
