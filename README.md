Check	Test ID	Result	Evidence	Conclusion or action	Source
Required fields available	AP02	✓ Pass	All 68 SQL fields have a matching column in the Snowflake view.	The original query skipped value checks for nine of these fields. Their columns were confirmed in the view.	Nov 19 view review
Total packages	AP03a	✗ Fail	SQL Server has 4,142 packages. Snowflake has 4,872.	Snowflake has 730 more packages overall.	Nov 20 exports
Same packages in both files	AP03b	✗ Fail	4,113 packages appear in both files. 29 appear only in SQL Server. 759 appear only in Snowflake.	Confirm which packages should be included using the same filters and data cutoff.	Nov 20 exports
Missing IDs	AP04	✓ Pass	Zero missing Content_ID or SAP_ID_RTN values in either file.	Every package has both IDs.	Nov 20 exports
Duplicate IDs	AP05	✓ Pass	Zero repeated Content_ID or SAP_ID_RTN values in either file.	Each ID appears once in its file.	Nov 20 exports
SAP_ID_RTN matches	AP06	✓ Pass	All 4,113 packages found in both files have the same SAP_ID_RTN.	This check passed for every shared package.	Nov 20 exports
Continuous monitoring	AP07	✗ Fail	Of the 4,113 shared packages, 1,279 have a monitoring value in SQL Server. None have a value in Snowflake.	Trace a package with a SQL value and confirm why the Snowflake value is empty.	Nov 20 exports
Additional Visibility source	AP11	✓ Pass	The updated Snowflake view reads the package visibility field and uses the visibility lookup for names and IDs.	Grady’s mapping fix is confirmed and closed.	Nov 19 view definition
SAP_ID matches	AP06b	✓ Pass	All 4,113 packages found in both files have the same SAP_ID.	This check passed for every shared package.	Nov 20 exports
Authorization Package UAT	AP13	✗ Fail	November 20 comparison complete.	Sign-off is on hold for the package differences and missing monitoring values. Hardware testing can proceed.	Nov 20 UAT




