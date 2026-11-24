A. SQL Server — run this SELECT in SSMS

-- Full Global Trade population, one row per source row
;WITH src AS (
SELECT
COALESCE(CONVERT(nvarchar(max), CONTENT_ID), '') AS CONTENT_ID,
NULLIF(TRIM(CONVERT(nvarchar(max), LAST_UPDATED, 126)), '') AS LAST_UPDATED,
NULLIF(TRIM(CONVERT(nvarchar(max), CREATED_DATE, 126)), '') AS CREATED_DATE,
COALESCE(CONVERT(nvarchar(max), GLOBAL_TRADE_ASSESSMENT_ID), '') AS GLOBAL_TRADE_ASSESSMENT_ID,
COALESCE(CONVERT(nvarchar(max), RESPONDENT_STATUS), '') AS RESPONDENT_STATUS,
COALESCE(CONVERT(nvarchar(max), SYSTEM_TYPE_DECISION_TREE_31), '') AS SYSTEM_TYPE_DECISION_TREE_31,
COALESCE(CONVERT(nvarchar(max), SYSTEM_TYPE_DECISION_TREE_38), '') AS SYSTEM_TYPE_DECISION_TREE_38,
COALESCE(CONVERT(nvarchar(max), SYSTEM_TYPE_DECISION_TREE_39), '') AS SYSTEM_TYPE_DECISION_TREE_39,
COALESCE(CONVERT(nvarchar(max), IDENTIFYING_INFORMATION_11), '') AS IDENTIFYING_INFORMATION_11,
COALESCE(CONVERT(nvarchar(max), ASSESSMENT_TYPE), '') AS ASSESSMENT_TYPE,
COALESCE(CONVERT(nvarchar(max), AUTHORIZATION_PACKAGE), '') AS AUTHORIZATION_PACKAGE,
NULLIF(TRIM(CONVERT(nvarchar(max), CONFIRMEDINARCHER, 126)), '') AS CONFIRMEDINARCHER,
NULLIF(TRIM(CONVERT(nvarchar(max), APPROVAL_DATE, 126)), '') AS APPROVAL_DATE,
COALESCE(CONVERT(nvarchar(max), SECOND_LEVEL_REVIEWER_COLLABORATOR_NAME), '') AS SECOND_LEVEL_REVIEWER_COLLABORATOR_NAME,
COALESCE(CONVERT(nvarchar(max), SECOND_LEVEL_REVIEWER_COLLABORATOR_UID), '') AS SECOND_LEVEL_REVIEWER_COLLABORATOR_UID,
COALESCE(CONVERT(nvarchar(max), FIRST_LEVEL_REVIEWER_COLLABORATOR_NAME), '') AS FIRST_LEVEL_REVIEWER_COLLABORATOR_NAME,
COALESCE(CONVERT(nvarchar(max), FIRST_LEVEL_REVIEWER_COLLABORATOR_UID), '') AS FIRST_LEVEL_REVIEWER_COLLABORATOR_UID
FROM ACP_Connection.dbo.ArcherMetrics_GlobalTrade_PROD
)
SELECT
CONTENT_ID,
CASE WHEN LAST_UPDATED IS NULL THEN ''
ELSE COALESCE(CONVERT(char(10), COALESCE(TRY_CONVERT(datetime2, LAST_UPDATED, 126), TRY_CONVERT(datetime2, LAST_UPDATED, 121), TRY_CONVERT(datetime2, LAST_UPDATED, 101)), 23), '!INVALID_DATE!') END AS LAST_UPDATED,
CASE WHEN CREATED_DATE IS NULL THEN ''
ELSE COALESCE(CONVERT(char(10), COALESCE(TRY_CONVERT(datetime2, CREATED_DATE, 126), TRY_CONVERT(datetime2, CREATED_DATE, 121), TRY_CONVERT(datetime2, CREATED_DATE, 101)), 23), '!INVALID_DATE!') END AS CREATED_DATE,
GLOBAL_TRADE_ASSESSMENT_ID,
RESPONDENT_STATUS,
SYSTEM_TYPE_DECISION_TREE_31,
SYSTEM_TYPE_DECISION_TREE_38,
SYSTEM_TYPE_DECISION_TREE_39,
IDENTIFYING_INFORMATION_11,
ASSESSMENT_TYPE,
AUTHORIZATION_PACKAGE,
CASE WHEN CONFIRMEDINARCHER IS NULL THEN ''
ELSE COALESCE(CONVERT(char(10), COALESCE(TRY_CONVERT(datetime2, CONFIRMEDINARCHER, 126), TRY_CONVERT(datetime2, CONFIRMEDINARCHER, 121), TRY_CONVERT(datetime2, CONFIRMEDINARCHER, 101)), 23), '!INVALID_DATE!') END AS CONFIRMEDINARCHER,
CASE WHEN APPROVAL_DATE IS NULL THEN ''
ELSE COALESCE(CONVERT(char(10), COALESCE(TRY_CONVERT(datetime2, APPROVAL_DATE, 126), TRY_CONVERT(datetime2, APPROVAL_DATE, 121), TRY_CONVERT(datetime2, APPROVAL_DATE, 101)), 23), '!INVALID_DATE!') END AS APPROVAL_DATE,
SECOND_LEVEL_REVIEWER_COLLABORATOR_NAME,
SECOND_LEVEL_REVIEWER_COLLABORATOR_UID,
FIRST_LEVEL_REVIEWER_COLLABORATOR_NAME,
FIRST_LEVEL_REVIEWER_COLLABORATOR_UID,
COUNT(*) OVER () AS SOURCE_ROWS
FROM src;

