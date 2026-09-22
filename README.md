USE [C2S_Development];
GO

IF DB_NAME() <> N'C2S_Development'
    THROW 51000, 'This test must run in C2S_Development.', 1;

IF @@TRANCOUNT <> 0
    THROW 51001, 'Use a new window with no open transaction.', 1;

SET NOCOUNT ON;
SET XACT_ABORT ON;

BEGIN TRY
    BEGIN TRANSACTION;

    -- Test the existing Splunk loading logic.
    EXEC(N'DROP VIEW dbo.dimSplunk;');

    EXEC(N'
        DROP TABLE dbo.tbl_C2Sreport_raw_historical_splunk_hosts_raw;
    ');

    EXEC(N'
        SELECT IDENTITY(INT, 1, 1) AS SPLUNKID, S.*
        INTO dbo.tbl_C2Sreport_raw_historical_splunk_hosts_raw
        FROM (
            SELECT DISTINCT *
            FROM dbo.view_c2sreport_tbl_raw_historical_splunk_hosts_raw
        ) AS S;
    ');

    EXEC(N'
        ALTER TABLE dbo.tbl_C2Sreport_raw_historical_splunk_hosts_raw
        ADD CONSTRAINT PK_SPLUNKID PRIMARY KEY (SPLUNKID);

        ALTER TABLE dbo.tbl_C2Sreport_raw_historical_splunk_hosts_raw
        ADD LastUpdatedDate DATETIME2(3) DEFAULT GETDATE();
    ');

    EXEC(N'
        UPDATE dbo.tbl_C2Sreport_raw_historical_splunk_hosts_raw
        SET LastUpdatedDate = GETDATE();
    ');

    EXEC(N'
        CREATE VIEW dbo.dimSplunk AS
        SELECT *
        FROM dbo.tbl_C2Sreport_raw_historical_splunk_hosts_raw;
    ');

    -- Validate the rebuilt dimension before rolling back.
    EXEC(N'
        WITH Expected AS (
            SELECT Splunk_Hostname, Splunk_Reporting_Asset
            FROM dbo.view_c2sreport_tbl_raw_historical_splunk_hosts_raw
        ),
        Actual AS (
            SELECT Splunk_Hostname, Splunk_Reporting_Asset
            FROM dbo.dimSplunk
        ),
        MissingOrChanged AS (
            SELECT * FROM Expected
            EXCEPT
            SELECT * FROM Actual
        ),
        ExtraOrChanged AS (
            SELECT * FROM Actual
            EXCEPT
            SELECT * FROM Expected
        )
        SELECT ''Source view rows'' AS CheckName,
               COUNT_BIG(*) AS Result
        FROM Expected
        UNION ALL
        SELECT ''Stored table rows'', COUNT_BIG(*)
        FROM dbo.tbl_C2Sreport_raw_historical_splunk_hosts_raw
        UNION ALL
        SELECT ''dimSplunk rows'', COUNT_BIG(*) FROM Actual
        UNION ALL
        SELECT ''Missing or changed'', COUNT_BIG(*)
        FROM MissingOrChanged
        UNION ALL
        SELECT ''Extra or changed'', COUNT_BIG(*)
        FROM ExtraOrChanged
        UNION ALL
        SELECT ''Missing LastUpdatedDate'', COUNT_BIG(*)
        FROM dbo.dimSplunk
        WHERE LastUpdatedDate IS NULL;
    ');

    -- This is a test: restore the original DEV objects and data.
    ROLLBACK TRANSACTION;

    SELECT
        N'Test rolled back' AS TestStatus,
        @@TRANCOUNT AS OpenTransactions,
        COUNT_BIG(*) AS Restored_DimSplunk_Rows
    FROM dbo.dimSplunk;
END TRY
BEGIN CATCH
    IF XACT_STATE() <> 0
        ROLLBACK TRANSACTION;
    THROW;
END CATCH;
