let
// Change just these two paths to your full, comma-separated exports with headers.
SQLPath = "C:\Exports\SQL Server Global Trade.csv",
SnowflakePath = "C:\Exports\Snowflake Global Trade.csv",

ReadExport = (path as text, optional encodingOverride as nullable number) as table =>
    let
        bytes = Binary.Buffer(File.Contents(path)),
        prefix = Text.Upper(Binary.ToText(Binary.Range(bytes, 0, List.Min({3, Binary.Length(bytes)})), BinaryEncoding.Hex)),
        bomEncoding =
            if Text.StartsWith(prefix, "EFBBBF") then 65001
            else if Text.StartsWith(prefix, "FFFE") then 1200
            else if Text.StartsWith(prefix, "FEFF") then 1201
            else null,
        payload = if bomEncoding = null then bytes else Binary.Range(bytes, if bomEncoding = 65001 then 3 else 2),
        encoding =
            if encodingOverride <> null then encodingOverride
            else if bomEncoding <> null then bomEncoding
            else
                let
                    utf8 = try Text.FromBinary(payload, 65001)
                in
                    if utf8[HasError] then 1252
                    else if not Text.Contains(utf8[Value], Character.FromNumber(65533)) then 65001
                    else if Text.ToBinary(utf8[Value], 65001, false) = payload then 65001
                    else 1252,
        options = [Delimiter = ",", Encoding = encoding, QuoteStyle = QuoteStyle.Csv],
        first = Table.First(Csv.Document(payload, options), null),
        headers = if first = null then error ("Empty CSV: " & path)
            else List.Transform(Record.FieldValues(first), each if _ = null then "" else Text.Upper(Text.Trim(_), "en-US")),
        checkedHeaders =
            if List.Count(headers) < 2 then error ("Expected a comma-separated export with headers: " & path)
            else if List.Contains(headers, "") then error ("Blank CSV header or inconsistent row width: " & path)
            else if List.Count(List.Distinct(headers)) <> List.Count(headers) then error ("Duplicate CSV headers after trimming and uppercasing: " & path)
            else headers,
        data = Csv.Document(payload, options & [Columns = checkedHeaders, ExtraValues = ExtraValues.Error])
    in
        Table.Skip(data, 1),