B. Snowflake — run this SELECT against the published view

-- Full Global Trade population, one row per source row
WITH src AS (
SELECT
COALESCE(TO_VARCHAR(ARCHER_CONTENT_GLOBAL_TRADE_ASSESSMENT_CONTENT_ID), '') AS CONTENT_ID,
NULLIF(TRIM(TO_VARCHAR(ARCHER_CONTENT_GLOBAL_TRADE_ASSESSMENT_LAST_UPDATED)), '') AS LAST_UPDATED,
NULLIF(TRIM(TO_VARCHAR(ARCHER_CONTENT_GLOBAL_TRADE_ASSESSMENT_FIRST_PUBLISHED)), '') AS CREATED_DATE,
COALESCE(TO_VARCHAR(GLOBAL_TRADE_ASSESSMENT_ID), '') AS GLOBAL_TRADE_ASSESSMENT_ID,
COALESCE(TO_VARCHAR(RESPONDENT_STATUS), '') AS RESPONDENT_STATUS,
COALESCE(TO_VARCHAR(SYSTEM_TYPE_DECISION_TREE_31), '') AS SYSTEM_TYPE_DECISION_TREE_31,
COALESCE(TO_VARCHAR(SYSTEM_TYPE_DECISION_TREE_38), '') AS SYSTEM_TYPE_DECISION_TREE_38,
COALESCE(TO_VARCHAR(SYSTEM_TYPE_DECISION_TREE_39), '') AS SYSTEM_TYPE_DECISION_TREE_39,
COALESCE(TO_VARCHAR(IDENTIFYING_INFORMATION_11), '') AS IDENTIFYING_INFORMATION_11,
COALESCE(TO_VARCHAR(ASSESSMENT_TYPE), '') AS ASSESSMENT_TYPE,
COALESCE(TO_VARCHAR(AUTHORIZATION_PACKAGE_NAME), '') AS AUTHORIZATION_PACKAGE,
NULLIF(TRIM(TO_VARCHAR(ARCHER_CONTENT_GLOBAL_TRADE_ASSESSMENT_CONFIRMED_IN_ARCHER)), '') AS CONFIRMEDINARCHER,
NULLIF(TRIM(TO_VARCHAR(APPROVAL_DATE)), '') AS APPROVAL_DATE,
COALESCE(TO_VARCHAR(SECOND_LEVEL_REVIEWER_COLLABORATOR_ONLY), '') AS SECOND_LEVEL_REVIEWER_COLLABORATOR_NAME,
COALESCE(TO_VARCHAR(SECOND_LEVEL_REVIEWER_COLLABORATOR_ID_ONLY), '') AS SECOND_LEVEL_REVIEWER_COLLABORATOR_UID,
COALESCE(TO_VARCHAR(FIRST_LEVEL_REVIEWER_COLLABORATOR_NAME_ONLY), '') AS FIRST_LEVEL_REVIEWER_COLLABORATOR_NAME,
COALESCE(TO_VARCHAR(FIRST_LEVEL_REVIEWER_COLLABORATOR_ID_ONLY), '') AS FIRST_LEVEL_REVIEWER_COLLABORATOR_UID,
COALESCE(TO_VARCHAR(AUTHORIZATION_PACKAGE), '') AS RAW_PACKAGE
FROM RTX_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_GLOBAL_TRADE
)
SELECT
CONTENT_ID,
CASE WHEN LAST_UPDATED IS NULL THEN ''
ELSE COALESCE(TO_CHAR(TRY_TO_TIMESTAMP_NTZ(LAST_UPDATED), 'YYYY-MM-DD'), '!INVALID_DATE!') END AS LAST_UPDATED,
CASE WHEN CREATED_DATE IS NULL THEN ''
ELSE COALESCE(TO_CHAR(TRY_TO_TIMESTAMP_NTZ(CREATED_DATE), 'YYYY-MM-DD'), '!INVALID_DATE!') END AS CREATED_DATE,
GLOBAL_TRADE_ASSESSMENT_ID,
RESPONDENT_STATUS,
SYSTEM_TYPE_DECISION_TREE_31,
SYSTEM_TYPE_DECISION_TREE_38,
SYSTEM_TYPE_DECISION_TREE_39,
IDENTIFYING_INFORMATION_11,
ASSESSMENT_TYPE,
AUTHORIZATION_PACKAGE,
CASE WHEN CONFIRMEDINARCHER IS NULL THEN ''
ELSE COALESCE(TO_CHAR(TRY_TO_TIMESTAMP_NTZ(CONFIRMEDINARCHER), 'YYYY-MM-DD'), '!INVALID_DATE!') END AS CONFIRMEDINARCHER,
CASE WHEN APPROVAL_DATE IS NULL THEN ''
ELSE COALESCE(TO_CHAR(TRY_TO_TIMESTAMP_NTZ(APPROVAL_DATE), 'YYYY-MM-DD'), '!INVALID_DATE!') END AS APPROVAL_DATE,
SECOND_LEVEL_REVIEWER_COLLABORATOR_NAME,
SECOND_LEVEL_REVIEWER_COLLABORATOR_UID,
FIRST_LEVEL_REVIEWER_COLLABORATOR_NAME,
FIRST_LEVEL_REVIEWER_COLLABORATOR_UID,
RAW_PACKAGE,
COUNT(*) OVER () AS SOURCE_ROWS
FROM src;

