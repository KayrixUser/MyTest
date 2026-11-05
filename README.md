USE [C2S];
GO

SET ANSI_NULLS ON;
GO
SET QUOTED_IDENTIFIER ON;
GO

ALTER VIEW [dbo].[cATO_Summary]
AS
SELECT
    CONVERT(uniqueidentifier, A.[ASSETGUID]) AS [AssetGuid],
    CONVERT(varchar(500),
        A.[BUSINESS_UNIT] COLLATE SQL_Latin1_General_CP1_CI_AS)
        AS [Business Unit],
    CONVERT(varchar(500),
        A.[HOSTNAME] COLLATE SQL_Latin1_General_CP1_CI_AS)
        AS [Hostname],
    CONVERT(varchar(2000),
        A.[FQDN] COLLATE SQL_Latin1_General_CP1_CI_AS)
        AS [FQDN],
    CONVERT(datetime2(7), A.[LAST_SCANNED]) AS [Last Scanned],
    CONVERT(varchar(4000),
        A.[SAP] COLLATE SQL_Latin1_General_CP1_CI_AS)
        AS [SAP],
    CONVERT(int, A.[TOTAL_OPEN]) AS [TotalOpen],
    CONVERT(int, A.[OPEN_30_DAYS_PAST_SLA])
        AS [Open 30 Days Past SLA],
    CONVERT(varchar(150),
        A.[VERDANT_URL] COLLATE SQL_Latin1_General_CP1_CI_AS)
        AS [VerdantURL],
    CONVERT(bit, A.[IS_EXTERNAL]) AS [is_external]
FROM [ACP_Connection].[dbo].[C2S_CATO_SUMMARY_DATA] AS A;
GO

ALTER VIEW [dbo].[view_c2sreport_tbl_cATO]
AS
WITH TAB AS
(
    SELECT
        A.[BUSINESS_UNIT] AS [Business Unit],
        A.[HOSTNAME] AS [Hostname],
        A.[FQDN] AS [FQDN],
        A.[LAST_SCANNED] AS [Last Scanned],
        A.[SAP] AS [SAP],
        A.[TOTAL_OPEN] AS [TotalOpen],
        A.[OPEN_30_DAYS_PAST_SLA] AS [Open 30 Days Past SLA],
        A.[IS_EXTERNAL] AS [is_external],
        A.[VERDANT_URL] AS [VerdantURL],
        ROW_NUMBER() OVER
        (
            PARTITION BY A.[HOSTNAME]
            ORDER BY A.[LAST_SCANNED] DESC
        ) AS dupCount
    FROM [ACP_Connection].[dbo].[C2S_CATO_SUMMARY_DATA] AS A
    INNER JOIN [dbo].[ArcherMetricsHardwareBaseTable] AS B
        ON B.[HostNameFirstPartHW] = A.[HOSTNAME]
)
SELECT
    CONVERT(varchar(500),
        [Business Unit] COLLATE SQL_Latin1_General_CP1_CI_AS)
        AS [Business Unit],
    CONVERT(varchar(500),
        [Hostname] COLLATE SQL_Latin1_General_CP1_CI_AS)
        AS [Hostname],
    CONVERT(varchar(2000),
        [FQDN] COLLATE SQL_Latin1_General_CP1_CI_AS)
        AS [FQDN],
    [Last Scanned],
    CONVERT(int, [TotalOpen]) AS [TotalOpen],
    CONVERT(int, [Open 30 Days Past SLA])
        AS [Open 30 Days Past SLA],
    [is_external],
    CONVERT(varchar(150),
        [VerdantURL] COLLATE SQL_Latin1_General_CP1_CI_AS)
        AS [VerdantURL]
FROM TAB
WHERE dupCount = 1;
GO