FieldMap = #table({"SQL", "Snowflake"}, {
    {"CONTENT_ID", "ARCHER_CONTENT_GLOBAL_TRADE_ASSESSMENT_CONTENT_ID"},
    {"LAST_UPDATED", "ARCHER_CONTENT_GLOBAL_TRADE_ASSESSMENT_LAST_UPDATED"},
    {"CREATED_DATE", "ARCHER_CONTENT_GLOBAL_TRADE_ASSESSMENT_FIRST_PUBLISHED"},
    {"GLOBAL_TRADE_ASSESSMENT_ID", "GLOBAL_TRADE_ASSESSMENT_ID"},
    {"RESPONDENT_STATUS", "RESPONDENT_STATUS"},
    {"WORKFLOW_CURRENT_NODE", "WORKFLOW_CURRENT_NODE"},
    {"WORKFLOW_PROCESS_VERSION", "WORKFLOW_PROCESS_VERSION"},
    {"WORKFLOW_JOB_STATUS", "WORKFLOW_JOB_STATUS"},
    {"BACKGROUND_INFORMATION_AND_PRIOR_GLOBAL_TRADE_REVIEWS_23", "BACKGROUND_INFORMATION_AND_PRIOR_GLOBAL_TRADE_REVIEWS_23"},
    {"BACKGROUND_INFORMATION_AND_PRIOR_GLOBAL_TRADE_REVIEWS_24", "BACKGROUND_INFORMATION_AND_PRIOR_GLOBAL_TRADE_REVIEWS_24"},
    {"BACKGROUND_INFORMATION_AND_PRIOR_GLOBAL_TRADE_REVIEWS_25", "BACKGROUND_INFORMATION_AND_PRIOR_GLOBAL_TRADE_REVIEWS_25"},
    {"BACKGROUND_INFORMATION_AND_PRIOR_GLOBAL_TRADE_REVIEWS_26", "BACKGROUND_INFORMATION_AND_PRIOR_GLOBAL_TRADE_REVIEWS_26"},
    {"BACKGROUND_INFORMATION_AND_PRIOR_GLOBAL_TRADE_REVIEWS_21", "BACKGROUND_INFORMATION_AND_PRIOR_GLOBAL_TRADE_REVIEWS_21"},
    {"BACKGROUND_INFORMATION_AND_PRIOR_GLOBAL_TRADE_REVIEWS_22", "BACKGROUND_INFORMATION_AND_PRIOR_GLOBAL_TRADE_REVIEWS_22"},
    {"SYSTEM_TYPE_DECISION_TREE_31", "SYSTEM_TYPE_DECISION_TREE_31"},
    {"SYSTEM_TYPE_DECISION_TREE_32", "SYSTEM_TYPE_DECISION_TREE_32"},
    {"SYSTEM_TYPE_DECISION_TREE_34", "SYSTEM_TYPE_DECISION_TREE_34"},
    {"SYSTEM_TYPE_DECISION_TREE_35", "SYSTEM_TYPE_DECISION_TREE_35"},
    {"SYSTEM_TYPE_DECISION_TREE_36", "SYSTEM_TYPE_DECISION_TREE_36"},
    {"SYSTEM_TYPE_DECISION_TREE_37", "SYSTEM_TYPE_DECISION_TREE_37"},
    {"SYSTEM_TYPE_DECISION_TREE_38", "SYSTEM_TYPE_DECISION_TREE_38"},
    {"SYSTEM_TYPE_DECISION_TREE_39", "SYSTEM_TYPE_DECISION_TREE_39"},
    {"TYPE_A_CONTROLS_QUESTIONS_41", "TYPE_A_CONTROLS_QUESTIONS_41"},
    {"TYPE_A_CONTROLS_QUESTIONS_42", "TYPE_A_CONTROLS_QUESTIONS_42"},
    {"TYPE_A_CONTROLS_QUESTIONS_43", "TYPE_A_CONTROLS_QUESTIONS_43"},
    {"TYPE_A_CONTROLS_QUESTIONS_44", "TYPE_A_CONTROLS_QUESTIONS_44"},
    {"TYPE_A_CONTROLS_QUESTIONS_45", "TYPE_A_CONTROLS_QUESTIONS_45"},
    {"TYPE_A_CONTROLS_QUESTIONS_46", "TYPE_A_CONTROLS_QUESTIONS_46"},
    {"TYPE_A_CONTROLS_QUESTIONS_47", "TYPE_A_CONTROLS_QUESTIONS_47"},
    {"TYPE_A_CONTROLS_QUESTIONS_48", "TYPE_A_CONTROLS_QUESTIONS_48"},
    {"TYPE_B_CONTROLS_QUESTIONS_51", "TYPE_B_CONTROLS_QUESTIONS_51"},
    {"TYPE_B_CONTROLS_QUESTIONS_52", "TYPE_B_CONTROLS_QUESTIONS_52"},
    {"TYPE_B_CONTROLS_QUESTIONS_53", "TYPE_B_CONTROLS_QUESTIONS_53"},
    {"TYPE_B_CONTROLS_QUESTIONS_54", "TYPE_B_CONTROLS_QUESTIONS_54"},
    {"TYPE_B_CONTROLS_QUESTIONS_55", "TYPE_B_CONTROLS_QUESTIONS_55"},
    {"TYPE_B_CONTROLS_QUESTIONS_56", "TYPE_B_CONTROLS_QUESTIONS_56"},
    {"TYPE_B_CONTROLS_QUESTIONS_57", "TYPE_B_CONTROLS_QUESTIONS_57"},
    {"TYPE_B_CONTROLS_QUESTIONS_58", "TYPE_B_CONTROLS_QUESTIONS_58"},
    {"TYPE_C_CONTROLS_QUESTIONS_61", "TYPE_C_CONTROLS_QUESTIONS_61"},
    {"TYPE_C_CONTROLS_QUESTIONS_62", "TYPE_C_CONTROLS_QUESTIONS_62"},
    {"TYPE_C_CONTROLS_QUESTIONS_63", "TYPE_C_CONTROLS_QUESTIONS_63"},
    {"TYPE_C_CONTROLS_QUESTIONS_64", "TYPE_C_CONTROLS_QUESTIONS_64"},
    {"TYPE_C_CONTROLS_QUESTIONS_65", "TYPE_C_CONTROLS_QUESTIONS_65"},
    {"TYPE_C_CONTROLS_QUESTIONS_66", "TYPE_C_CONTROLS_QUESTIONS_66"},
    {"TYPE_C_CONTROLS_QUESTIONS_67", "TYPE_C_CONTROLS_QUESTIONS_67"},
    {"TYPE_C_CONTROLS_QUESTIONS_68", "TYPE_C_CONTROLS_QUESTIONS_68"},
    {"TYPE_C_CONTROLS_QUESTIONS_69", "TYPE_C_CONTROLS_QUESTIONS_69"},
    {"TYPE_D_CONTROLS_QUESTIONS_71", "TYPE_D_CONTROLS_QUESTIONS_71"},
    {"TYPE_D_CONTROLS_QUESTIONS_72", "TYPE_D_CONTROLS_QUESTIONS_72"},
    {"TYPE_D_CONTROLS_QUESTIONS_73", "TYPE_D_CONTROLS_QUESTIONS_73"},
    {"TYPE_D_CONTROLS_QUESTIONS_74", "TYPE_D_CONTROLS_QUESTIONS_74"},
    {"TYPE_D_CONTROLS_QUESTIONS_75", "TYPE_D_CONTROLS_QUESTIONS_75"},
    {"TYPE_D_CONTROLS_QUESTIONS_76", "TYPE_D_CONTROLS_QUESTIONS_76"},
    {"TYPE_D_CONTROLS_QUESTIONS_77", "TYPE_D_CONTROLS_QUESTIONS_77"},
    {"TYPE_D_CONTROLS_QUESTIONS_78", "TYPE_D_CONTROLS_QUESTIONS_78"},
    {"TYPE_D_CONTROLS_QUESTIONS_710", "TYPE_D_CONTROLS_QUESTIONS_710"},
    {"TYPE_D_CONTROLS_QUESTIONS_711", "TYPE_D_CONTROLS_QUESTIONS_711"},
    {"TYPE_D_CONTROLS_QUESTIONS_712", "TYPE_D_CONTROLS_QUESTIONS_712"},
    {"TYPE_D_CONTROLS_QUESTIONS_713", "TYPE_D_CONTROLS_QUESTIONS_713"},
    {"TYPE_D_CONTROLS_QUESTIONS_714", "TYPE_D_CONTROLS_QUESTIONS_714"},
    {"TYPE_D_CONTROLS_QUESTIONS_715", "TYPE_D_CONTROLS_QUESTIONS_715"},
    {"TYPE_D_CONTROLS_QUESTIONS_716", "TYPE_D_CONTROLS_QUESTIONS_716"},
    {"TYPE_E_CONTROLS_QUESTIONS_81", "TYPE_E_CONTROLS_QUESTIONS_81"},
    {"TYPE_E_CONTROLS_QUESTIONS_82", "TYPE_E_CONTROLS_QUESTIONS_82"},
    {"TYPE_E_CONTROLS_QUESTIONS_83", "TYPE_E_CONTROLS_QUESTIONS_83"},
    {"TYPE_E_CONTROLS_QUESTIONS_84", "TYPE_E_CONTROLS_QUESTIONS_84"},
    {"TYPE_E_CONTROLS_QUESTIONS_85", "TYPE_E_CONTROLS_QUESTIONS_85"},
    {"TYPE_E_CONTROLS_QUESTIONS_86", "TYPE_E_CONTROLS_QUESTIONS_86"},
    {"TYPE_E_CONTROLS_QUESTIONS_87", "TYPE_E_CONTROLS_QUESTIONS_87"},
    {"TYPE_E_CONTROLS_QUESTIONS_88", "TYPE_E_CONTROLS_QUESTIONS_88"},
    {"TYPE_E_CONTROLS_QUESTIONS_89", "TYPE_E_CONTROLS_QUESTIONS_89"},
    {"TYPE_E_CONTROLS_QUESTIONS_810", "TYPE_E_CONTROLS_QUESTIONS_810"},
    {"TYPE_E_CONTROLS_QUESTIONS_811", "TYPE_E_CONTROLS_QUESTIONS_811"},
    {"TYPE_E_CONTROLS_QUESTIONS_812", "TYPE_E_CONTROLS_QUESTIONS_812"},
    {"TYPE_E_CONTROLS_QUESTIONS_813", "TYPE_E_CONTROLS_QUESTIONS_813"},
    {"TYPE_F_CONTROLS_QUESTIONS_91", "TYPE_F_CONTROLS_QUESTIONS_91"},
    {"TYPE_F_CONTROLS_QUESTIONS_92", "TYPE_F_CONTROLS_QUESTIONS_92"},
    {"TYPE_F_CONTROLS_QUESTIONS_93", "TYPE_F_CONTROLS_QUESTIONS_93"},
    {"SYSTEM_TYPE_DECISION_TREE_33", "SYSTEM_TYPE_DECISION_TREE_33"},
    {"TYPE_D_CONTROLS_QUESTIONS_79", "TYPE_D_CONTROLS_QUESTIONS_79"},
    {"SYSTEM_TYPE_DECISION_TREE_33_SELECTION_D_SUPPORTING_INFORMATION", "SYSTEM_TYPE_DECISION_TREE_33_SELECTION_D_SUPPORTING_INFORMATION"},
    {"SYSTEM_TYPE_DECISION_TREE_33_SELECTION_E_SUPPORTING_INFORMATION", "SYSTEM_TYPE_DECISION_TREE_33_SELECTION_E_SUPPORTING_INFORMATION"},
    {"SYSTEM_TYPE_DECISION_TREE_38_SUPPORTING_INFORMATION_RTX_MANAGED", "SYSTEM_TYPE_DECISION_TREE_38_SUPPORTING_INFORMATION_RTX_MANAGED"},
    {"SYSTEM_TYPE_DECISION_TREE_38_SUPPORTING_INFORMATION_THIRD_PARTY_MANAGER", "SYSTEM_TYPE_DECISION_TREE_38_SUPPORTING_INFORMATION_THIRD_PARTY_MANAGER"},
    {"SYSTEM_TYPE_DECISION_TREE_38_SUPPORTING_INFORMATION_CLOUD_SERVICE_PROVIDER", "SYSTEM_TYPE_DECISION_TREE_38_SUPPORTING_INFORMATION_CLOUD_SERVICE_PROVIDER"},
    {"SYSTEM_TYPE_DECISION_TREE_38_SUPPORTING_INFORMATION_CLOUD_TYPE", "SYSTEM_TYPE_DECISION_TREE_38_SUPPORTING_INFORMATION_CLOUD_TYPE"},
    {"SYSTEM_TYPE_DECISION_TREE_38_SUPPORTING_INFORMATION_SAAS_PROVIDER__THIRD_PARTY_NAME", "SYSTEM_TYPE_DECISION_TREE_38_SUPPORTING_INFORMATION_SAAS_PROVIDER_THIRD_PARTY_NAME"},
    {"SYSTEM_TYPE_DECISION_TREE_38_SUPPORTING_INFORMATION_COUNTRIES", "SYSTEM_TYPE_DECISION_TREE_38_SUPPORTING_INFORMATION_COUNTRIES"},
    {"HELPER_2ND_LEVEL_REVIEW_REQUIRED", "HELPER_2ND_LEVEL_REVIEW_REQUIRED"},
    {"FIRST_LEVEL_REVIEWER_COMMENTS", "FIRST_LEVEL_REVIEWER_COMMENTS"},
    {"SECOND_LEVEL_REVIEWER_COMMENTS", "SECOND_LEVEL_REVIEWER_COMMENTS"},
    {"SYSTEM_TYPE_DECISION_TREE_34_SELECTION_C_SUPPORTING_INFORMATION", "SYSTEM_TYPE_DECISION_TREE_34_SELECTION_C_SUPPORTING_INFORMATION"},
    {"TYPE_A_CONTROLS_QUESTIONS_46_SELECTION_OTHER_DESCRIBE_SUPPORTING_INFORMATION", "TYPE_A_CONTROLS_QUESTIONS_46_SELECTION_OTHER_DESCRIBE_SUPPORTING_INFORMATION"},
    {"TYPE_B_CONTROLS_QUESTIONS_56_SELECTION_OTHER_DESCRIBE_SUPPORTING_INFORMATION", "TYPE_B_CONTROLS_QUESTIONS_56_SELECTION_OTHER_DESCRIBE_SUPPORTING_INFORMATION"},
    {"TYPE_C_CONTROLS_QUESTIONS_67_SELECTION_OTHER_DESCRIBE_SUPPORTING_INFORMATION", "TYPE_C_CONTROLS_QUESTIONS_67_SELECTION_OTHER_DESCRIBE_SUPPORTING_INFORMATION"},
    {"TYPE_D_CONTROLS_QUESTIONS_76_SELECTION_OTHER_DESCRIBE_SUPPORTING_INFORMATION", "TYPE_D_CONTROLS_QUESTIONS_76_SELECTION_OTHER_DESCRIBE_SUPPORTING_INFORMATION"},
    {"TYPE_E_CONTROLS_QUESTIONS_811_SELECTION_OTHER_DESCRIBE_SUPPORTING_INFORMATION", "TYPE_E_CONTROLS_QUESTIONS_811_SELECTION_OTHER_DESCRIBE_SUPPORTING_INFORMATION"},
    {"TYPE_F_CONTROLS_QUESTIONS_93_SELECTION_OTHER_DESCRIBE_SUPPORTING_INFORMATION", "TYPE_F_CONTROLS_QUESTIONS_93_SELECTION_OTHER_DESCRIBE_SUPPORTING_INFORMATION"},
    {"IDENTIFYING_INFORMATION_11", "IDENTIFYING_INFORMATION_11"},
    {"IDENTIFYING_INFORMATION_14", "IDENTIFYING_INFORMATION_14"},
    {"BASELINE_SYSTEM_1", "BASELINE_SYSTEM_1"},
    {"BASELINE_SYSTEM_2", "BASELINE_SYSTEM_2"},
    {"BASELINE_SYSTEM_3", "BASELINE_SYSTEM_3"},
    {"BASELINE_SYSTEM_4", "BASELINE_SYSTEM_4"},
    {"BASELINE_SYSTEM_5", "BASELINE_SYSTEM_5"},
    {"BASELINE_SYSTEM_6", "BASELINE_SYSTEM_6"},
    {"BASELINE_SYSTEM_7", "BASELINE_SYSTEM_7"},
    {"BASELINE_SYSTEM_8", "BASELINE_SYSTEM_8"},
    {"BASELINE_SYSTEM_9", "BASELINE_SYSTEM_9"},
    {"BASELINE_SYSTEM_10", "BASELINE_SYSTEM_10"},
    {"BASELINE_SYSTEM_1_SUPPORTING_INFORMATION_CLOUD_SERVICE_PROVIDER", "BASELINE_SYSTEM_1_SUPPORTING_INFORMATION_CLOUD_SERVICE_PROVIDER"},
    {"BASELINE_SYSTEM_1_SUPPORTING_INFORMATION_CLOUD_TYPE", "BASELINE_SYSTEM_1_SUPPORTING_INFORMATION_CLOUD_TYPE"},
    {"BASELINE_SYSTEM_1_SUPPORTING_INFORMATION_COUNTRIES", "BASELINE_SYSTEM_1_SUPPORTING_INFORMATION_COUNTRIES"},
    {"BASELINE_SYSTEM_1_SUPPORTING_INFORMATION_RTX_MANAGED", "BASELINE_SYSTEM_1_SUPPORTING_INFORMATION_RTX_MANAGED"},
    {"BASELINE_SYSTEM_1_SUPPORTING_INFORMATION_SAAS_PROVIDER__THIRD_PARTY_NAME", "BASELINE_SYSTEM_1_SUPPORTING_INFORMATION_SAAS_PROVIDER_THIRD_PARTY_NAME"},
    {"BASELINE_SYSTEM_1_SUPPORTING_INFORMATION_THIRD_PARTY_MANAGER", "BASELINE_SYSTEM_1_SUPPORTING_INFORMATION_THIRD_PARTY_MANAGER"},
    {"BASELINE_SYSTEM_3_SUPPORTING_INFORMATION_NATURE_OR_PURPOSE", "BASELINE_SYSTEM_3_SUPPORTING_INFORMATION_NATURE_OR_PURPOSE"},
    {"BASELINE_SYSTEM_4_SUPPORTING_INFORMATION_COUNTRIES", "BASELINE_SYSTEM_4_SUPPORTING_INFORMATION_COUNTRIES"},
    {"BASELINE_SYSTEM_5_SUPPORTING_INFORMATION_NATURE_OR_PURPOSE", "BASELINE_SYSTEM_5_SUPPORTING_INFORMATION_NATURE_OR_PURPOSE"},
    {"BASELINE_SYSTEM_9_SUPPORTING_INFORMATION_ACCESS_LIMITATIONS", "BASELINE_SYSTEM_9_SUPPORTING_INFORMATION_ACCESS_LIMITATIONS"},
    {"HELPER_BASELINE_SYSTEM_ASSESSMENT", "HELPER_BASELINE_SYSTEM_ASSESSMENT"},
    {"FIRST_LEVEL_REVIEWER_DENIAL_JUSTIFICATION", "FIRST_LEVEL_REVIEWER_DENIAL_JUSTIFICATION"},
    {"SECOND_LEVEL_REVIEWER_DENIAL_JUSTIFICATION", "SECOND_LEVEL_REVIEWER_DENIAL_JUSTIFICATION"},
    {"FIRST_LEVEL_REVIEWER_STATUS", "FIRST_LEVEL_REVIEWER_STATUS"},
    {"SECOND_LEVEL_REVIEWER_STATUS", "SECOND_LEVEL_REVIEWER_STATUS"},
    {"ASSESSMENT_TYPE", "ASSESSMENT_TYPE"},
    {"BASELINE_SYSTEM_1_SUPPORTING_INFORMATION_CUSTOMER", "BASELINE_SYSTEM_1_SUPPORTING_INFORMATION_CUSTOMER"},
    {"BASELINE_SYSTEM_1_SUPPORTING_INFORMATION_HOSTING_LOCATION", "BASELINE_SYSTEM_1_SUPPORTING_INFORMATION_HOSTING_LOCATION"},
    {"BASELINE_SYSTEM_3_SUPPORTING_INFORMATION_DESCRIPTION_OF_CONTROL", "BASELINE_SYSTEM_3_SUPPORTING_INFORMATION_DESCRIPTION_OF_CONTROL"},
    {"BASELINE_SYSTEM_5_SUPPORTING_INFORMATION_DESCRIPTION_OF_CONTROL", "BASELINE_SYSTEM_5_SUPPORTING_INFORMATION_DESCRIPTION_OF_CONTROL"},
    {"SYSTEM_TYPE_DECISION_TREE_38_SUPPORTING_INFORMATION_DESCRIPTION_OF_OTHER", "SYSTEM_TYPE_DECISION_TREE_38_SUPPORTING_INFORMATION_DESCRIPTION_OF_OTHER"},
    {"BASELINE_SYSTEM_1_SUPPORTING_INFORMATION_DESCRIPTION_OF_OTHER", "BASELINE_SYSTEM_1_SUPPORTING_INFORMATION_DESCRIPTION_OF_OTHER"},
    {"SYSTEM_TYPE_DECISION_TREE_38_SUPPORTING_INFORMATION_CUSTOMER_NAME", "SYSTEM_TYPE_DECISION_TREE_38_SUPPORTING_INFORMATION_CUSTOMER_NAME"},
    {"SYSTEM_TYPE_DECISION_TREE_38_SUPPORTING_INFORMATION_HOSTING_LOCATION", "SYSTEM_TYPE_DECISION_TREE_38_SUPPORTING_INFORMATION_HOSTING_LOCATION"},
    {"HELPER_ASSESSMENT_REQUIRED", "HELPER_ASSESSMENT_REQUIRED"},
    {"IDENTIFYING_INFORMATION_13", "IDENTIFYING_INFORMATION_13"},
    {"SECOND_LEVEL_REVIEWER_COLLABORATOR", "SECOND_LEVEL_REVIEWER_COLLABORATOR"},
    {"RESPONDENT", "RESPONDENT"},
    {"FIRST_LEVEL_REVIEWER_COLLABORATOR", "FIRST_LEVEL_REVIEWER_COLLABORATOR"},
    {"IDENTIFYING_INFORMATION_12", "IDENTIFYING_INFORMATION_12"},
    {"FIRST_LEVEL_REVIEWER", "FIRST_LEVEL_REVIEWER"},
    {"SECOND_LEVEL_REVIEWER", "SECOND_LEVEL_REVIEWER"},
    {"TARGET_CONTACTS", "TARGET_CONTACTS"},
    {"BUSINESS", "BUSINESS_NAME"},
    {"AUTHORIZATION_PACKAGE", "AUTHORIZATION_PACKAGE_NAME"},
    {"IDENTIFYING_INFORMATION_13_COLLINS_SUPPORTING_INFORMATION", "IDENTIFYING_INFORMATION_13_COLLINS_SUPPORTING_INFORMATION"},
    {"HELPER_ASSESSMENT_COMPLETE", "HELPER_ASSESSMENT_COMPLETE"},
    {"HELPER_EVIDENCE_REQUIRED", "HELPER_EVIDENCE_REQUIRED"},
    {"HELPER_BACKGROUND_PREVIOUS_ASSESSMENT_INFORMATION_COMPLETE", "HELPER_BACKGROUND_PREVIOUS_ASSESSMENT_INFORMATION_COMPLETE"},
    {"HELPER_EVIDENCE_ATTACHED", "HELPER_EVIDENCE_ATTACHED"},
    {"HELPER_SYSTEM_TYPE_DECISION_TREE_COMPLETE", "HELPER_SYSTEM_TYPE_DECISION_TREE_COMPLETE"},
    {"IDENTIFYING_INFORMATION_16", "IDENTIFYING_INFORMATION_16"},
    {"IDENTIFYING_INFORMATION_15", "IDENTIFYING_INFORMATION_15"},
    {"SECOND_LEVEL_REVIEWER_COLLABORATOR_NAME", "SECOND_LEVEL_REVIEWER_COLLABORATOR_ONLY"},
    {"SECOND_LEVEL_REVIEWER_COLLABORATOR_UID", "SECOND_LEVEL_REVIEWER_COLLABORATOR_ID_ONLY"},
    {"RESPONDENT_NAME", "RESPONDENT_NAME_ONLY"},
    {"RESPONDENT_UID", "RESPONDENT_ID_ONLY"},
    {"FIRST_LEVEL_REVIEWER_COLLABORATOR_NAME", "FIRST_LEVEL_REVIEWER_COLLABORATOR_NAME_ONLY"},
    {"FIRST_LEVEL_REVIEWER_COLLABORATOR_UID", "FIRST_LEVEL_REVIEWER_COLLABORATOR_ID_ONLY"},
    {"IDENTIFYING_INFORMATION_12_NAME", "IDENTIFYING_INFORMATION_12_NAME_ONLY"},
    {"IDENTIFYING_INFORMATION_12_UID", "IDENTIFYING_INFORMATION_12_ID_ONLY"},
    {"FIRST_LEVEL_REVIEWER_NAME", "FIRST_LEVEL_REVIEWER_NAME_ONLY"},
    {"FIRST_LEVEL_REVIEWER_UID", "FIRST_LEVEL_REVIEWER_ID_ONLY"},
    {"SECOND_LEVEL_REVIEWER_NAME", "SECOND_LEVEL_REVIEWER_NAME_ONLY"},
    {"SECOND_LEVEL_REVIEWER_UID", "SECOND_LEVEL_REVIEWER_ID_ONLY"},
    {"CONFIRMEDINARCHER", "ARCHER_CONTENT_GLOBAL_TRADE_ASSESSMENT_CONFIRMED_IN_ARCHER"},
    {"APPROVAL_DATE", "APPROVAL_DATE"},
    {"FIRST_LEVEL_REVIEWER_APPROVAL_COMMENTS", "FIRST_LEVEL_REVIEWER_APPROVAL_COMMENTS"},
    {"SECOND_LEVEL_REVIEWER_APPROVAL_COMMENTS", "SECOND_LEVEL_REVIEWER_APPROVAL_COMMENTS"}
}),
C2S = {"LAST_UPDATED", "CREATED_DATE", "GLOBAL_TRADE_ASSESSMENT_ID", "RESPONDENT_STATUS", "SYSTEM_TYPE_DECISION_TREE_31", "SYSTEM_TYPE_DECISION_TREE_38", "SYSTEM_TYPE_DECISION_TREE_39", "IDENTIFYING_INFORMATION_11", "ASSESSMENT_TYPE", "AUTHORIZATION_PACKAGE", "CONFIRMEDINARCHER", "APPROVAL_DATE"},
Restored = {"SECOND_LEVEL_REVIEWER_COLLABORATOR_NAME", "SECOND_LEVEL_REVIEWER_COLLABORATOR_UID", "FIRST_LEVEL_REVIEWER_COLLABORATOR_NAME", "FIRST_LEVEL_REVIEWER_COLLABORATOR_UID", "APPROVAL_DATE"},
SourceSQL = ReadExport(SQLPath),
SourceSF = ReadExport(SnowflakePath),
HeadersSQL = List.Buffer(Table.ColumnNames(SourceSQL)),
HeadersSF = List.Buffer(Table.ColumnNames(SourceSF)),
Target = Record.FromList(FieldMap[Snowflake], FieldMap[SQL]),
SFName = (f as text) as text => Record.Field(Target, f),
HasBoth = (f as text) as logical => List.Contains(HeadersSQL, f) and List.Contains(HeadersSF, SFName(f)),
Key = "CONTENT_ID",
Assessment = "GLOBAL_TRADE_ASSESSMENT_ID",
RawPackage = "AUTHORIZATION_PACKAGE",
CheckFields = List.Distinct({Key} & C2S & Restored),
DayFields = {"LAST_UPDATED", "CONFIRMEDINARCHER", "CREATED_DATE", "APPROVAL_DATE"},
ListFields = List.RemoveItems(Restored, {"APPROVAL_DATE"}),

