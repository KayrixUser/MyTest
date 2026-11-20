Test ID	Validation	Expected result	Current finding	Status
HW01	Refresh timing	Compare data from the same refresh cycle.	The exports show different latest update dates. Confirm the source cutoffs and that Matillion finished before C2S started.	Pending
HW02	Required fields and wireless fix	Both exports contain all 17 required fields, and wireless approval matches.	All 17 fields are present on both sides. Wireless approval matched on all 66,058 shared records. Both sides had a value on 58,696 and were blank on the rest.	Pass
HW03	Hardware records included	Both systems include the same hardware records.	SQL Server had 66,196 records and Snowflake had 72,871. Of those, 66,058 appeared in both, 138 appeared only in SQL Server, and 6,813 appeared only in Snowflake.	Fail
HW04	Missing IDs	Every record has a Content_ID and a Hardware_ID.	Neither export had a blank Content_ID or Hardware_ID.	Pass
HW05	Repeated IDs	Each Content_ID and Hardware_ID appears once per export.	Neither export had a repeated Content_ID or Hardware_ID.	Pass
HW06	Hardware ID format	The same hardware record has the expected Hardware_ID in both systems.	Hardware_ID differed on all 66,058 shared records. Snowflake adds the HW- prefix. Confirm which format C2S needs.	Fail
HW07	Hardware details	Basic hardware details match for shared records.	Differences by field: hardware name 70, host name 67, internal IP address 3, MAC address 170, and type 11.	Fail
HW08	Authorization package	The same hardware record is linked to the same package names.	Package names differed on 4,358 of the 66,058 shared records. Confirm which package links C2S should use.	Fail
HW09	Hardware dates	Last_Updated and ConfirmedInArcher can be compared and match on shared records.	Last_Updated differed on all 66,058 shared records. The latest was Nov 19, 2026 at 17:18:00 in SQL Server and Nov 13, 2026 at 23:21:21 in Snowflake. Power Query could not read the SQL ConfirmedInArcher dates, so that field still needs a format check.	Fail
HW10	Deviation approvals	Approval values match on shared hardware records.	Tanium differed on 98 records. The other six approval fields matched, including wireless.	Fail
HW11	Derived host names	C2S produces the expected host names.	Check Archer_Asset_Name and HostNameFirstPartHW in the downstream C2S views.	Pending
HW12	C2S hardware and package joins	C2S keeps the expected hardware records and package links after refresh.	The export comparison is complete. The downstream hardware views, dimHardware, and package joins still need a check.	Pending
HW13	Overall Hardware UAT	Required checks pass or the remaining differences are explained and accepted.	The wireless issue is closed. Record counts, hardware IDs, field values, and downstream checks remain open for sign-off.	Fail
