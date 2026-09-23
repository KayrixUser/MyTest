USE ;
GO

IF DB_NAME() <> N'C2S_Development'
    THROW 51000, 'This test must run only in C2S_Development.', 1;
IF @@TRANCOUNT <> 0
    THROW 51001, 'Use a fresh query window with no open transaction.', 1;

SET NOCOUNT ON;
SET XACT_ABORT ON;

IF OBJECT_ID(N'dbo.view_c2sreport_tbl_raw_historical_splunk_hosts_raw', N'V') IS NULL
   OR OBJECT_ID(N'dbo.dimSplunk', N'V') IS NULL
   OR OBJECT_ID(N'dbo.dimHardware', N'V') IS NULL
   OR OBJECT_ID(N'dbo.factCyberEvents', N'V') IS NULL
   OR OBJECT_ID(N'dbo.tbl_C2Sreport_raw_historical_splunk_hosts_raw', N'U') IS NULL
   OR OBJECT_ID(N'dbo.tbl_C2Sreport_factCyberEvents', N'U') IS NULL
   OR OBJECT_ID(N'dbo.tbl_C2Sreport_assetCalculationsServer', N'U') IS NULL
   OR OBJECT_ID(N'dbo.tbl_C2Sreport_assetCalculationsFRS', N'U') IS NULL
   OR OBJECT_ID(N'dbo.CreateFactCyberEvents', N'P') IS NULL
   OR OBJECT_ID(N'dbo.CreateAssetCalculationTables', N'P') IS NULL
    THROW 51002, 'A required existing DEV object was not found. No rebuild was started.', 1;

DECLARE @DimBefore bigint, @FactBefore bigint,
        @ServerBefore bigint, @FRSBefore bigint,
        @DimAfter bigint, @FactAfter bigint,
        @ServerAfter bigint, @FRSAfter bigint,
        @SourceRows bigint, @FailedChecks bigint,
        @Stage nvarchar(120) = N'Capture baseline';

DECLARE @Checks TABLE
(
    CheckName nvarchar(120) NOT NULL,
    Result bigint NOT NULL
);

