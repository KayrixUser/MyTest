IF @@TRANCOUNT <> 0
    THROW 51000, 'Open a new query window with no active transaction.', 1;

IF (@@OPTIONS & 2) = 2
    THROW 51001, 'IMPLICIT_TRANSACTIONS is ON. Use a new window with this option OFF.', 1;

IF OBJECT_ID(N'PA499.cATO_Summary', N'U') IS NULL
    THROW 51002, 'Select C2S: PA499.cATO_Summary is missing or not visible here.', 1;

IF OBJECT_ID(N'tempdb..#CatoRehearsalStage') IS NOT NULL
   OR OBJECT_ID(N'tempdb..#CatoRehearsalTarget') IS NOT NULL
   OR OBJECT_ID(N'tempdb..#CatoRehearsalBefore') IS NOT NULL
    THROW 51003, 'Rehearsal temporary tables already exist. Use a new query window.', 1;

-- The intentional failure below relies on the supplied TotalOpen int column.
IF NOT EXISTS
(
    SELECT 1
    FROM sys.columns
    WHERE object_id = OBJECT_ID(N'PA499.cATO_Summary', N'U')
      AND name = N'TotalOpen'
      AND system_type_id = 56
      AND is_identity = 0
      AND is_computed = 0
)
    THROW 51004, 'The TotalOpen definition differs from the reviewed int column.', 1;

SET NOCOUNT ON;
SET ANSI_WARNINGS ON;
SET XACT_ABORT ON;
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
SET LOCK_TIMEOUT 10000;

DECLARE @StageRows bigint;
DECLARE @TargetRows bigint;
DECLARE @CapturedError int = NULL;
DECLARE @LoaderCaught bit = 0;
DECLARE @IntermediateCaught bit = 0;
DECLARE @OuterCaught bit = 0;
DECLARE @BadInteger nvarchar(80) = N'CATO_REHEARSAL_EXPECTED_CONVERSION_FAILURE';

DECLARE @Results table
(
    TestNumber int NOT NULL,
    TestName nvarchar(180) NOT NULL,
    Result varchar(4) NOT NULL,
    RowsChecked bigint NULL,
    ObservedError int NULL
);