// Keep IDs as text. Treat CSV null markers and blank cells as missing.
Clean = (x as nullable text) as nullable text =>
    if x = null then null else
    let t = Text.Trim(x) in if t = "" or Text.Upper(t, "en-US") = "NULL" then null else t,
Project = (t as table, isSF as logical) as table =>
    let
        original = if isSF then List.Transform(CheckFields, SFName) & {RawPackage} else CheckFields,
        names = if isSF then CheckFields & {"__RAW_PACKAGE"} else CheckFields,
        renameMap = Record.FromList(names, original),
        cols = Table.SelectColumns(t, original, MissingField.UseNull),
        renamed = Table.TransformColumnNames(cols, each Record.Field(renameMap, _)),
        cleaned = Table.TransformColumns(renamed, List.Transform(names, each {_, Clean, type nullable text}))
    in Table.Buffer(cleaned),
S = Project(SourceSQL, false),
F = Project(SourceSF, true),
NS = Table.RowCount(S),
NF = Table.RowCount(F),
N = (x as number) as text => Number.ToText(x, "#,0", "en-US"),
Ex = (ids as list) as text =>
    let sample = List.FirstN(List.Sort(List.Distinct(ids), Comparer.Ordinal), 3)
    in if List.IsEmpty(sample) then "" else " Example Content_IDs: " & Text.Combine(sample, ", ") & ".",