BEGIN TRY
    BEGIN TRANSACTION;

    SELECT @DimBefore = COUNT_BIG(*) FROM dbo.dimSplunk;
    SELECT @FactBefore = COUNT_BIG(*) FROM dbo.factCyberEvents;
    SELECT @ServerBefore = COUNT_BIG(*) FROM dbo.tbl_C2Sreport_assetCalculationsServer;
    SELECT @FRSBefore = COUNT_BIG(*) FROM dbo.tbl_C2Sreport_assetCalculationsFRS;
    SELECT @SourceRows = COUNT_BIG(*)
    FROM dbo.view_c2sreport_tbl_raw_historical_splunk_hosts_raw;

    IF @SourceRows = 0
        THROW 51003, 'The DEV Splunk source view is empty. Test stopped.', 1;

    SET @Stage = N'Temporary Splunk dimension rebuild';

    -- Separate dynamic batches avoid stale compilation around DROP/CREATE.
    EXEC sys.sp_executesql N'DROP VIEW dbo.dimSplunk;';
    EXEC sys.sp_executesql N'DROP TABLE dbo.tbl_C2Sreport_raw_historical_splunk_hosts_raw;';

    EXEC sys.sp_executesql N'
        SELECT IDENTITY(int, 1, 1) AS SPLUNKID, S.*
        INTO dbo.tbl_C2Sreport_raw_historical_splunk_hosts_raw
        FROM
        (
            SELECT DISTINCT *
            FROM dbo.view_c2sreport_tbl_raw_historical_splunk_hosts_raw
        ) AS S;';

    EXEC sys.sp_executesql N'
        ALTER TABLE dbo.tbl_C2Sreport_raw_historical_splunk_hosts_raw
            ADD CONSTRAINT PK_SPLUNKID PRIMARY KEY (SPLUNKID);
        ALTER TABLE dbo.tbl_C2Sreport_raw_historical_splunk_hosts_raw
            ADD LastUpdatedDate datetime2(3) DEFAULT GETDATE();';

    EXEC sys.sp_executesql N'
        UPDATE dbo.tbl_C2Sreport_raw_historical_splunk_hosts_raw
        SET LastUpdatedDate = GETDATE();';

    EXEC sys.sp_executesql N'
        CREATE VIEW dbo.dimSplunk AS
        SELECT * FROM dbo.tbl_C2Sreport_raw_historical_splunk_hosts_raw;';

    SET @Stage = N'Temporary fact rebuild';
    EXEC [C2S_Development].[dbo].[CreateFactCyberEvents];

    SET @Stage = N'Temporary asset-calculation rebuild';
    EXEC [C2S_Development].[dbo].[CreateAssetCalculationTables];

    SET @Stage = N'Validate Splunk asset results';

    -- Compare distinct (asset group, HWID, flag) pairs. Server-table rows
    -- can differ in other security metrics, so raw row counts alone are
    -- not a sufficient test of the Splunk result.
    INSERT INTO @Checks (CheckName, Result)
    EXEC sys.sp_executesql N'
        WITH Expected AS
        (
            SELECT DISTINCT
                CASE WHEN H.[type] IN (''Linux Server'', ''Unix Server'', ''Windows Server'')
                     THEN ''Server'' ELSE ''FRS'' END AS AssetGroup,
                F.HWID,
                CASE WHEN H.Splunk_Deviation_Approved = ''Yes'' THEN 4
                     WHEN S.Splunk_Reporting_Asset IS NULL THEN 2
                     ELSE S.Splunk_Reporting_Asset END AS Flag
            FROM dbo.factCyberEvents AS F
            INNER JOIN dbo.dimHardware AS H ON H.HWID = F.HWID
            LEFT JOIN dbo.dimSplunk AS S ON S.SPLUNKID = F.SPLUNKID
            WHERE H.[type] IN (''Linux Server'', ''Unix Server'', ''Windows Server'',
                               ''IP Firewall'', ''Router'', ''Switch'')
        ), Actual AS
        (
            SELECT ''Server'' AS AssetGroup, HWID, Splunk_Reporting_Asset AS Flag
            FROM dbo.tbl_C2Sreport_assetCalculationsServer
            UNION
            SELECT ''FRS'', HWID, Splunk_Reporting_Asset
            FROM dbo.tbl_C2Sreport_assetCalculationsFRS
        ), MissingOrChanged AS
        (
            SELECT AssetGroup, HWID, Flag FROM Expected
            EXCEPT
            SELECT AssetGroup, HWID, Flag FROM Actual
        ), ExtraOrChanged AS
        (
            SELECT AssetGroup, HWID, Flag FROM Actual
            EXCEPT
            SELECT AssetGroup, HWID, Flag FROM Expected
        ), ConflictingFlags AS
        (
            SELECT AssetGroup, HWID
            FROM Actual
            GROUP BY AssetGroup, HWID
            HAVING COUNT(DISTINCT Flag) > 1
        ), MissingDimensionPairs AS
        (
            SELECT Splunk_Hostname, Splunk_Reporting_Asset
            FROM dbo.view_c2sreport_tbl_raw_historical_splunk_hosts_raw
            EXCEPT
            SELECT Splunk_Hostname, Splunk_Reporting_Asset FROM dbo.dimSplunk
        ), ExtraDimensionPairs AS
        (
            SELECT Splunk_Hostname, Splunk_Reporting_Asset FROM dbo.dimSplunk
            EXCEPT
            SELECT Splunk_Hostname, Splunk_Reporting_Asset
            FROM dbo.view_c2sreport_tbl_raw_historical_splunk_hosts_raw
        )
        SELECT N''Dimension missing or changed pairs'', COUNT_BIG(*) FROM MissingDimensionPairs
        UNION ALL SELECT N''Dimension extra or changed pairs'', COUNT_BIG(*) FROM ExtraDimensionPairs
        UNION ALL SELECT N''Server missing or changed flags'', COUNT_BIG(*) FROM MissingOrChanged WHERE AssetGroup = ''Server''
        UNION ALL SELECT N''Server extra or changed flags'', COUNT_BIG(*) FROM ExtraOrChanged WHERE AssetGroup = ''Server''
        UNION ALL SELECT N''FRS missing or changed flags'', COUNT_BIG(*) FROM MissingOrChanged WHERE AssetGroup = ''FRS''
        UNION ALL SELECT N''FRS extra or changed flags'', COUNT_BIG(*) FROM ExtraOrChanged WHERE AssetGroup = ''FRS''
        UNION ALL SELECT N''Assets with conflicting Splunk flags'', COUNT_BIG(*) FROM ConflictingFlags
        UNION ALL SELECT N''Invalid or NULL Splunk asset flags'', COUNT_BIG(*)
            FROM Actual WHERE Flag IS NULL OR Flag NOT IN (0, 1, 2, 4)
        UNION ALL SELECT N''Orphan fact Splunk IDs'', COUNT_BIG(*)
            FROM dbo.factCyberEvents F
            WHERE F.SPLUNKID IS NOT NULL
              AND NOT EXISTS (SELECT 1 FROM dbo.dimSplunk S WHERE S.SPLUNKID = F.SPLUNKID);';

    SELECT @FailedChecks = COUNT_BIG(*) FROM @Checks WHERE Result <> 0;

    -- Result grid 1: every Result should be zero.
    SELECT CheckName, Result, CAST(0 AS bigint) AS Expected
    FROM @Checks;

    -- Result grid 2: actual scope tested. These are distinct assets per
    -- status, not fact-row counts and not a comparison with PROD data.
    EXEC sys.sp_executesql N'
        SELECT ''Server'' AS AssetGroup, Splunk_Reporting_Asset,
               COUNT_BIG(DISTINCT HWID) AS Distinct_Asset_Count
        FROM dbo.tbl_C2Sreport_assetCalculationsServer
        GROUP BY Splunk_Reporting_Asset
        UNION ALL
        SELECT ''FRS'', Splunk_Reporting_Asset, COUNT_BIG(DISTINCT HWID)
        FROM dbo.tbl_C2Sreport_assetCalculationsFRS
        GROUP BY Splunk_Reporting_Asset
        ORDER BY AssetGroup, Splunk_Reporting_Asset;';

    SET @Stage = N'Rollback test rebuilds';
    ROLLBACK TRANSACTION;

    -- Read the restored objects in a fresh compilation after rollback.
    EXEC sys.sp_executesql N'
        SELECT @D = COUNT_BIG(*) FROM dbo.dimSplunk;
        SELECT @F = COUNT_BIG(*) FROM dbo.factCyberEvents;
        SELECT @S = COUNT_BIG(*) FROM dbo.tbl_C2Sreport_assetCalculationsServer;
        SELECT @R = COUNT_BIG(*) FROM dbo.tbl_C2Sreport_assetCalculationsFRS;',
        N'@D bigint OUTPUT, @F bigint OUTPUT, @S bigint OUTPUT, @R bigint OUTPUT',
        @D = @DimAfter OUTPUT, @F = @FactAfter OUTPUT,
        @S = @ServerAfter OUTPUT, @R = @FRSAfter OUTPUT;

    -- Result grid 3: all Before_Test and After_Rollback values must match.
    SELECT N'dimSplunk' AS Component, @DimBefore AS Before_Test, @DimAfter AS After_Rollback
    UNION ALL SELECT N'factCyberEvents', @FactBefore, @FactAfter
    UNION ALL SELECT N'Asset calculations - Server', @ServerBefore, @ServerAfter
    UNION ALL SELECT N'Asset calculations - FRS', @FRSBefore, @FRSAfter;

    -- Result grid 4: confirms completion and whether checks passed.
    SELECT CASE WHEN @FailedChecks = 0
                     AND @DimBefore = @DimAfter AND @FactBefore = @FactAfter
                     AND @ServerBefore = @ServerAfter AND @FRSBefore = @FRSAfter
                     AND @@TRANCOUNT = 0
                THEN N'PASS - test rolled back'
                ELSE N'REVIEW - test rolled back; inspect results' END AS TestStatus,
           DB_NAME() AS DatabaseName,
           @@TRANCOUNT AS OpenTransactions,
           @SourceRows AS Source_View_Rows,
           @FailedChecks AS Failed_Checks;
END TRY
BEGIN CATCH
    IF XACT_STATE() <> 0 ROLLBACK TRANSACTION;
    SELECT @Stage AS Failed_Stage,
           ERROR_NUMBER() AS Error_Number,
           ERROR_MESSAGE() AS Error_Message,
           @@TRANCOUNT AS OpenTransactions;
    THROW;
END CATCH;