BEGIN TRY
    -- TOP (0) reads target metadata; it does not copy production target rows.
    SELECT TOP (0)
        [AssetGuid], [Business Unit], [Hostname], [FQDN], [Last Scanned],
        [SAP], [TotalOpen], [Open 30 Days Past SLA], [is_external], [VerdantURL]
    INTO #CatoRehearsalStage
    FROM [PA499].[cATO_Summary];

    SELECT TOP (0) * INTO #CatoRehearsalTarget FROM #CatoRehearsalStage;
    SELECT TOP (0) * INTO #CatoRehearsalBefore FROM #CatoRehearsalStage;

    -- This is the only ACP data scan. All ten destination columns are explicit.
    -- Assignment performs conversion to the actual destination column types.
    INSERT INTO #CatoRehearsalStage
    (
        [AssetGuid], [Business Unit], [Hostname], [FQDN], [Last Scanned],
        [SAP], [TotalOpen], [Open 30 Days Past SLA], [is_external], [VerdantURL]
    )
    SELECT
        [ASSETGUID], [BUSINESS_UNIT], [HOSTNAME], [FQDN], [LAST_SCANNED],
        [SAP], [TOTAL_OPEN], [OPEN_30_DAYS_PAST_SLA], [IS_EXTERNAL], [VERDANT_URL]
    FROM [ACP_Connection].[dbo].[C2S_CATO_SUMMARY_DATA];

    SELECT @StageRows = COUNT_BIG(*) FROM #CatoRehearsalStage;
    IF @StageRows = 0
        THROW 51005, 'ACP supplied zero rows. No replacement test was attempted.', 1;

    INSERT INTO @Results
    VALUES (1, N'ACP mapped INSERT into target-shaped temporary table', 'PASS', @StageRows, NULL);

    -- One known row represents the previously committed temporary target.
    INSERT INTO #CatoRehearsalBefore
    SELECT TOP (1) * FROM #CatoRehearsalStage;

    INSERT INTO #CatoRehearsalTarget
    SELECT * FROM #CatoRehearsalBefore;

    -- Test a REAL failed INSERT after TRUNCATE, inside an explicit transaction.
    -- Three nested catches model loader -> Weekly_A -> Weekly error propagation.
    -- The outside observer catches the expected error so the test can continue.
    BEGIN TRY
        BEGIN TRY
            BEGIN TRY
                BEGIN TRY
                    BEGIN TRANSACTION;
                    TRUNCATE TABLE #CatoRehearsalTarget;

                    INSERT INTO #CatoRehearsalTarget
                    (
                        [AssetGuid], [Business Unit], [Hostname], [FQDN], [Last Scanned],
                        [SAP], [TotalOpen], [Open 30 Days Past SLA], [is_external], [VerdantURL]
                    )
                    SELECT TOP (1)
                        [AssetGuid], [Business Unit], [Hostname], [FQDN], [Last Scanned],
                        [SAP], @BadInteger, [Open 30 Days Past SLA], [is_external], [VerdantURL]
                    FROM #CatoRehearsalStage;

                    -- The conversion above must fail; never commit this branch.
                    THROW 51006, 'Expected conversion failure did not occur.', 1;
                END TRY
                BEGIN CATCH
                    SET @LoaderCaught = 1;
                    IF XACT_STATE() <> 0 ROLLBACK TRANSACTION;
                    THROW;
                END CATCH;
            END TRY
            BEGIN CATCH
                SET @IntermediateCaught = 1;
                THROW;
            END CATCH;
        END TRY
        BEGIN CATCH
            SET @OuterCaught = 1;
            THROW;
        END CATCH;
    END TRY
    BEGIN CATCH
        SET @CapturedError = ERROR_NUMBER();
        IF @CapturedError <> 245 THROW;
    END CATCH;

    IF @CapturedError IS NULL OR @CapturedError <> 245
       OR @LoaderCaught <> 1 OR @IntermediateCaught <> 1 OR @OuterCaught <> 1
        THROW 51007, 'Expected error did not propagate through every simulated caller.', 1;

    IF @@TRANCOUNT <> 0 OR XACT_STATE() <> 0
        THROW 51008, 'The forced-failure transaction was not fully rolled back.', 1;

    INSERT INTO @Results
    VALUES (2, N'Failed INSERT reaches loader and both simulated callers', 'PASS', NULL, @CapturedError);

    SELECT @TargetRows = COUNT_BIG(*) FROM #CatoRehearsalTarget;
    IF @TargetRows <> 1
        THROW 51009, 'The previously committed temporary target row was not restored.', 1;

    IF EXISTS
    (
        SELECT * FROM #CatoRehearsalBefore
        EXCEPT
        SELECT * FROM #CatoRehearsalTarget
    )
    OR EXISTS
    (
        SELECT * FROM #CatoRehearsalTarget
        EXCEPT
        SELECT * FROM #CatoRehearsalBefore
    )
        THROW 51010, 'Temporary target values differ after rollback.', 1;

    INSERT INTO @Results
    VALUES (3, N'TRUNCATE and failed INSERT rolled back; previous temporary row restored', 'PASS', @TargetRows, NULL);

    -- Normal replacement of the TEMPORARY target, using the staged typed rows.
    BEGIN TRANSACTION;
    TRUNCATE TABLE #CatoRehearsalTarget;

    INSERT INTO #CatoRehearsalTarget
    (
        [AssetGuid], [Business Unit], [Hostname], [FQDN], [Last Scanned],
        [SAP], [TotalOpen], [Open 30 Days Past SLA], [is_external], [VerdantURL]
    )
    SELECT
        [AssetGuid], [Business Unit], [Hostname], [FQDN], [Last Scanned],
        [SAP], [TotalOpen], [Open 30 Days Past SLA], [is_external], [VerdantURL]
    FROM #CatoRehearsalStage;

    SELECT @TargetRows = COUNT_BIG(*) FROM #CatoRehearsalTarget;
    IF @TargetRows <> @StageRows
        THROW 51011, 'Successful-path temporary target count differs from staged rows.', 1;

    -- Include multiplicity so duplicate rows cannot conceal a mismatch.
    IF EXISTS
    (
        SELECT
            [AssetGuid], [Business Unit], [Hostname], [FQDN], [Last Scanned],
            [SAP], [TotalOpen], [Open 30 Days Past SLA], [is_external], [VerdantURL],
            COUNT_BIG(*) AS Copies
        FROM #CatoRehearsalStage
        GROUP BY
            [AssetGuid], [Business Unit], [Hostname], [FQDN], [Last Scanned],
            [SAP], [TotalOpen], [Open 30 Days Past SLA], [is_external], [VerdantURL]
        EXCEPT
        SELECT
            [AssetGuid], [Business Unit], [Hostname], [FQDN], [Last Scanned],
            [SAP], [TotalOpen], [Open 30 Days Past SLA], [is_external], [VerdantURL],
            COUNT_BIG(*) AS Copies
        FROM #CatoRehearsalTarget
        GROUP BY
            [AssetGuid], [Business Unit], [Hostname], [FQDN], [Last Scanned],
            [SAP], [TotalOpen], [Open 30 Days Past SLA], [is_external], [VerdantURL]
    )
        THROW 51012, 'Staged values or duplicate counts differ from the temporary target.', 1;

    -- With equal total counts, matching each staged group and its multiplicity
    -- accounts for every destination row; no second grouped EXCEPT is needed.
    COMMIT TRANSACTION;

    IF @@TRANCOUNT <> 0
        THROW 51013, 'The successful-path transaction did not finish.', 1;

    INSERT INTO @Results
    VALUES (4, N'Successful temporary replacement committed; values and duplicate counts match', 'PASS', @TargetRows, NULL);

    DROP TABLE #CatoRehearsalBefore;
    DROP TABLE #CatoRehearsalTarget;
    DROP TABLE #CatoRehearsalStage;

    SELECT
        @@SERVERNAME AS ServerName,
        DB_NAME() AS DatabaseName,
        TestNumber, TestName, Result, RowsChecked, ObservedError,
        @@TRANCOUNT AS OpenTransactions
    FROM @Results
    ORDER BY TestNumber;
END TRY
BEGIN CATCH
    -- No ambient transaction was allowed; any open transaction is our own.
    IF XACT_STATE() <> 0 ROLLBACK TRANSACTION;
    IF OBJECT_ID(N'tempdb..#CatoRehearsalBefore') IS NOT NULL DROP TABLE #CatoRehearsalBefore;
    IF OBJECT_ID(N'tempdb..#CatoRehearsalTarget') IS NOT NULL DROP TABLE #CatoRehearsalTarget;
    IF OBJECT_ID(N'tempdb..#CatoRehearsalStage') IS NOT NULL DROP TABLE #CatoRehearsalStage;
    THROW;
END CATCH;
