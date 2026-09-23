
IF DB_NAME() <> N'C2S_Development'
    THROW 51000, 'This test must run only in C2S_Development.', 1;
IF @@TRANCOUNT <> 0
    THROW 51001, 'Use a fresh query window with no open transaction.', 1;

SET IMPLICIT_TRANSACTIONS OFF;
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
   OR OBJECT_ID(N'dbo.CreateSAPCalculationTables', N'P') IS NULL
   OR OBJECT_ID(N'dbo.dimAuthorizationPackage', N'V') IS NULL
   OR OBJECT_ID(N'dbo.Splunk_Reporting_SAP', N'V') IS NULL
   OR OBJECT_ID(N'dbo.tbl_SAP_Calculations', N'U') IS NULL
   OR OBJECT_ID(N'dbo.tbl_SAP_Riskscore', N'U') IS NULL
   OR OBJECT_ID(N'dbo.tbl_SAP_FRS_Riskscore', N'U') IS NULL
    THROW 51002, 'A required existing DEV object was not found. No rebuild was started.', 1;

DECLARE @DimBefore bigint, @FactBefore bigint,
        @ServerBefore bigint, @FRSBefore bigint,
        @DimAfter bigint, @FactAfter bigint,
        @ServerAfter bigint, @FRSAfter bigint,
        @SAPBefore bigint, @SAPAfter bigint,
        @RiskBefore bigint, @RiskAfter bigint,
        @FRSRiskBefore bigint, @FRSRiskAfter bigint,
        @RiskNullBefore bigint, @FRSRiskNullBefore bigint,
        @RiskNullTest bigint, @FRSRiskNullTest bigint,
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
    SELECT @SAPBefore = COUNT_BIG(*) FROM dbo.tbl_SAP_Calculations;
    SELECT @RiskBefore = COUNT_BIG(*),
           @RiskNullBefore = COUNT_BIG(CASE WHEN Risk_Score IS NULL THEN 1 END)
    FROM dbo.tbl_SAP_Riskscore;
    SELECT @FRSRiskBefore = COUNT_BIG(*),
           @FRSRiskNullBefore = COUNT_BIG(CASE WHEN Risk_Score IS NULL THEN 1 END)
    FROM dbo.tbl_SAP_FRS_Riskscore;
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

    SET @Stage = N'Temporary SAP and risk calculation rebuild';
    EXEC [C2S_Development].[dbo].[CreateSAPCalculationTables];

    SET @Stage = N'Validate SAP Splunk integration';
    INSERT INTO @Checks (CheckName, Result)
    EXEC sys.sp_executesql N'
        -- Local temporary tables exist only inside this dynamic batch.
        -- DISTINCT SAP/HWID pairs avoid repeatedly processing ~9 million facts.
        SELECT DISTINCT F.SAPID, F.HWID
        INTO #SAPHardware
        FROM dbo.factCyberEvents F;

        WITH AssetStatus AS
        (
            SELECT ''Server'' AS AssetGroup, HWID, Splunk_Reporting_Asset AS Flag
            FROM dbo.tbl_C2Sreport_assetCalculationsServer
            UNION
            SELECT ''FRS'', HWID, Splunk_Reporting_Asset
            FROM dbo.tbl_C2Sreport_assetCalculationsFRS
        ), Counts AS
        (
            SELECT F.SAPID, A.Authorization_Package_Name, S.AssetGroup,
                   COUNT(DISTINCT F.HWID) AS TotalAssets,
                   COUNT(DISTINCT CASE WHEN S.Flag < 3 THEN F.HWID END) AS Applicable,
                   COUNT(DISTINCT CASE WHEN S.Flag = 1 THEN F.HWID END) AS Failed,
                   COUNT(DISTINCT CASE WHEN S.Flag = 2 THEN F.HWID END) AS DataErrors,
                   COUNT(DISTINCT CASE WHEN S.Flag = 4 THEN F.HWID END) AS Exceptions
            FROM #SAPHardware F
            JOIN dbo.dimAuthorizationPackage A ON A.SAPID = F.SAPID
            JOIN AssetStatus S ON S.HWID = F.HWID
            GROUP BY F.SAPID, A.Authorization_Package_Name, S.AssetGroup
        ), Fractions AS
        (
            SELECT SAPID, Authorization_Package_Name, AssetGroup,
                   CASE WHEN Exceptions = TotalAssets THEN 4.0
                        WHEN DataErrors = TotalAssets THEN 2.0
                        ELSE COALESCE(CAST(Failed AS float) / NULLIF(Applicable, 0), 0)
                   END AS Fraction
            FROM Counts
        ), Flags AS
        (
            SELECT SAPID, Authorization_Package_Name, AssetGroup,
                   CASE WHEN Fraction < 0.2 THEN 0
                        WHEN Fraction BETWEEN 0.2 AND 1.0 THEN 1
                        ELSE Fraction END AS Flag
            FROM Fractions
        )
        SELECT SAPID, Authorization_Package_Name,
               MAX(CASE WHEN AssetGroup = ''Server'' THEN Flag END) AS Splunk_Reporting_SAP,
               MAX(CASE WHEN AssetGroup = ''FRS'' THEN Flag END) AS Splunk_Reporting_SAP_FRS
        INTO #ExpectedSAP
        FROM Flags
        GROUP BY SAPID, Authorization_Package_Name;

        SELECT SAPID, Authorization_Package_Name,
               Splunk_Reporting_SAP, Splunk_Reporting_SAP_FRS
        INTO #ActualReport
        FROM dbo.Splunk_Reporting_SAP;

        -- The existing calculation procedure joins the report by SAPID only.
        -- Keep that behavior; do not silently add a package-name join here.
        SELECT DISTINCT F.SAPID, R.Splunk_Reporting_SAP, R.Splunk_Reporting_SAP_FRS
        INTO #ExpectedStored
        FROM #SAPHardware F
        JOIN dbo.dimAuthorizationPackage A ON A.SAPID = F.SAPID
        LEFT JOIN #ActualReport R ON R.SAPID = F.SAPID;

        SELECT DISTINCT F.SAPID,
               CASE WHEN H.[type] IN (''Linux Server'', ''Unix Server'', ''Windows Server'')
                    THEN ''Server'' ELSE ''FRS'' END AS AssetGroup
        INTO #RiskScope
        FROM #SAPHardware F
        JOIN dbo.dimHardware H ON H.HWID = F.HWID
        JOIN dbo.tbl_SAP_Calculations C ON C.SAPID = F.SAPID
        WHERE H.[type] IN (''Linux Server'', ''Unix Server'', ''Windows Server'',
                           ''IP Firewall'', ''Router'', ''Switch'');

        WITH ReportMissing AS
        (
            SELECT * FROM #ExpectedSAP EXCEPT SELECT * FROM #ActualReport
        ), ReportExtra AS
        (
            SELECT * FROM #ActualReport EXCEPT SELECT * FROM #ExpectedSAP
        ), StoredMissing AS
        (
            SELECT * FROM #ExpectedStored
            EXCEPT
            SELECT SAPID, Splunk_Reporting_SAP, Splunk_Reporting_SAP_FRS
            FROM dbo.tbl_SAP_Calculations
        ), StoredExtra AS
        (
            SELECT SAPID, Splunk_Reporting_SAP, Splunk_Reporting_SAP_FRS
            FROM dbo.tbl_SAP_Calculations
            EXCEPT SELECT * FROM #ExpectedStored
        ), ActualRiskScope AS
        (
            SELECT SAPID, ''Server'' AS AssetGroup FROM dbo.tbl_SAP_Riskscore
            UNION
            SELECT SAPID, ''FRS'' FROM dbo.tbl_SAP_FRS_Riskscore
        ), RiskMissing AS
        (
            SELECT SAPID, AssetGroup FROM #RiskScope
            EXCEPT SELECT SAPID, AssetGroup FROM ActualRiskScope
        ), RiskExtra AS
        (
            SELECT SAPID, AssetGroup FROM ActualRiskScope
            EXCEPT SELECT SAPID, AssetGroup FROM #RiskScope
        )
        SELECT N''SAP reporting: missing or changed flags'', COUNT_BIG(*) FROM ReportMissing
        UNION ALL SELECT N''SAP reporting: extra or changed flags'', COUNT_BIG(*) FROM ReportExtra
        UNION ALL SELECT N''Stored SAP calculations: missing or changed flags'', COUNT_BIG(*) FROM StoredMissing
        UNION ALL SELECT N''Stored SAP calculations: extra or changed flags'', COUNT_BIG(*) FROM StoredExtra
        UNION ALL SELECT N''Server risk: missing SAP IDs'', COUNT_BIG(*) FROM RiskMissing WHERE AssetGroup = ''Server''
        UNION ALL SELECT N''Server risk: extra SAP IDs'', COUNT_BIG(*) FROM RiskExtra WHERE AssetGroup = ''Server''
        UNION ALL SELECT N''FRS risk: missing SAP IDs'', COUNT_BIG(*) FROM RiskMissing WHERE AssetGroup = ''FRS''
        UNION ALL SELECT N''FRS risk: extra SAP IDs'', COUNT_BIG(*) FROM RiskExtra WHERE AssetGroup = ''FRS''
        UNION ALL SELECT N''Stored SAP calculations: NULL SAP IDs'', COUNT_BIG(*)
            FROM dbo.tbl_SAP_Calculations WHERE SAPID IS NULL
        UNION ALL SELECT N''Stored SAP calculations: unexpectedly empty output'',
            CAST(CASE WHEN EXISTS (SELECT 1 FROM dbo.tbl_SAP_Calculations) THEN 0 ELSE 1 END AS bigint)
        UNION ALL SELECT N''Stored SAP calculations: invalid Splunk flags'', COUNT_BIG(*)
            FROM dbo.tbl_SAP_Calculations
            WHERE Splunk_Reporting_SAP NOT IN (0, 1, 2, 4)
               OR Splunk_Reporting_SAP_FRS NOT IN (0, 1, 2, 4);
        -- NULL category flags are allowed when that category has no assets.
        -- Expected/actual comparisons above catch unexpected NULLs.
