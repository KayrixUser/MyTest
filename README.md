
USE [C2S_Development];
GO
SET NOCOUNT ON;
SET XACT_ABORT ON;

IF DB_NAME() <> N'C2S_Development'
    THROW 51000, 'Run this test only in C2S_Development.', 1;
IF @@TRANCOUNT <> 0 OR (@@OPTIONS & 2) = 2
    THROW 51001, 'Use a new window with no open or implicit transaction.', 1;

DECLARE @Today date = CAST(GETDATE() AS date);

BEGIN TRY
    BEGIN TRANSACTION;

    -- Save and lock exactly one current Splunk monitoring row.
    SELECT CheckDate, RecordCount, LastUpdated
    INTO #SplunkMailOriginal
    FROM [C2S_Development].[dbo].[BaseViewCheckLogs] WITH (UPDLOCK, HOLDLOCK)
    WHERE CheckDate >= @Today
      AND CheckDate < DATEADD(DAY, 1, @Today)
      AND ViewGroup = 'Splunk'
      AND ViewName = 'Splunk_Historical_Splunk_Hosts_Raw';

    IF (SELECT COUNT(*) FROM #SplunkMailOriginal) <> 1
        THROW 51002, 'Expected exactly one Splunk log row for today. No test sent.', 1;

    IF EXISTS (
        SELECT 1 FROM #SplunkMailOriginal
        WHERE RecordCount IS NULL OR RecordCount <= 0
           OR LastUpdated IS NULL
           OR CAST(LastUpdated AS date) NOT IN (@Today, DATEADD(DAY, -1, @Today))
    )
        THROW 51003, 'The starting Splunk row is not healthy. No test sent.', 1;

    -- Simulate a missing date while retaining the real positive record count.
    UPDATE l
    SET LastUpdated = NULL
    FROM [C2S_Development].[dbo].[BaseViewCheckLogs] AS l
    INNER JOIN #SplunkMailOriginal AS b ON b.CheckDate = l.CheckDate
    WHERE l.ViewGroup = 'Splunk'
      AND l.ViewName = 'Splunk_Historical_Splunk_Hosts_Raw';

    IF @@ROWCOUNT <> 1
        THROW 51004, 'Unexpected update count; cancelling the test.', 1;

    -- The sender builds the issue email from the temporary state.
    -- Database Mail remains part of this transaction until COMMIT.
    EXEC [C2S_Development].[dbo].[sp_C2SSendFailureNotificationsBase];

    -- Restore the original value BEFORE committing the queued email.
    UPDATE l
    SET LastUpdated = b.LastUpdated
    FROM [C2S_Development].[dbo].[BaseViewCheckLogs] AS l
    INNER JOIN #SplunkMailOriginal AS b ON b.CheckDate = l.CheckDate
    WHERE l.ViewGroup = 'Splunk'
      AND l.ViewName = 'Splunk_Historical_Splunk_Hosts_Raw';

    IF @@ROWCOUNT <> 1
        THROW 51005, 'Restore count mismatch; rolling back the test and email.', 1;

    COMMIT TRANSACTION;

    SELECT l.RecordCount,
           b.LastUpdated AS OriginalLastUpdated,
           l.LastUpdated AS RestoredLastUpdated,
           CASE WHEN l.LastUpdated = b.LastUpdated THEN 'YES' ELSE 'NO' END AS Restored
    FROM [C2S_Development].[dbo].[BaseViewCheckLogs] AS l
    INNER JOIN #SplunkMailOriginal AS b ON b.CheckDate = l.CheckDate
    WHERE l.ViewGroup = 'Splunk'
      AND l.ViewName = 'Splunk_Historical_Splunk_Hosts_Raw';

    DROP TABLE #SplunkMailOriginal;
END TRY
BEGIN CATCH
    IF XACT_STATE() <> 0 ROLLBACK TRANSACTION;
    THROW;
END CATCH;
