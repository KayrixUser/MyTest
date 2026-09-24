
his change brings the Snowflake/Matillion Splunk feed into the existing C2S historical-data process. It also checks the upstream source date so that copying old data into history does not make the source appear newly refreshed.
Current status: The ACP history procedure was updated and executed. The monitoring change was tested in DEV. Production monitoring deployment and the schedule change are still awaiting confirmation.
Source to C2S flow
1. Matillion loads the Snowflake Splunk host summary into ACP_Connection.dbo.C2S_SPLUNK_HOST_SUMMARY_DATA.
2. ACP_Connection.dbo.sp_UpdateHistoricalSplunkHosts appends that data to dbo.HISTORICAL_Splunk_Hosts_Raw in ACP_Connection.
3. The existing dbo.view_HISTORICAL_Splunk_Hosts_Raw and C2S reporting views continue supplying the model.
4. Existing C2S processing refreshes dimensions, facts, asset/SAP calculations and the data used by Power BI.
The history loader now reads the Matillion landing table instead of the legacy Splunk_Hosts_Raw table. The reviewed model/view path and calculation logic remain unchanged.
History procedure changes
The existing target columns are retained:
Historical column	Value loaded
hostname	Source HOST_SEEN
reporting_time	Source REPORTING_TIME, unchanged
log_type_compliant	Typed NULL (bit)
splunk_source	Typed NULL (nvarchar(9))
last_refreshed	Existing SQL history-load date at midnight


The two NULL columns are unavailable in the new feed and were confirmed unused in the reviewed C2S calculations. The original transaction handling and five-day purge condition remain in place. Retention is based on historical last_refreshed, not host reporting time. Each execution appends the source again, so repeated manual runs can add duplicate snapshots.
Monitoring change
In dbo.sp_InsertBaseViewCheckLogs, only the base Splunk block changes:
- Read ACP_Connection.dbo.C2S_SPLUNK_HOST_SUMMARY_DATA.
- Write its row count and latest source INSERT_DATE into dbo.BaseViewCheckLogs.
- Retain the monitoring label Splunk_Historical_Splunk_Hosts_Raw.
- Flag zero records, no usable timestamp, or a timestamp older than the existing 24-hour cutoff. Query errors continue through the existing CATCH logging.
The implementation interprets INSERT_DATE as UTC and converts it to Central time for comparison with the SQL Server clock. Confirm the upstream UTC convention before production sign-off.
The separate Dimensions / dimSplunk check continues using the model's LastUpdatedDate. No change is required there.
Date meanings and email behavior
REPORTING_TIME describes host reporting activity. Source INSERT_DATE provides upstream load freshness. Source last_refreshed records the Matillion load, while historical last_refreshed records the SQL history snapshot date. These dates serve different purposes.
Keep PROD dbo.sp_C2SSendFailureNotificationsBase and dbo.sp_C2SSendFailureNotificationsDimensions unchanged. The base sender reads the updated monitoring logs automatically.
For positive counts, the original Archer/Splunk email rules are:
- Today: normal date formatting.
- Yesterday: accepted, with bold date text and no red background.
- Before yesterday: an issue, with red background and bold date text.
Other source rules remain unchanged. The email uses calendar dates independently of the logger's 24-hour flag; it is not a strict 24-hour email alert. The original missing-date limitation is also retained: a positive count with NULL LastUpdated alone does not trigger an email issue.
Validation completed
- The history execution inserted 663,696 rows and deleted zero. History and its ACP view both returned 2,953,788 rows, with the same latest reporting time as the source.
- The DEV logger returned 661,695 source rows, a current timestamp and no logged Splunk outage. Source counts naturally differ from accumulated history counts.
- DEV email tests were delivered only to the tester. Today, yesterday, missing-date and older-date cases were exercised. The experimental sender's missing-date alert and changed yesterday bolding are excluded from PROD to preserve original behavior.
Production sequence
The agreed timing is Matillion at 2 AM Central, then the history job at 3 AM Central. Savannah needs to confirm the history schedule change. Confirm that the C2S model refresh follows history completion.
Promote the tested logger change into C2S, retain the original PROD email procedures and recipients, and verify the first scheduled load, monitoring row and email. Production completion should be recorded after that scheduled run succeeds.