Row = (id as text, label as text, result as text, evidence as text, action as text) as list =>
    {label, id, result, evidence, action, null},

SchemaRow = (label as text, fields as list) as list =>
    let
        missSQL = List.Difference(fields, HeadersSQL),
        missSF = List.Difference(List.Transform(fields, SFName), HeadersSF),
        total = List.Count(fields),
        none = List.IsEmpty(missSQL) and List.IsEmpty(missSF)
    in Row("Schema", label, if none then "Pass" else "Needs Review",
        "SQL " & N(total - List.Count(missSQL)) & " of " & N(total) &
        ". Snowflake " & N(total - List.Count(missSF)) & " of " & N(total) & ".",
        if none then "All expected headers found." else "See the missing-field rows."),
MissingRows = List.Transform(List.Select(FieldMap[SQL], each not HasBoth(_)), (f) =>
    Row("Schema", f, "Needs Review",
        Text.Combine(List.RemoveNulls({
            if List.Contains(HeadersSQL,f) then null else "Missing SQL header: " & f,
            if List.Contains(HeadersSF,SFName(f)) then null else "Missing Snowflake header: " & SFName(f)
        }), ". "), "Check the full export and column mapping.")),
NewSQL = List.Difference(HeadersSQL, FieldMap[SQL]),
ExtraRows = if List.IsEmpty(NewSQL) then {} else
    {Row("Schema", "SQL columns outside the field list", "Review mapping",
        Text.Combine(NewSQL, ", "), "These were outside the 169-field workbook baseline.")},

