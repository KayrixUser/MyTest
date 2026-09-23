
USE [ACP_Connection];

DECLARE @Today date = CONVERT(date, GETDATE());

SELECT
    '1 - Matillion source' AS Component,
    COUNT_BIG(*) AS Total_Rows,
    MAX([REPORTING_TIME]) AS Latest_Reporting_Time,
    MAX(TRY_CONVERT(datetime, [last_refreshed])) AS Latest_Last_Refreshed,
    CAST(NULL AS bigint) AS History_Rows_Loaded_Today
FROM [dbo].[C2S_SPLUNK_HOST_SUMMARY_DATA]

UNION ALL

SELECT
    '2 - History table',
    COUNT_BIG(*),
    MAX([reporting_time]),
    MAX(TRY_CONVERT(datetime, [last_refreshed])),
    COUNT_BIG(CASE WHEN CONVERT(date, TRY_CONVERT(datetime, [last_refreshed])) = @Today THEN 1 END)
FROM [dbo].[HISTORICAL_Splunk_Hosts_Raw]

UNION ALL

SELECT
    '3 - ACP history view',
    COUNT_BIG(*),
    MAX([reporting_time]),
    MAX(TRY_CONVERT(datetime, [last_refreshed])),
    COUNT_BIG(CASE WHEN CONVERT(date, TRY_CONVERT(datetime, [last_refreshed])) = @Today THEN 1 END)
FROM [dbo].[view_HISTORICAL_Splunk_Hosts_Raw]
ORDER BY Component;
