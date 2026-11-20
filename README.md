et
    SqlFile = "C:\YourFolder\SQL Hardware.csv",
    SnowflakeFile = "C:\YourFolder\Snowflake Hardware.csv",

    // SQL PROD_All field, published Snowflake ALL field, comparison rule.
    Fields = #table({"SQL", "SF", "Rule"}, {
        {"Content_ID", "ARCHER_CONTENT_HARDWARE_CONTENT_ID", "Key"},
        {"Wireless_Access_Point_Deviation_Approved", "WIRELESS_ACCESS_POINT_DEVIATION_APPROVED", "Text"},
        {"Hardware_ID", "HARDWARE_ID", "Text"},
        {"Hardware_Name", "HARDWARE_NAME", "Text"},
        {"Host_Name", "HOST_NAME", "Text"},
        {"Internal_IP_Address", "INTERNAL_IP_ADDRESS", "Text"},
        {"MAC_Address", "MAC_ADDRESS", "Text"},
        {"Type", "TYPE", "Text"},
        {"Authorization_Package", "AUTHORIZATION_PACKAGE_NAME", "Package names"},
        {"Last_Updated", "ARCHER_CONTENT_HARDWARE_LAST_UPDATED", "Date"},
        {"ConfirmedInArcher", "ARCHER_CONTENT_HARDWARE_CONFIRMED_IN_ARCHER", "Date"},
        {"Splunk_Deviation_Approved", "SPLUNK_DEVIATION_APPROVED", "Text"},
        {"Tanium_Deviation_Approved", "TANIUM_DEVIATION_APPROVED", "Text"},
        {"Access_Management_Deviation_Approved", "ACCESS_MANAGEMENT_DEVIATION_APPROVED", "Text"},
        {"Privileged_Access_Management_Deviation_Approved", "PRIVILEGED_ACCESS_MANAGEMENT_DEVIATION_APPROVED", "Text"},
        {"Password_Management_Deviation_Approved", "PASSWORD_MANAGEMENT_DEVIATION_APPROVED", "Text"},
        {"Session_Control_Deviation_Approved", "SESSION_CONTROL_DEVIATION_APPROVED", "Text"}
    }),

    Clean = (v as nullable text) as nullable text =>
        let s = if v = null then "" else Text.Trim(v)
        in if List.Contains({"", "NULL", "(NULL)", "\N"}, Text.Upper(s)) then null else s,

    ReadCsv = (path as text, sourceHeaders as list) as record =>
        let
            bytes = Binary.Buffer(File.Contents(path)),
            bom = if Binary.Length(bytes) < 2 then "" else Binary.ToText(Binary.Range(bytes, 0, 2), BinaryEncoding.Hex),
            enc = if Text.Upper(bom) = "FFFE" then 1200 else if Text.Upper(bom) = "FEFF" then 1201 else
                let utf8 = try Text.FromBinary(bytes, 65001)
                in if utf8[HasError] then 1252 else if Text.Contains(utf8[Value], Character.FromNumber(65533)) then 1252 else 65001,
            csv = Csv.Document(bytes, [Delimiter = ",", Encoding = enc, QuoteStyle = QuoteStyle.Csv]),
            headers = Table.TransformColumnNames(Table.PromoteHeaders(csv, [PromoteAllScalars = true]),
                each Text.Upper(Text.Trim(Text.Replace(_, Character.FromNumber(65279), "")))),
            missing = List.Difference(sourceHeaders, Table.ColumnNames(headers)),
            selected = Table.SelectColumns(headers, sourceHeaders, MissingField.UseNull),
            renamed = Table.RenameColumns(selected, List.Select(List.Zip({sourceHeaders, Fields[SQL]}), each _{0} <> _{1})),
            cleaned = Table.TransformColumns(renamed, List.Transform(Fields[SQL], each {_, Clean, type nullable text}))
        in [Data = Table.Buffer(cleaned), Missing = missing],

    Sql = ReadCsv(SqlFile, List.Transform(Fields[SQL], each Text.Upper(_))),
    Sf = ReadCsv(SnowflakeFile, Fields[SF]),
    SqlData = Sql[Data],
    SfData = Sf[Data],
    NSql = Table.RowCount(SqlData),
    NSf = Table.RowCount(SfData),
    BothHaveRows = NSql > 0 and NSf > 0,
    HasField = (f as record) as logical =>
        not List.Contains(Sql[Missing], Text.Upper(f[SQL])) and not List.Contains(Sf[Missing], f[SF]),
    HasKey = HasField(Fields{0}),
    HasHardware = HasField(Fields{[SQL = "Hardware_ID"]}),
    Profiles = (t as table, col as text, n as text) as table =>
        let one = Table.SelectColumns(t, {col}), populated = Table.SelectRows(one, each Record.Field(_, col) <> null)
        in Table.Buffer(Table.Group(populated, {col}, {{n, each Table.RowCount(_), Int64.Type}})),
    SqlKeys = Profiles(SqlData, "Content_ID", "SQLRows"),
    SfKeys = Profiles(SfData, "Content_ID", "SFRows"),
    SqlHw = Profiles(SqlData, "Hardware_ID", "SQLRows"),
    SfHw = Profiles(SfData, "Hardware_ID", "SFRows"),

    KeyJoin = Table.Join(SqlKeys, {"Content_ID"}, Table.RenameColumns(SfKeys, {{"Content_ID", "SfID"}}), {"SfID"}, JoinKind.FullOuter),
    SqlOnly = Table.Buffer(Table.SelectRows(KeyJoin, each [SFRows] = null)),
    SfOnly = Table.Buffer(Table.SelectRows(KeyJoin, each [SQLRows] = null)),
    Shared = Table.SelectRows(KeyJoin, each [SFRows] <> null and [SQLRows] <> null),
    UniqueKeys = Table.SelectColumns(Table.SelectRows(Shared, each [SQLRows] = 1 and [SFRows] = 1), {"Content_ID"}),
    SqlMatched = Table.Join(Table.RenameColumns(UniqueKeys, {{"Content_ID", "JoinID"}}), {"JoinID"}, SqlData, {"Content_ID"}, JoinKind.Inner),
    Pairs = Table.Buffer(Table.Join(SqlMatched, {"Content_ID"}, Table.PrefixColumns(SfData, "SF"), {"SF.Content_ID"}, JoinKind.Inner)),
    NPairs = Table.RowCount(Pairs),
    NShared = Table.RowCount(Shared),
    NOnly = Table.RowCount(SqlOnly) + Table.RowCount(SfOnly),

    Sample = (values as list) as text => Text.Combine(List.FirstN(values, 3), ", "),
    Num = (n as number) as text => Number.ToText(n, "#,##0", "en-US"),
    Row = (check as text, sql as any, sf as any, compared as nullable number, diff as nullable number, status as text, note as text) as list =>
        {check, sql, sf, compared, diff, status, note},
    Grade = (available as logical, ok as logical) as text => if not available then "Review" else if ok then "Pass" else "Fail",
    SumCounts = (values as list) as number => List.Sum({0} & values),
    KeyChecks = (label as text, a as table, b as table, available as logical) as list =>
        let
            na = Table.RowCount(a), nb = Table.RowCount(b),
            rowsA = SumCounts(a[SQLRows]), rowsB = SumCounts(b[SFRows]),
            ready = available and BothHaveRows,
            repeatA = Table.SelectRows(a, each [SQLRows] > 1),
            repeatB = Table.SelectRows(b, each [SFRows] > 1),
            repeatNote = if not available then "Check the required headers."
                else if rowsA = na and rowsB = nb then "Each populated ID occurs once in its file."
                else "Repeated ID examples. SQL: " & Sample(Table.Column(repeatA, label)) & ". Snowflake: " & Sample(Table.Column(repeatB, label)) & "."
        in {
            Row(label & " - different IDs in each file", if available then na else null, if available then nb else null, null, null,
                if available then "Info" else "Review", "Each populated ID counted once."),
            Row(label & " - rows with a missing ID", if available then NSql - rowsA else null, if available then NSf - rowsB else null,
                null, null, Grade(ready, rowsA = NSql and rowsB = NSf), "Expected zero on both sides."),
            Row(label & " - extra rows with a repeated ID", if available then rowsA - na else null, if available then rowsB - nb else null,
                null, null, Grade(ready, rowsA = na and rowsB = nb), repeatNote)
        },
    HeaderNote = (label as text, names as list) as text =>
        label & (if List.IsEmpty(names) then " complete." else " missing: " & Text.Combine(names, ", ") & "."),
    MappingOK = List.IsEmpty(Sql[Missing]) and List.IsEmpty(Sf[Missing]),
    Wireless = Fields{[SQL = "Wireless_Access_Point_Deviation_Approved"]},
    Core = {
        Row("Required field headers", Table.RowCount(Fields) - List.Count(Sql[Missing]), Table.RowCount(Fields) - List.Count(Sf[Missing]), null, null,
            if MappingOK then "Pass" else "Fail", HeaderNote("SQL", Sql[Missing]) & " " & HeaderNote("Snowflake", Sf[Missing])),
        Row("Wireless approval field present", if List.Contains(Sql[Missing], Text.Upper(Wireless[SQL])) then "Missing" else "Present",
            if List.Contains(Sf[Missing], Wireless[SF]) then "Missing" else "Present", null, null,
            if HasField(Wireless) then "Pass" else "Fail", "The separate wireless value check uses matched hardware records."),
        Row("Total hardware rows", NSql, NSf, null, Number.Abs(NSql - NSf), Grade(BothHaveRows, NSql = NSf), "All data rows in the two exports.")
    } & KeyChecks("Content_ID", SqlKeys, SfKeys, HasKey) & KeyChecks("Hardware_ID", SqlHw, SfHw, HasHardware) & {
        Row("Content_IDs found on one side only", if HasKey then Table.RowCount(SqlOnly) else null, if HasKey then Table.RowCount(SfOnly) else null,
            if HasKey then NShared else null, if HasKey then NOnly else null, Grade(HasKey and BothHaveRows, NOnly = 0),
            if not HasKey then "Content_ID header missing. Compare again with both keys present." else
            "Compared = IDs in both. SQL-only examples: " & Sample(SqlOnly[Content_ID]) & ". Snowflake-only examples: " & Sample(SfOnly[SfID]) & "."),
        Row("Records used for field checks", null, null, NPairs, NShared - NPairs, Grade(HasKey and NPairs > 0, NShared = NPairs),
            "One row per shared Content_ID on each side. Repeated shared IDs are excluded.")
    },

    // Preserve case and ID prefixes. Compare package name lists without relying on their order.
    PackageNames = (s as text) as list => List.Sort(List.Transform(Text.Split(s, ";"), each Text.Trim(_)), Comparer.Ordinal),
    // Both source fields are timezone-free. A zoned or unreadable value needs review.
    AsDate = (s as nullable text) as nullable datetime =>
        if s = null then null else
        let
            tail = if Text.Length(s) > 10 then Text.Range(s, 10) else "",
            zoned = Text.EndsWith(Text.Upper(s), "Z") or Text.Contains(tail, "+") or Text.Contains(tail, "-"),
            value = if zoned then null else try DateTime.FromText(s, [Culture = "en-US"]) otherwise null
        in value,
    DateTables = (t as table) as table => Table.Buffer(Table.TransformColumns(Table.SelectColumns(t, {"Last_Updated", "ConfirmedInArcher"}),
        {{"Last_Updated", AsDate, type nullable datetime}, {"ConfirmedInArcher", AsDate, type nullable datetime}})),
    LatestRow = (label as text, a as table, b as table) as list =>
        let
            parsedA = List.RemoveNulls(Table.Column(a, label)), parsedB = List.RemoveNulls(Table.Column(b, label)),
            badA = List.NonNullCount(Table.Column(SqlData, label)) - List.Count(parsedA),
            badB = List.NonNullCount(Table.Column(SfData, label)) - List.Count(parsedB),
            fmt = (v as list) as nullable text => if List.IsEmpty(v) then null else DateTime.ToText(List.Max(v), [Format = "yyyy-MM-dd HH:mm:ss.fffffff", Culture = "en-US"]),
            ready = not List.IsEmpty(parsedA) and not List.IsEmpty(parsedB) and badA + badB = 0
        in Row("Latest " & label, fmt(parsedA), fmt(parsedB), null, null, if ready then "Info" else "Review",
            "Latest readable value across each full file. Unreadable populated dates: SQL " & Num(badA) & ", Snowflake " & Num(badB) & "."),
    SqlDates = DateTables(SqlData), SfDates = DateTables(SfData),

    CompareField = (f as record) as list =>
        let
            name = f[SQL], sfName = "SF." & name,
            columns = Table.RenameColumns(Table.SelectColumns(Pairs, {"Content_ID", name, sfName}), {{name, "A"}, {sfName, "B"}}),
            judged = Table.Buffer(Table.AddColumn(columns, "Result", each
                let a = [A], b = [B] in
                if a = null and b = null then "Both blank"
                else if a = null or b = null then "Different"
                else if f[Rule] = "Date" then
                    let da = AsDate(a), db = AsDate(b)
                    in if da = null or db = null then "Unreadable" else if da = db then "Same" else "Different"
                else if f[Rule] = "Package names" then
                    if PackageNames(a) = PackageNames(b) then "Same" else "Different"
                else if a = b then "Same" else "Different", type text)),
            issues = Table.SelectRows(judged, each [Result] = "Different" or [Result] = "Unreadable"),
            mismatches = Table.RowCount(Table.SelectRows(issues, each [Result] = "Different")),
            unreadable = Table.RowCount(issues) - mismatches,
            popA = List.NonNullCount(judged[A]), popB = List.NonNullCount(judged[B]),
            ready = HasKey and HasField(f) and NPairs > 0,
            note = if not HasField(f) then "Required header missing. See the field-header check."
                else if not HasKey or NPairs = 0 then "No unique shared Content_IDs available."
                else if popA + popB = 0 then "Every matched row is blank on both sides. Confirm whether this is expected."
                else (if f[Rule] = "Package names" then "SQL package vs. Snowflake resolved names. " else "")
                    & (if unreadable > 0 then Num(unreadable) & " date pairs also need review. " else "")
                    & (if Table.IsEmpty(issues) then "Populated values match." else "Example Content_IDs: " & Sample(issues[Content_ID]) & "."),
            status = if not ready then "Review" else if mismatches > 0 then "Fail" else if unreadable > 0 or popA + popB = 0 then "Review" else "Pass"
        in Row(name, if ready then popA else null, if ready then popB else null, if ready then NPairs else null,
            if ready then mismatches else null, status, note),
    PrefixIssues = (t as table) as number => Table.RowCount(Table.SelectRows(t, each [Hardware_ID] <> null and
        (not Text.StartsWith([Hardware_ID], "HW-") or Text.StartsWith([Hardware_ID], "HW-HW-")))),
    PrefixSql = PrefixIssues(SqlData), PrefixSf = PrefixIssues(SfData),
    Prefix = Row("Hardware_ID prefix exceptions", if HasHardware then PrefixSql else null, if HasHardware then PrefixSf else null, null, null,
        if not HasHardware or not BothHaveRows or PrefixSql + PrefixSf > 0 then "Review" else "Info",
        "Populated IDs missing the HW- prefix or starting HW-HW-. Confirm the expected format if any appear."),
    Values = List.Transform(Table.ToRecords(Table.SelectRows(Fields, each [Rule] <> "Key")), CompareField),
    Checked = List.Buffer(Core & {Prefix, LatestRow("Last_Updated", SqlDates, SfDates), LatestRow("ConfirmedInArcher", SqlDates, SfDates)} & Values),
    Result = if List.AnyTrue(List.Transform(Checked, each _{5} = "Fail")) then "Fail"
        else if List.AnyTrue(List.Transform(Checked, each _{5} = "Review")) then "Review" else "Pass",
    OutsideExports = {
        Row("HW01 - refresh timing", null, null, null, null, "Check separately", "Use both source cutoffs and the run log to confirm Matillion finished before C2S started."),
        Row("HW11 - derived host names", null, null, null, null, "Check separately", "Verify Archer_Asset_Name and HostNameFirstPartHW in their downstream C2S views."),
        Row("HW08 / HW12 - C2S outputs and package joins", null, null, null, null, "Check separately", "Verify package filters and joins, including hardware linked to multiple packages. Check the affected C2S views and dimHardware.")
    },
    Summary = #table({"Check", "SQL Server", "Snowflake", "Compared", "Differences", "Status", "Notes or example Content_IDs"},
        {Row("Hardware export comparison", null, null, null, null, Result, "Field rows show populated values in the matched records. See the last three rows for downstream sign-off.")} & Checked & OutsideExports),

    // Optional: duplicate this query and change the final word Summary to UnmatchedIDs for the complete list.
    UnmatchedIDs = if not HasKey then error "Both Content_ID headers are required. See Summary." else Table.Sort(Table.Combine({
        Table.AddColumn(Table.SelectColumns(SqlOnly, {"Content_ID"}), "Found in", each "SQL Server only", type text),
        Table.AddColumn(Table.RenameColumns(Table.SelectColumns(SfOnly, {"SfID"}), {{"SfID", "Content_ID"}}), "Found in", each "Snowflake only", type text)
    }), {{"Found in", Order.Ascending}, {"Content_ID", Order.Ascending}})
in
    Summary