Profile = (t as table, f as text) as table =>
    let ids = Table.SelectColumns(t,{f}),
        present = Table.SelectRows(ids, each Record.Field(_,f) <> null)
    in Table.Buffer(Table.Group(present, {f}, {{"Uses", each Table.RowCount(_), Int64.Type}})),
SK = Profile(S, Key), FK = Profile(F, Key),
SA = Profile(S, Assessment), FA = Profile(F, Assessment),
IdCheck = (f as text, left as table, right as table) as list =>
    if not HasBoth(f) then Row("GT02", f & " completeness and duplicates", "Not tested", "A key header is missing.", "")
    else let
        popSQL = List.Sum({0} & left[Uses]), popSF = List.Sum({0} & right[Uses]),
        blankSQL = NS - popSQL, blankSF = NF - popSF,
        repSQL = popSQL - Table.RowCount(left), repSF = popSF - Table.RowCount(right),
        clean = NS > 0 and NF > 0 and blankSQL + blankSF + repSQL + repSF = 0,
        describe = (p as table, blanks as number, reps as number) as text =>
            N(Table.RowCount(p)) & " distinct, " & N(blanks) & " missing, " & N(reps) & " extra duplicate rows"
    in Row("GT02", f & " completeness and duplicates", if clean then "Pass" else "Needs Review",
        "SQL: " & describe(left,blankSQL,repSQL) & ". Snowflake: " & describe(right,blankSF,repSF) & ".",
        "Each populated ID should occur once within its source."),

