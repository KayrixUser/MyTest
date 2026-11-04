USE [C2S];
GO
SET ANSI_NULLS ON;
GO
SET QUOTED_IDENTIFIER ON;
GO

ALTER VIEW [dbo].[C2S_TLS_Assets]
AS
SELECT
    CONVERT(varchar(256),
        [HOSTNAME] COLLATE SQL_Latin1_General_CP1_CI_AS) AS [hostname],
    CONVERT(varchar(628),
        [FQDN] COLLATE SQL_Latin1_General_CP1_CI_AS) AS [fqdn],
    CONVERT(varchar(256),
        [BUSINESS_UNIT] COLLATE SQL_Latin1_General_CP1_CI_AS) AS [business_unit],
    CONVERT(nvarchar(4000),
        [SAPS] COLLATE SQL_Latin1_General_CP1_CI_AS) AS [SAPs],
    [TITLE] AS [title],
    [DESCRIPTION] AS [description],
    CONVERT(datetime, [FIRST_SEEN_UTC]) AS [first_seen_utc],
    CONVERT(int, [RISK_SCORE]) AS [risk_score],
    [SLA_UTC] AS [sla_utc],
    CONVERT(varchar(64),
        [SEVERITY] COLLATE SQL_Latin1_General_CP1_CI_AS) AS [severity],
    [IS_EXTERNAL] AS [is_external]
FROM [ACP_Connection].[dbo].[C2S_TLS_ASSETS_DATA];
GO

-- Update dependent-view metadata after the base view's types change.
-- This system procedure does not run the daily model or send notifications.
EXEC sys.sp_refreshview N'dbo.view_c2sreport_tbl_TLSVerdant';
GO
