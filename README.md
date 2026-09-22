
    -- Capture the original fact count before rebuilding.
    DECLARE @FactRowsBefore BIGINT;

    SELECT @FactRowsBefore = COUNT_BIG(*)
    FROM dbo.factCyberEvents;

    -- Rebuild DEV facts using the temporarily rebuilt dimSplunk.
    EXEC [C2S_Development].[dbo].[CreateFactCyberEvents];

    EXEC(N'
        SELECT ''Total fact rows'' AS CheckName,
               COUNT_BIG(*) AS Result
        FROM dbo.factCyberEvents

        UNION ALL

        SELECT ''Fact rows with SplunkID'', COUNT_BIG(*)
        FROM dbo.factCyberEvents
        WHERE SPLUNKID IS NOT NULL

        UNION ALL

        SELECT ''Orphan Splunk IDs'', COUNT_BIG(*)
        FROM dbo.factCyberEvents F
        WHERE F.SPLUNKID IS NOT NULL
          AND NOT EXISTS (
              SELECT 1
              FROM dbo.dimSplunk S
              WHERE S.SPLUNKID = F.SPLUNKID
          )

        UNION ALL

        SELECT ''Wrong hostname links'', COUNT_BIG(*)
        FROM dbo.factCyberEvents F
        JOIN dbo.dimSplunk S ON S.SPLUNKID = F.SPLUNKID
        LEFT JOIN dbo.dimHardware H ON H.HWID = F.HWID
        WHERE H.HWID IS NULL
           OR H.Archer_Asset_Name IS NULL
           OR S.Splunk_Hostname IS NULL
           OR S.Splunk_Hostname <> H.Archer_Asset_Name

        UNION ALL

        SELECT ''Missing expected Splunk links'', COUNT_BIG(*)
        FROM dbo.factCyberEvents F
        JOIN dbo.dimHardware H ON H.HWID = F.HWID
        WHERE F.SPLUNKID IS NULL
          AND EXISTS (
              SELECT 1
              FROM dbo.dimSplunk S
              WHERE S.Splunk_Hostname = H.Archer_Asset_Name
          );
    ');

    -- Restore BOTH the original Splunk dimension and facts.
    ROLLBACK TRANSACTION;

    SELECT
        N'Test rolled back' AS TestStatus,
        @@TRANCOUNT AS OpenTransactions,
        (SELECT COUNT_BIG(*) FROM dbo.dimSplunk)
            AS Restored_DimSplunk_Rows,
        @FactRowsBefore AS Fact_Rows_Before_Test,
        (SELECT COUNT_BIG(*) FROM dbo.factCyberEvents)
            AS Restored_Fact_Rows;
END TRY
BEGIN CATCH
    IF XACT_STATE() <> 0
        ROLLBACK TRANSACTION;
    THROW;
END CATCH;