IdJoin = (left as table, right as table, kind as number) as table =>
    Table.RemoveColumns(Table.NestedJoin(left,{Key},right,{Key},"__hit",kind),{"__hit"}),
HasKey = HasBoth(Key),
SQLOnly = IdJoin(SK, FK, JoinKind.LeftAnti),
SFOnly = IdJoin(FK, SK, JoinKind.LeftAnti),
Shared = IdJoin(SK, FK, JoinKind.Inner),
UniqueSQL = IdJoin(S, Table.SelectRows(SK, each [Uses]=1), JoinKind.Inner),
UniqueSF = IdJoin(F, Table.SelectRows(FK, each [Uses]=1), JoinKind.Inner),
RightFields = List.RemoveItems(CheckFields,{Key}),
Join = Table.NestedJoin(UniqueSQL,{Key},UniqueSF,{Key},"__sf",JoinKind.Inner),
Pairs = Table.Buffer(Table.ExpandTableColumn(Join, "__sf", RightFields, List.Transform(RightFields, each "SF." & _))),
NP = Table.RowCount(Pairs),
Coverage = if not HasKey then
    {Row("GT04","Same assessments in both exports","Not tested","A Content_ID header is missing.","Check the schema rows.")}
    else {
        Row("GT04","Same assessments in both exports",
            if Table.RowCount(SQLOnly)+Table.RowCount(SFOnly)=0 and Table.RowCount(Shared)>0 then "Pass" else "Needs Review",
            N(Table.RowCount(Shared)) & " in both. " & N(Table.RowCount(SQLOnly)) & " only in SQL. " &
            N(Table.RowCount(SFOnly)) & " only in Snowflake.",
            "SQL only." & Ex(Table.Column(SQLOnly,Key)) & " Snowflake only." & Ex(Table.Column(SFOnly,Key))),
        Row("GT04","Shared assessments used for field checks", if NP=0 or NP<Table.RowCount(Shared) then "Needs Review" else "Pass",
            N(NP) & " shared Content_IDs occur once on both sides. " &
            N(Table.RowCount(Shared)-NP) & " shared IDs excluded because they repeat.", "Each field check uses these matched records.")
    },

