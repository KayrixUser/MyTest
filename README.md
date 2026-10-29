SELECT
    COUNT_BIG(CASE WHEN HasExisting = 1 THEN 1 END) AS ExistingVariants,
    COUNT_BIG(CASE WHEN HasLanding = 1 THEN 1 END) AS LandingVariants,
    COUNT_BIG(CASE WHEN HasExisting = 1 AND HasLanding = 0
                   THEN 1 END) AS ExistingOnly,
    COUNT_BIG(CASE WHEN HasLanding = 1 AND HasExisting = 0
                   THEN 1 END) AS LandingOnly
FROM (
    SELECT
        Hostname, BusinessUnit, FQDN, LastScanned,
        TotalOpen, Open30Days, IsExternal, VerdantURL,
        MAX(CASE WHEN SourceName = N'Existing'
                 THEN 1 ELSE 0 END) AS HasExisting,
        MAX(CASE WHEN SourceName = N'Landing'
                 THEN 1 ELSE 0 END) AS HasLanding
    FROM Variants
    GROUP BY
        Hostname, BusinessUnit, FQDN, LastScanned,
        TotalOpen, Open30Days, IsExternal, VerdantURL
) AS V;
