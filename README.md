
USE [C2S];
GO

DECLARE @PreviewTime datetime = GETDATE();
DECLARE @PreviewUtcTime datetime = GETUTCDATE();

;WITH SourceRows AS
(
    SELECT
        CAST(s.HOST_SEEN AS nvarchar(max)) COLLATE DATABASE_DEFAULT AS Hostname,
        s.REPORTING_TIME AS OriginalReportingTime,
        TRY_CONVERT(datetime, s.REPORTING_TIME) AS ReportingTime,
        s.last_refreshed AS OriginalLoadTime,
        TRY_CONVERT(datetime, s.last_refreshed, 120) AS SourceLoadTime,
        TRY_CONVERT(datetime, s.INSERT_DATE) AS SourceInsertTime
    FROM [ACP_Connection].[dbo].[C2S_SPLUNK_HOST_SUMMARY_DATA] AS s
),
SourceSummary AS
(
    SELECT
        COUNT_BIG(*) AS SourceRows,
        COUNT_BIG(CASE WHEN Hostname IS NULL OR LTRIM(RTRIM(Hostname)) = N''
                       THEN 1 END) AS MissingHostnames,
        COUNT_BIG(CASE WHEN OriginalReportingTime IS NULL THEN 1 END) AS NullReportingTimes,
        COUNT_BIG(CASE WHEN OriginalReportingTime IS NOT NULL AND ReportingTime IS NULL
                       THEN 1 END) AS InvalidReportingTimes,
        COUNT_BIG(CASE WHEN OriginalLoadTime IS NULL OR SourceLoadTime IS NULL
                       THEN 1 END) AS MissingOrInvalidLoadTimes,
        MIN(ReportingTime) AS EarliestReportingTime,
        MAX(ReportingTime) AS LatestReportingTime,
        MIN(SourceLoadTime) AS EarliestSourceLoadTime,
        MAX(SourceLoadTime) AS LatestSourceLoadTime,
        MAX(SourceInsertTime) AS LatestSourceInsertTime
    FROM SourceRows
),
DuplicateSourceHosts AS
(
    SELECT Hostname
    FROM SourceRows
    WHERE Hostname IS NOT NULL AND LTRIM(RTRIM(Hostname)) <> N''
    GROUP BY Hostname
    HAVING COUNT_BIG(*) > 1
),
DuplicateSummary AS
(
    SELECT COUNT_BIG(*) AS DuplicateHostGroups FROM DuplicateSourceHosts
),
HistoryRows AS
(
    SELECT
        h.hostname COLLATE DATABASE_DEFAULT AS Hostname,
        h.reporting_time AS ReportingTime,
        h.last_refreshed AS OriginalLoadTime,
        TRY_CONVERT(datetime, h.last_refreshed) AS HistoryLoadTime
    FROM [ACP_Connection].[dbo].[HISTORICAL_Splunk_Hosts_Raw] AS h
),
HistorySummary AS
(
    SELECT
        COUNT_BIG(*) AS ExistingHistoryRows,
        COUNT_BIG(CASE WHEN OriginalLoadTime IS NOT NULL AND HistoryLoadTime IS NULL
                       THEN 1 END) AS InvalidHistoryLoadTimes,
        COUNT_BIG(CASE WHEN OriginalLoadTime IS NULL THEN 1 END) AS NullHistoryLoadTimes,
        COUNT_BIG(CASE WHEN HistoryLoadTime < DATEADD(day, -5, @PreviewTime)
                       THEN 1 END) AS HistoryRowsToPurge,
        COUNT_BIG(CASE WHEN HistoryLoadTime >= DATEADD(day, -5, @PreviewTime)
                            OR OriginalLoadTime IS NULL THEN 1 END) AS HistoryRowsToRetain
    FROM HistoryRows
),
ProposedHistory AS
(
    -- Preserve old rows that the existing DELETE would not remove.
    SELECT Hostname, ReportingTime
    FROM HistoryRows
    WHERE HistoryLoadTime >= DATEADD(day, -5, @PreviewTime)
       OR OriginalLoadTime IS NULL

    UNION ALL

    -- Newly appended rows receive today's history load date, so survive purge.
    SELECT Hostname, ReportingTime FROM SourceRows
),
ProposedInput AS
(
    -- Same hostname match, per-host MAX and four-day rule as the PROD input view.
    SELECT p.Hostname AS Splunk_Hostname,
           CASE WHEN DATEDIFF(day, MAX(p.ReportingTime), @PreviewTime) >= 4
                THEN 1 ELSE 0 END AS Splunk_Reporting_Asset
    FROM ProposedHistory AS p
    INNER JOIN dbo.view_c2sreport_tbl_RIS_ArcherMetrics_Hardware_PROD_LS AS hw
        ON p.Hostname = dbo.ParseStringValue(hw.Host_Name)
    GROUP BY p.Hostname
),
CurrentInput AS
(
    SELECT Splunk_Hostname COLLATE DATABASE_DEFAULT AS Splunk_Hostname,
           Splunk_Reporting_Asset
    FROM dbo.view_c2sreport_tbl_raw_historical_splunk_hosts_raw
),
ComparisonSummary AS
(
    SELECT
        COUNT_BIG(c.Splunk_Hostname) AS CurrentMatchedHosts,
        COUNT_BIG(p.Splunk_Hostname) AS ProposedMatchedHosts,
        COUNT_BIG(CASE WHEN c.Splunk_Hostname IS NOT NULL AND p.Splunk_Hostname IS NULL
                       THEN 1 END) AS CurrentOnlyHosts,
        COUNT_BIG(CASE WHEN c.Splunk_Hostname IS NULL AND p.Splunk_Hostname IS NOT NULL
                       THEN 1 END) AS ProposedOnlyHosts,
        COUNT_BIG(CASE WHEN c.Splunk_Hostname IS NOT NULL AND p.Splunk_Hostname IS NOT NULL
                            AND c.Splunk_Reporting_Asset = p.Splunk_Reporting_Asset
                       THEN 1 END) AS CommonHostsSameFlag,
        COUNT_BIG(CASE WHEN c.Splunk_Hostname IS NOT NULL AND p.Splunk_Hostname IS NOT NULL
                            AND (c.Splunk_Reporting_Asset <> p.Splunk_Reporting_Asset
                                 OR c.Splunk_Reporting_Asset IS NULL)
                       THEN 1 END) AS CommonHostsChangedFlag,
        COUNT_BIG(CASE WHEN c.Splunk_Reporting_Asset = 0 THEN 1 END) AS CurrentFlag0,
        COUNT_BIG(CASE WHEN c.Splunk_Reporting_Asset = 1 THEN 1 END) AS CurrentFlag1,
        COUNT_BIG(CASE WHEN p.Splunk_Reporting_Asset = 0 THEN 1 END) AS ProposedFlag0,
        COUNT_BIG(CASE WHEN p.Splunk_Reporting_Asset = 1 THEN 1 END) AS ProposedFlag1
    FROM CurrentInput AS c
    FULL OUTER JOIN ProposedInput AS p ON p.Splunk_Hostname = c.Splunk_Hostname
)
SELECT v.CheckName, v.Result
FROM SourceSummary AS s
CROSS JOIN DuplicateSummary AS d
CROSS JOIN HistorySummary AS h
CROSS JOIN ComparisonSummary AS c
CROSS APPLY (VALUES
 (1,  N'Preview server', CONVERT(nvarchar(128), SERVERPROPERTY('ServerName'))),
 (2,  N'Preview database', CONVERT(nvarchar(128), DB_NAME())),
 (3,  N'Preview server-local time', CONVERT(nvarchar(128), @PreviewTime, 121)),
 (4,  N'Preview UTC time', CONVERT(nvarchar(128), @PreviewUtcTime, 121)),
 (5,  N'Production source rows', CONVERT(nvarchar(128), s.SourceRows)),
 (6,  N'Source NULL or blank hostnames', CONVERT(nvarchar(128), s.MissingHostnames)),
 (7,  N'Source NULL reporting times', CONVERT(nvarchar(128), s.NullReportingTimes)),
 (8,  N'Source invalid reporting-time conversions', CONVERT(nvarchar(128), s.InvalidReportingTimes)),
 (9,  N'Source duplicate host groups (C2S collation)', CONVERT(nvarchar(128), d.DuplicateHostGroups)),
 (10, N'Source NULL or invalid load times', CONVERT(nvarchar(128), s.MissingOrInvalidLoadTimes)),
 (11, N'Source earliest reporting time', COALESCE(CONVERT(nvarchar(128), s.EarliestReportingTime, 121), N'<NULL>')),
 (12, N'Source latest reporting time', COALESCE(CONVERT(nvarchar(128), s.LatestReportingTime, 121), N'<NULL>')),
 (13, N'Source earliest last_refreshed', COALESCE(CONVERT(nvarchar(128), s.EarliestSourceLoadTime, 121), N'<NULL>')),
 (14, N'Source latest last_refreshed', COALESCE(CONVERT(nvarchar(128), s.LatestSourceLoadTime, 121), N'<NULL>')),
 (15, N'Source latest INSERT_DATE', COALESCE(CONVERT(nvarchar(128), s.LatestSourceInsertTime, 121), N'<NULL>')),
 (16, N'Existing history rows', CONVERT(nvarchar(128), h.ExistingHistoryRows)),
 (17, N'History invalid non-NULL load dates', CONVERT(nvarchar(128), h.InvalidHistoryLoadTimes)),
 (18, N'History NULL load dates (existing purge retains these)', CONVERT(nvarchar(128), h.NullHistoryLoadTimes)),
 (19, N'History rows eligible for existing five-day purge', CONVERT(nvarchar(128), h.HistoryRowsToPurge)),
 (20, N'History rows retained before adding new source', CONVERT(nvarchar(128), h.HistoryRowsToRetain)),
 (21, N'Projected history rows after one new-source load', CONVERT(nvarchar(128), h.HistoryRowsToRetain + s.SourceRows)),
 (22, N'Current C2S input matched hosts', CONVERT(nvarchar(128), c.CurrentMatchedHosts)),
 (23, N'Proposed C2S input matched hosts', CONVERT(nvarchar(128), c.ProposedMatchedHosts)),
 (24, N'Current-only matched hosts after projected load/purge', CONVERT(nvarchar(128), c.CurrentOnlyHosts)),
 (25, N'New matched hosts after projected load/purge', CONVERT(nvarchar(128), c.ProposedOnlyHosts)),
 (26, N'Common matched hosts with unchanged flag', CONVERT(nvarchar(128), c.CommonHostsSameFlag)),
 (27, N'Common matched hosts with changed flag', CONVERT(nvarchar(128), c.CommonHostsChangedFlag)),
 (28, N'Current flag 0 hosts', CONVERT(nvarchar(128), c.CurrentFlag0)),
 (29, N'Current flag 1 hosts', CONVERT(nvarchar(128), c.CurrentFlag1)),
 (30, N'Proposed flag 0 hosts', CONVERT(nvarchar(128), c.ProposedFlag0)),
 (31, N'Proposed flag 1 hosts', CONVERT(nvarchar(128), c.ProposedFlag1))
) AS v(DisplayOrder, CheckName, Result)
ORDER BY v.DisplayOrder;