// Compare dates by the calendar day displayed in each export. Keep parse failures visible.
Day = (s as nullable text) as nullable date =>
    if s=null then null else
    try Date.From(DateTimeZone.RemoveZone(DateTimeZone.FromText(s,[Culture="en-US"])))
    otherwise try Date.From(DateTime.FromText(s,[Culture="en-US"]))
    otherwise try Date.FromText(s,[Culture="en-US"]) otherwise null,
Tokens = (s as text) as list => List.Sort(List.Transform(Text.Split(s,";"), Text.Trim), Comparer.Ordinal),
Classify = (a as nullable text, b as nullable text, f as text) as text =>
    if a=null and b=null then "Both blank"
    else let isDay = List.Contains(DayFields,f),
             x = if isDay then Day(a) else a, y = if isDay then Day(b) else b
    in if (a<>null and x=null) or (b<>null and y=null) then "Unreadable"
       else if a=null or b=null then "One blank"
       else if (if List.Contains(ListFields,f) then Tokens(a)=Tokens(b) else x=y)
            then "Match" else "Different",
FieldRow = (f as text) as list =>
    let
        test = if f="APPROVAL_DATE" then "GT05" else if f="CREATED_DATE" then "GT06"
            else if f="AUTHORIZATION_PACKAGE" then "GT07" else "GT04",
        isDay = List.Contains(DayFields,f),
        label = f & (if isDay then " (calendar date)" else ""),
        action = if f="CREATED_DATE" then "SQL creation vs Snowflake first publication. Confirm they represent the same event."
            else if f="AUTHORIZATION_PACKAGE" then "Compares SQL package text with Snowflake resolved package names, including list order."
            else if List.Contains(ListFields,f) then "Compares semicolon lists after trimming and sorting. Repeated entries are retained."
            else if isDay then "Compares the displayed day using US date formats." else "Compares trimmed text.",
        cols = Table.SelectColumns(Pairs,{Key,f,"SF." & f}),
        scored = Table.Buffer(Table.AddColumn(cols,"Outcome",
            each Classify(Record.Field(_,f), Record.Field(_,"SF." & f), f), type text)),
        counts = Table.Group(scored,{"Outcome"},{{"n", each Table.RowCount(_), Int64.Type}}),
        byOutcome = Record.FromList(counts[n], counts[Outcome]),
        count = (k as text) as number => Record.FieldOrDefault(byOutcome,k,0),
        examples = Table.SelectRows(scored, each List.Contains({"Different","One blank","Unreadable"},[Outcome])),
        issues = count("Different")+count("One blank")+count("Unreadable"),
        status = if f="CREATED_DATE" or issues>0 then "Needs Review" else if count("Match")=0 then "No populated pairs" else "Pass"
    in if not HasBoth(f) or not HasKey or NP=0 then
        Row(test,label,"Not tested","Missing mapped header or no shared unique Content_IDs.", action)
    else Row(test,label,status,
        N(count("Match")) & " populated matches, " & N(count("Different")) & " different, " &
        N(count("One blank")) & " blank on one side, " & N(count("Both blank")) & " blank on both, " &
        N(count("Unreadable")) & " unreadable." & Ex(Table.Column(examples,Key)), action),