C. Power Query — paste from the next line through the end of the file

let
SQLPath = "C:\GT\GT_SQL.xlsx",
SnowflakePath = "C:\GT\GT_SF.csv",

Dates = {"LAST_UPDATED", "CONFIRMEDINARCHER", "CREATED_DATE", "APPROVAL_DATE"},
Names = {"FIRST_LEVEL_REVIEWER_COLLABORATOR_NAME", "SECOND_LEVEL_REVIEWER_COLLABORATOR_NAME"},
UIDs = {"FIRST_LEVEL_REVIEWER_COLLABORATOR_UID", "SECOND_LEVEL_REVIEWER_COLLABORATOR_UID"},
Fields = List.Buffer({
    "GLOBAL_TRADE_ASSESSMENT_ID", "RESPONDENT_STATUS", "ASSESSMENT_TYPE",
    "SYSTEM_TYPE_DECISION_TREE_31", "SYSTEM_TYPE_DECISION_TREE_38", "SYSTEM_TYPE_DECISION_TREE_39",
    "IDENTIFYING_INFORMATION_11", "AUTHORIZATION_PACKAGE", "LAST_UPDATED", "CONFIRMEDINARCHER",
    "CREATED_DATE", "APPROVAL_DATE"
} & Names & UIDs),
Key = "CONTENT_ID",
CountText = (n as number) as text => Number.ToText(n, "#,##0", "en-US"),
Clean = (x as any) as nullable text =>
    if x = null then null else
    let t = if Value.Is(x, type datetime) or Value.Is(x, type date)
            then Date.ToText(Date.From(x),[Format="yyyy-MM-dd",Culture="en-US"])
            else Text.Trim(Text.From(x, "en-US"))
    in if t = "" then null else t,

ReadFile = (path as text, extras as list) as table =>
    let
        expected = {Key, "SOURCE_ROWS"} & Fields & extras,
        headers = (raw as table) as table =>
            Table.TransformColumnNames(Table.PromoteHeaders(raw,[PromoteAllScalars=true]),
                                       each Text.Upper(Text.Trim(_), "en-US")),
        workbook = () as table =>
            let sheets = Table.SelectRows(Excel.Workbook(File.Contents(path),null,true), each [Kind] = "Sheet" and [Hidden] <> true),
                candidates = List.Select(List.Transform(sheets[Data],headers),
                    (t) => List.ContainsAll(Table.ColumnNames(t),expected))
            in if List.Count(candidates) = 1 then candidates{0}
               else error ("Use one visible worksheet with all source result headers in row 1: " & path),
        csv = Csv.Document(File.Contents(path), [Delimiter=",", Columns=List.Count(expected),
                Encoding=65001, QuoteStyle=QuoteStyle.Csv, ExtraValues=ExtraValues.Error]),
        imported = if Text.EndsWith(path,".xlsx",Comparer.OrdinalIgnoreCase) then workbook() else headers(csv),
        selected = Table.SelectColumns(imported, expected),
        cleaned = Table.Buffer(Table.TransformColumns(selected,
                    List.Transform(expected, each {_, Clean, type nullable text}))),
        actual = Table.RowCount(cleaned),
        declared = List.Distinct(cleaned[SOURCE_ROWS]),
        checked = if actual > 0 and List.Count(declared) = 1 and
                     (try Number.FromText(declared{0}, "en-US") otherwise null) = actual
                  then Table.RemoveColumns(cleaned, {"SOURCE_ROWS"})
                  else error ("Incomplete extract or invalid SOURCE_ROWS. Export the full result with headers: " & path)
    in checked,