// In the UAT workbook, fill in the two paths in UATFiles. Paste this whole file
// into the Advanced Editor of one blank Excel Power Query, then load and refresh.
// AuthorizationFields supplies the column mappings. UATChecks selects value checks.
let
Workbook = Excel.CurrentWorkbook(),
Config = (name as text) as table => Workbook{[Name = name]}[Content],
Header = (value as any) as text =>
if value = null then "" else Text.Upper(Text.Trim(Text.Replace(Text.From(value), Character.FromNumber(65279), "")), "en-US"),
Clean = (value as any) as nullable text =>
let s = if value = null then "" else Text.Trim(Text.From(value))
in if s = "" or List.Contains({"NULL", "\N"}, Text.Upper(s, "en-US")) then null else s,
Files = Config("UATFiles"),
FileRow = if Table.RowCount(Files) = 1 then Files{0} else error "Use one row in the UATFiles table.",
Load = (path as nullable text) as table =>
let
cleanPath = if path = null then "" else Text.Trim(Text.Trim(path), """"),
bytes = if cleanPath = "" then error "Fill in both file paths in UATFiles." else Binary.Buffer(File.Contents(cleanPath)),
bom = if Binary.Length(bytes) < 2 then "" else Text.Upper(Binary.ToText(Binary.Range(bytes, 0, 2), BinaryEncoding.Hex)),
encoding = if bom = "FFFE" then 1200 else if bom = "FEFF" then 1201 else
let utf8 = try Text.FromBinary(bytes, 65001)
in if utf8[HasError] then 1252 else if Text.Contains(utf8[Value], Character.FromNumber(65533)) then 1252 else 65001,
csv = Csv.Document(bytes, [Delimiter = ",", Encoding = encoding, QuoteStyle = QuoteStyle.Csv]),
headers = if Table.IsEmpty(csv) then error "The CSV is empty. Export with column headers."
else List.Transform(Record.FieldValues(csv{0}), Header),
valid = if List.Contains(headers, "") or List.Count(List.Distinct(headers)) <> List.Count(headers)
then error "The CSV has an empty or duplicate header. Export again with unique column headers." else headers
in
Table.Buffer(Table.FromColumns(Table.ToColumns(Table.Skip(csv, 1)), valid)),
Sql = Load(FileRow[SQLFile]),
Sf = Load(FileRow[SnowflakeFile]),
SqlHeaders = List.Buffer(Table.ColumnNames(Sql)),
SfHeaders = List.Buffer(Table.ColumnNames(Sf)),
MapRows = List.Transform(Table.ToRecords(Config("AuthorizationFields")),
each [Field = Header([SQL field]), Target = Header([Snowflake field or candidate])]),
Fields = List.Transform(MapRows, each [Field]),
Mapping = if List.IsEmpty(Fields) or List.Contains(Fields, "") or List.Count(Fields) <> List.Count(List.Distinct(Fields))
then error "AuthorizationFields must list each SQL field once, with a nonblank name."
else Record.FromList(List.Transform(MapRows, each [Target]), Fields),
Options = List.Transform(Table.ToRecords(Config("UATChecks")),
each [Field = Header([SQL field]), Mode = Header([Value check])]),
Allowed = {"PACKAGE KEY", "KEY", "TEXT", "WHOLE NUMBER", "PRESENCE"},
BadOptions = List.Select(Options, each not Record.HasFields(Mapping, [Field]) or not List.Contains(Allowed, [Mode])),
Selected = List.Transform(Options, each [Field]),
Rules = if not List.IsEmpty(BadOptions) then error "Check UATChecks for a field absent from AuthorizationFields or an unsupported Value check."
else if List.Count(Selected) <> List.Count(List.Distinct(Selected)) then error "Use one UATChecks row per SQL field."
else if List.Count(List.Select(Options, each [Mode] = "PACKAGE KEY")) <> 1 then error "Choose exactly one Package key in UATChecks."
else Options,
Keys = List.Select(Rules, each List.Contains({"PACKAGE KEY","KEY"}, [Mode])),
MajorFields = List.Select(Rules, each [Mode] <> "PACKAGE KEY"),
HasBoth = (field as text) as logical =>
List.Contains(SqlHeaders, field) and List.Contains(SfHeaders, Record.Field(Mapping, field)),
// Only the selected value columns are needed to profile keys and pair records.
Project = (source as table, isSql as logical) as table =>
let
rulesHere = List.Select(Rules, each HasBoth([Field])),
logicalNames = List.Transform(rulesHere, each [Field]),
values = List.Transform(logicalNames, (f) =>
List.Transform(Table.Column(source, if isSql then f else Record.Field(Mapping, f)), Clean))
in Table.Buffer(Table.FromColumns(values, logicalNames)),
SqlValues = Project(Sql, true),
SfValues = Project(Sf, false),
SqlTotal = Table.RowCount(Sql),
SfTotal = Table.RowCount(Sf),
HaveRows = SqlTotal > 0 and SfTotal > 0,
Result = (label as text, sql as nullable number, sf as nullable number, compared as nullable number,
diff as nullable number, tested as logical, note as text) as record =>
[Check = label, SQL = sql, Snowflake = sf, Compared = compared, Differences = diff,
Result = if not tested or diff = null then "— Not tested" else if diff = 0 then "✓ Pass" else "✗ Fail", Notes = note],
MissingSql = List.Select(Fields, each not List.Contains(SqlHeaders, )),
MissingSf = List.Select(Fields, each not List.Contains(SfHeaders, Record.Field(Mapping, ))),
MissingEither = List.Union({MissingSql, MissingSf}),
MissingNotes = Text.Combine(List.RemoveNulls({
if List.IsEmpty(MissingSql) then null else "SQL missing: " & Text.Combine(MissingSql, ", "),
if List.IsEmpty(MissingSf) then null else "Snowflake mapping missing for: " & Text.Combine(MissingSf, ", ")
}), " | "),
Schema = Result("Mapped fields present", List.Count(Fields)-List.Count(MissingSql),
List.Count(Fields)-List.Count(MissingSf), List.Count(Fields), List.Count(MissingEither), true,
if MissingNotes = "" then "All mapped fields are present in the exports." else MissingNotes),
Rows = Result("Export rows", SqlTotal, SfTotal, null, Number.Abs(SqlTotal-SfTotal),
SqlTotal+SfTotal > 0, "Counts every CSV data row, excluding the header."),
Groups = (data as table, field as text) as table =>
let
present = Table.SelectRows(data, each Record.Field(, field) <> null),
groups = Table.Group(present, {field}, {
{"Count", each Table.RowCount(), Int64.Type},
{"First", each Table.First(), type record}
})
in Table.Buffer(Table.RenameColumns(groups, {{field, "JoinKey"}})),
MakeProfile = (rule as record) as record =>
let
f = rule[Field],
left = Groups(SqlValues, f),
right = Groups(SfValues, f),
pairs = Table.NestedJoin(left, {"JoinKey"}, right, {"JoinKey"}, "RightSide", JoinKind.Inner),
both = Table.Buffer(Table.ExpandTableColumn(pairs, "RightSide", {"Count","First"}, {"RightCount","RightRecord"})),
s = Table.RowCount(left), t = Table.RowCount(right), shared = Table.RowCount(both),
sp = List.Sum({0} & left[Count]), tp = List.Sum({0} & right[Count]),
checks = {
Result(f & " / distinct populated IDs", s, t, shared, Number.Abs(s-t), HaveRows and s+t > 0, "Compared = shared distinct IDs. Differences = count gap."),
Result(f & " / missing ID rows", SqlTotal-sp, SfTotal-tp, null, SqlTotal-sp+SfTotal-tp, HaveRows, "Blank, whitespace, NULL and backslash-N count as missing."),
Result(f & " / excess duplicate rows", sp-s, tp-t, null, sp-s+tp-t, s > 0 and t > 0, "One excess row for each repeat after the first populated ID."),
Result(f & " / IDs on one side only", s-shared, t-shared, shared, s+t-2*shared, HaveRows and s+t > 0, "SQL-only and Snowflake-only counts. Differences = total unmatched distinct IDs.")
}
in if HasBoth(f) then [Field = f, Pairs = both, Checks = checks]
else [Field = f, Pairs = null, Checks = {Result(f & " / ID profile",null,null,null,null,false,"See mapped fields for the missing header.")}],
Profiles = List.Buffer(List.Transform(Keys, MakeProfile)),
PackageField = List.First(List.Select(Rules, each [Mode] = "PACKAGE KEY"))[Field],
PackageProfile = List.First(List.Select(Profiles, each [Field] = PackageField)),
Paired = if PackageProfile[Pairs] = null then {}
else List.Buffer(Table.ToRecords(Table.SelectRows(PackageProfile[Pairs], each [Count]=1 and [RightCount]=1))),
PairCount = List.Count(Paired),
// Normalize nonnegative whole counts as strings so large integers retain precision.
Whole = (s as text) as nullable text =>
let
unsigned = if Text.StartsWith(s,"+") then Text.Range(s,1) else s,
parts = Text.Split(unsigned,"."), groups = Text.Split(parts{0},","),
allDigits = (t as text) as logical => t <> "" and Text.Select(t, {"0".."9"}) = t,
groupingOK = List.Count(groups)=1 or (Text.Length(groups{0})<=3 and List.AllTrue(List.Transform(List.Skip(groups), each Text.Length()=3))),
intOK = groupingOK and List.AllTrue(List.Transform(groups,allDigits)),
fracOK = List.Count(parts)=1 or (List.Count(parts)=2 and parts{1} <> "" and Text.Trim(parts{1},"0")=""),
digits = Text.TrimStart(Text.Combine(groups,""),"0")
in if not intOK or not fracOK then null else if digits="" then "0" else digits,
Difference = (a as nullable text, b as nullable text, mode as text) as logical =>
if mode = "KEY" and (a = null or b = null) then true
else if a = null or b = null then a <> b
else if mode = "PRESENCE" then false
else if mode = "WHOLE NUMBER" then
let x = Whole(a), y = Whole(b) in x = null or y = null or x <> y
else if mode = "KEY" then a <> b
else Text.Upper(a,"en-US") <> Text.Upper(b,"en-US"),
CompareField = (rule as record) as record =>
let
f = rule[Field], mode = rule[Mode],
pairs = List.Buffer(List.Transform(Paired, each {Record.Field([First], f), Record.Field([RightRecord], f)})),
sp = List.Count(List.Select(pairs, each _{0} <> null)),
tp = List.Count(List.Select(pairs, each {1} <> null)),
d = List.Count(List.Select(pairs, each Difference({0}, _{1}, mode))),
tested = PairCount > 0 and (sp+tp > 0 or mode = "KEY"),
note = if PairCount = 0 then "No shared unique package IDs."
else if not tested then "Both exports are empty for this field on all paired rows."
else if mode = "PRESENCE" then "Checks populated versus missing on each paired package."
else if mode = "WHOLE NUMBER" then "Compares whole control counts, including one-sided blanks."
else if mode = "KEY" then "Compares populated identifiers exactly as text on paired packages."
else "Compares trimmed text ignoring case. Equivalent missing markers match."
in if not HasBoth(f) then Result(f & " / paired values",null,null,null,null,false,"Mapped header unavailable.")
else Result(f & " / paired values",sp,tp,PairCount,if tested then d else null,tested,note),
Results = {Schema, Rows} & List.Combine(List.Transform(Profiles, each [Checks])) & List.Transform(MajorFields, CompareField)
in
Table.FromRecords(Results, type table [Check=text, SQL=nullable number, Snowflake=nullable number,
Compared=nullable number, Differences=nullable number, Result=text, Notes=text])