';

    SELECT @FailedChecks = COUNT_BIG(*) FROM @Checks WHERE Result <> 0;

    -- Grid 1: all comparison results should be zero.
    SELECT CheckName, Result, CAST(0 AS bigint) AS Expected FROM @Checks;

    -- Grid 2: final risk outputs. NULL scores require review; they may come
    -- from existing non-Splunk metrics. Do not change any weights to hide them.
    EXEC sys.sp_executesql N'
        SELECT @S = COUNT_BIG(*) FROM dbo.tbl_SAP_Riskscore WHERE Risk_Score IS NULL;
        SELECT @F = COUNT_BIG(*) FROM dbo.tbl_SAP_FRS_Riskscore WHERE Risk_Score IS NULL;
        SELECT ''Server'' AS AssetGroup, COUNT_BIG(*) AS Risk_Rows,
               COUNT_BIG(DISTINCT SAPID) AS Distinct_SAPs,
               COUNT_BIG(CASE WHEN Risk_Score IS NULL THEN 1 END) AS NULL_Risk_Scores,
               @SB AS NULL_Risk_Scores_Before_Test,
               MIN(Risk_Score) AS Minimum_Risk, MAX(Risk_Score) AS Maximum_Risk
        FROM dbo.tbl_SAP_Riskscore
        UNION ALL
        SELECT ''FRS'', COUNT_BIG(*), COUNT_BIG(DISTINCT SAPID),
               COUNT_BIG(CASE WHEN Risk_Score IS NULL THEN 1 END), @FB,
               MIN(Risk_Score), MAX(Risk_Score)
        FROM dbo.tbl_SAP_FRS_Riskscore;',
        N'@S bigint OUTPUT, @F bigint OUTPUT, @SB bigint, @FB bigint',
        @S = @RiskNullTest OUTPUT, @F = @FRSRiskNullTest OUTPUT,
        @SB = @RiskNullBefore, @FB = @FRSRiskNullBefore;

    -- Grid 3: Splunk SAP status distribution; counts need not match PROD.
    EXEC sys.sp_executesql N'
        SELECT V.AssetGroup, V.Splunk_Flag,
               COUNT_BIG(DISTINCT C.SAPID) AS Distinct_SAPs
        FROM dbo.tbl_SAP_Calculations C
        CROSS APPLY (VALUES (''Server'', C.Splunk_Reporting_SAP),
                            (''FRS'', C.Splunk_Reporting_SAP_FRS)) V(AssetGroup, Splunk_Flag)
        GROUP BY V.AssetGroup, V.Splunk_Flag
        ORDER BY V.AssetGroup, V.Splunk_Flag;';

    SET @Stage = N'Rollback all test rebuilds';
    ROLLBACK TRANSACTION;

    EXEC sys.sp_executesql N'
        SELECT @D = COUNT_BIG(*) FROM dbo.dimSplunk;
        SELECT @F = COUNT_BIG(*) FROM dbo.factCyberEvents;
        SELECT @S = COUNT_BIG(*) FROM dbo.tbl_C2Sreport_assetCalculationsServer;
        SELECT @R = COUNT_BIG(*) FROM dbo.tbl_C2Sreport_assetCalculationsFRS;
        SELECT @C = COUNT_BIG(*) FROM dbo.tbl_SAP_Calculations;
        SELECT @SR = COUNT_BIG(*) FROM dbo.tbl_SAP_Riskscore;
        SELECT @FR = COUNT_BIG(*) FROM dbo.tbl_SAP_FRS_Riskscore;',
        N'@D bigint OUTPUT, @F bigint OUTPUT, @S bigint OUTPUT, @R bigint OUTPUT,
          @C bigint OUTPUT, @SR bigint OUTPUT, @FR bigint OUTPUT',
        @D = @DimAfter OUTPUT, @F = @FactAfter OUTPUT,
        @S = @ServerAfter OUTPUT, @R = @FRSAfter OUTPUT,
        @C = @SAPAfter OUTPUT, @SR = @RiskAfter OUTPUT, @FR = @FRSRiskAfter OUTPUT;

    -- Grid 4: all before/restored counts must match.
    SELECT N'dimSplunk' AS Component, @DimBefore AS Before_Test, @DimAfter AS After_Rollback
    UNION ALL SELECT N'factCyberEvents', @FactBefore, @FactAfter
    UNION ALL SELECT N'Asset calculations - Server', @ServerBefore, @ServerAfter
    UNION ALL SELECT N'Asset calculations - FRS', @FRSBefore, @FRSAfter
    UNION ALL SELECT N'SAP calculations', @SAPBefore, @SAPAfter
    UNION ALL SELECT N'SAP risk - Server', @RiskBefore, @RiskAfter
    UNION ALL SELECT N'SAP risk - FRS', @FRSRiskBefore, @FRSRiskAfter;

    -- Grid 5: PASS is limited to the integration checks described in the header.
    SELECT CASE WHEN @FailedChecks = 0
                     AND @RiskNullTest = 0 AND @FRSRiskNullTest = 0
                     AND @DimBefore = @DimAfter AND @FactBefore = @FactAfter
                     AND @ServerBefore = @ServerAfter AND @FRSBefore = @FRSAfter
                     AND @SAPBefore = @SAPAfter AND @RiskBefore = @RiskAfter
                     AND @FRSRiskBefore = @FRSRiskAfter AND @@TRANCOUNT = 0
                THEN N'PASS - SAP integration test rolled back'
                ELSE N'REVIEW - test rolled back; inspect checks and NULL scores'
           END AS TestStatus,
           DB_NAME() AS DatabaseName, @@TRANCOUNT AS OpenTransactions,
           @SourceRows AS Source_View_Rows, @FailedChecks AS Failed_Checks,
           @RiskNullTest + @FRSRiskNullTest AS NULL_Risk_Scores_During_Test;
END TRY
BEGIN CATCH
    IF XACT_STATE() <> 0 ROLLBACK TRANSACTION;
    SELECT @Stage AS Failed_Stage, ERROR_NUMBER() AS Error_Number,
           ERROR_MESSAGE() AS Error_Message, @@TRANCOUNT AS OpenTransactions;
    THROW;
END CATCH;