Sql = ReadFile(SQLPath, {}),
Sf = ReadFile(SnowflakePath, {"RAW_PACKAGE"}),
Dur = (t as table, col as text) as list => List.RemoveNulls(Table.Column(t, col)),
Profiles = (t as table, col as text) as record =>
    let populated = List.Buffer(Dur(t,col)), unique = List.Count(List.Distinct(populated))
    in [Distinct = unique, Missing = Table.RowCount(t)-List.Count(populated), Repeats = List.Count(populated)-unique],
KeyGroups = (t as table) as table =>
    Table.Buffer(Table.Group(Table.SelectRows(t, each [CONTENT_ID] <> null), {Key},
        {{"Uses", each Table.RowCount(_), Int64.Type},
         {"Data", each Table.First(_), type record}})),
L = KeyGroups(Sql),
R = KeyGroups(Sf),
LeftOnly = Table.RowCount(Table.NestedJoin(L, {Key}, R, {Key}, "Other", JoinKind.LeftAnti)),
RightOnly = Table.RowCount(Table.NestedJoin(R, {Key}, L, {Key}, "Other", JoinKind.LeftAnti)),
SharedGroups = Table.ExpandTableColumn(Table.NestedJoin(L, {Key}, R, {Key}, "Sf", JoinKind.Inner),
                                      "Sf", {"Uses","Data"}, {"SfUses","SfData"}),
UniquePairs = Table.SelectRows(SharedGroups, each [Uses] = 1 and [SfUses] = 1),
Shared = Table.RowCount(L)-LeftOnly,

// Each row is a unique Content_ID present in both sources. Equality is field-specific.
Parts = (t as nullable text) as list =>
    if t = null then {} else List.Transform(Text.Split(t,";"), Text.Trim),
Ordered = (xs as list) as list => List.Sort(xs, Comparer.Ordinal),
NonemptyCount = (t as nullable text) as number => List.Count(List.Select(Parts(t), each _ <> "")),
ISODate = (s as nullable text) as nullable date =>
    if s = null then null else try Date.FromText(s,[Format="yyyy-MM-dd",Culture="en-US"]) otherwise null,

// 0 both blank, 1 populated match, 2 name case only, 3 SQL blank, 4 Snowflake blank, 5 different, 6 invalid date.
Score = (f as text, x as nullable text, y as nullable text) as number =>
    if x = null and y = null then 0
    else if List.Contains(Dates, f) and
            ((x <> null and ISODate(x) = null) or (y <> null and ISODate(y) = null)) then 6
    else if x = null then 3
    else if y = null then 4
    else if x = y then 1
    else if List.Contains(Names & UIDs, f) then
        let a = Parts(x), b = Parts(y), upper = (v as text) as text => Text.Upper(v, "en-US")
        in if Ordered(a) = Ordered(b) then 1
           else if List.Contains(Names, f) and Ordered(List.Transform(a, upper)) = Ordered(List.Transform(b, upper)) then 2
           else 5
    else 5,
ScoreRows = List.Transform(Table.ToRecords(UniquePairs), (p) =>
                List.Transform(Fields, (f) => Score(f, Record.Field(p[Data], f), Record.Field(p[SfData], f)))),
Scores = Table.Buffer(Table.FromRows(ScoreRows, Fields)),
Compared = Table.RowCount(Scores),
Overall = List.Buffer(List.Transform(Table.ToRows(Scores), (s) =>
            if List.ContainsAny(s,{3,4,5}) then "Difference"
            else if List.Contains(s,6) then "Invalid"
            else if List.AllTrue(List.Transform(s, each _ = 0)) then "Empty"
            else "Agree")),