Latest = (t as table, f as text) as record =>
    let values = List.RemoveNulls(Table.Column(t,f)), parsed = List.RemoveNulls(List.Transform(values,Day))
    in [Value=List.Max(parsed,null), Bad=List.Count(values)-List.Count(parsed)],
DayText = (d as nullable date) as text => if d=null then "no readable date" else Date.ToText(d,"yyyy-MM-dd","en-US"),
LatestRow = (f as text) as list =>
    let a = Latest(S,f), b = Latest(F,f)
    in if not HasBoth(f) then Row("Dates","Latest " & f,"Not tested","Missing mapped date header.","")
    else Row("Dates","Latest " & f & " (calendar date)",
        if a[Bad]+b[Bad]>0 or a[Value]=null or b[Value]=null or a[Value]<>b[Value] then "Needs Review" else "Pass",
        "SQL: " & DayText(a[Value]) & ". Snowflake: " & DayText(b[Value]) &
        ". Unreadable populated dates: SQL " & N(a[Bad]) & ", Snowflake " & N(b[Bad]) & ".",
        "Maximum over each full export. Refresh order is checked separately."),
Package = "AUTHORIZATION_PACKAGE",
NamesAvailable = HasBoth(Package),
SQLNames = List.NonNullCount(Table.Column(S,Package)),
SFNames = List.NonNullCount(Table.Column(F,Package)),
RawCount = List.NonNullCount(F[__RAW_PACKAGE]),
Unresolved = Table.SelectRows(F, each [__RAW_PACKAGE]<>null and Record.Field(_,Package)=null),
PartCount = (v as nullable text) as number => if v=null then 0 else List.Count(List.Select(Text.Split(v,";"), each Text.Trim(_) <> "")),
Multi = Table.SelectRows(F, each PartCount([__RAW_PACKAGE])>1),
ListSizeDiff = Table.SelectRows(Multi, each PartCount([__RAW_PACKAGE])<>PartCount(Record.Field(_,Package))),
PackageRows = {
    if not NamesAvailable then Row("GT07","Rows with a package name","Not tested","A package-name header is missing.","")
    else Row("GT07","Rows with a package name","Info","SQL " & N(SQLNames) & " of " & N(NS) &
        ". Snowflake " & N(SFNames) & " of " & N(NF) & ".",
        "CSV population check. SQL IS NOT NULL can also keep an empty string."),
    if not List.ContainsAll(HeadersSF,{RawPackage,SFName(Package)}) then
        Row("GT07","Snowflake package references without a name","Not tested","Missing raw reference or resolved name header.","")
    else Row("GT07","Snowflake package references without a name",if RawCount=0 then "No populated references" else if Table.IsEmpty(Unresolved) then "Pass" else "Needs Review",
        N(Table.RowCount(Unresolved)) & " rows contain a reference and no resolved package name." & Ex(List.RemoveNulls(Table.Column(Unresolved,Key))),
        "Snowflake keeps assessments whose package lookup is empty."),
    if not List.ContainsAll(HeadersSF,{RawPackage,SFName(Package)}) then
        Row("GT07","Snowflake assessments with multiple packages","Not tested","Missing raw reference or resolved name header.","")
    else Row("GT07","Snowflake assessments with multiple packages",
        if Table.IsEmpty(Multi) then "No multi-package rows" else if Table.IsEmpty(ListSizeDiff) then "Pass (list counts)" else "Needs Review",
        N(Table.RowCount(Multi)) & " rows list multiple references. " & N(Table.RowCount(ListSizeDiff)) &
        " have a different number of resolved names." & Ex(List.RemoveNulls(Table.Column(ListSizeDiff,Key))),
        "Counts nonempty semicolon-separated entries. Package identity is checked in the downstream links.")
},
Result = #table(type table [Check=text, #"Test ID"=text, Result=text, Evidence=text, #"Details or Action"=text, #"Comments(Grady)"=nullable text],
    {SchemaRow("All workbook fields",FieldMap[SQL]), SchemaRow("Fields used by C2S",C2S),
     SchemaRow("Five previously missing fields",Restored)} & MissingRows & ExtraRows &
    {Row("GT03","Total assessment rows",if NS=NF and NS>0 then "Pass" else "Needs Review",
        "SQL " & N(NS) & ". Snowflake " & N(NF) & ".", "Counts cover the full exports before the C2S package filter."),
     IdCheck(Key,SK,FK), IdCheck(Assessment,SA,FA)} & Coverage &
    List.Transform(List.RemoveItems(CheckFields,{Key}), FieldRow) & PackageRows &
    {LatestRow("LAST_UPDATED"), LatestRow("CONFIRMEDINARCHER"),
     Row("GT01","Refresh order","Separate check","The exports do not contain the job run times.","Confirm Matillion completed before the C2S refresh using the run log."),
     Row("GT08","Downstream C2S results","Separate check","Source comparison only.","Check dimGlobalTrade, package links and affected asset or factCyberEvents results after refresh.")})

in
Result