TimingFields = {"LAST_UPDATED", "CONFIRMEDINARCHER"},
BusinessFields = List.RemoveItems(Fields, TimingFields),
TimingOnly = Table.RowCount(Table.SelectRows(Scores, (r) =>
    let other = List.Transform(BusinessFields, (f) => Record.Field(r, f)),
        timing = List.Transform(TimingFields, (f) => Record.Field(r, f))
    in not List.ContainsAny(other, {3,4,5,6}) and not List.Contains(timing,6)
       and List.ContainsAny(timing,{3,4,5}))),
Frequency = (xs as list, v as any) as number => List.Count(List.Select(xs, each _ = v)),
Evidence = (name as text, value as number) as text => name & " " & CountText(value),
SourcePair = (left as number, right as number) as text =>
    Evidence("SQL",left) & " | " & Evidence("Snowflake",right),
SourceStats = List.Combine(List.Transform({Key, "GLOBAL_TRADE_ASSESSMENT_ID"}, (f) =>
    let a = Profiles(Sql,f), b = Profiles(Sf,f) in {
        {"Keys", f & " missing", SourcePair(a[Missing],b[Missing])},
        {"Keys", f & " extra duplicate rows", SourcePair(a[Repeats],b[Repeats])}
    })),
FieldResult = (f as text) as list =>
    let s = List.Buffer(Table.Column(Scores, f)),
        labels = {"Both blank","Populated match","Name case only","SQL blank","Snowflake blank","Different","Invalid date"},
        counts = List.Transform({0..6}, each Frequency(s, _)),
        nonzero = List.Select({0..6}, (i) => counts{i} > 0)
    in {"Field", f, if Compared = 0 then "No eligible shared IDs"
                    else Text.Combine(List.Transform(nonzero, (i) => Evidence(labels{i},counts{i})), " | ")},
Latest = (t as table, f as text) as text =>
    let inputs = Dur(t,f), days = List.RemoveNulls(List.Transform(inputs,ISODate)),
        d = List.Max(days,null),
        shown = if d = null then "no readable date" else Date.ToText(d,[Format="yyyy-MM-dd",Culture="en-US"])
    in shown & " (" & Evidence("invalid",List.Count(inputs)-List.Count(days)) & ")",
Links = Table.Buffer(Table.FromRows(List.Transform(Table.ToRecords(Table.SelectColumns(Sf,{"RAW_PACKAGE","AUTHORIZATION_PACKAGE"})),
        (r) => {NonemptyCount(r[RAW_PACKAGE]), NonemptyCount(r[AUTHORIZATION_PACKAGE])}), {"Refs","Resolved"})),
Multi = Table.SelectRows(Links, each [Refs] > 1),
Rows = {
    {"Sources","Assessment rows", SourcePair(Table.RowCount(Sql),Table.RowCount(Sf))},
    {"Coverage","Shared Content_IDs",CountText(Shared)},
    {"Coverage","SQL-only Content_IDs",CountText(LeftOnly)},
    {"Coverage","Snowflake-only Content_IDs",CountText(RightOnly)},
    {"Coverage","Total IDs present in one source",CountText(LeftOnly+RightOnly)},
    {"Coverage","Distinct IDs across both sources",CountText(Shared+LeftOnly+RightOnly)}
} & SourceStats & {
    {"Compared records","Shared IDs unique on both sides",CountText(Compared)},
    {"Compared records","Shared IDs excluded due to repetition",CountText(Shared-Compared)},
    {"Compared records","All 16 fields agree under comparison rules",CountText(Frequency(Overall,"Agree"))},
    {"Compared records","At least one field differs or is blank on one side",CountText(Frequency(Overall,"Difference"))},
    {"Compared records","Unresolved only because of an invalid date",CountText(Frequency(Overall,"Invalid"))},
    {"Compared records","All tested values blank in both sources",CountText(Frequency(Overall,"Empty"))},
    {"Difference detail","Of the assessments with differences: only update or confirmation differs",CountText(TimingOnly)}
} & List.Transform(Fields, FieldResult) & {
    {"Package links","Snowflake rows with references and zero resolved names",CountText(Table.RowCount(Table.SelectRows(Links, each [Refs] > 0 and [Resolved] = 0)))},
    {"Package links","Snowflake rows with multiple references",CountText(Table.RowCount(Multi))},
    {"Package links","Multi-reference rows with a different number of names",CountText(Table.RowCount(Table.SelectRows(Multi, each [Refs] <> [Resolved])))}
} & List.Transform({"LAST_UPDATED","CONFIRMEDINARCHER"}, (f) =>
    {"Dates","Latest " & f,"SQL " & Latest(Sql,f) & " | Snowflake " & Latest(Sf,f)}),
Result = #table(type table [Section=text, Check=text, Evidence=text], Rows)

in
Result
