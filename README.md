let
    // Replace only the text between these two pairs of quotation marks.
    SqlFile = "C:\YourFolder\SQL AuthorizationPackage.csv",
    SnowflakeFile = "C:\YourFolder\Snowflake AuthorizationPackage.csv",

    ReadExport = (path as text, source as text, required as list) as table =>
        let
            bytes = Binary.Buffer(File.Contents(path)),
            byteCount = Binary.Length(bytes),
            prefix = if byteCount < 2 then "" else Text.Upper(Binary.ToText(Binary.Range(bytes, 0, 2), BinaryEncoding.Hex)),
            encoding =
                if prefix = "FFFE" then 1200
                else if prefix = "FEFF" then 1201
                else
                    let decoded = try Text.FromBinary(bytes, 65001)
                    in if decoded[HasError] then 1252
                       else if Text.Contains(decoded[Value], Character.FromNumber(65533)) then 1252
                       else 65001,
            csv = Csv.Document(bytes, [Delimiter = ",", Encoding = encoding, QuoteStyle = QuoteStyle.Csv]),
            promoted = Table.PromoteHeaders(csv, [PromoteAllScalars = true]),
            headers = Table.TransformColumnNames(promoted, each
                Text.Upper(Text.Trim(Text.Replace(_, Character.FromNumber(65279), "")), "en-US")),
            missing = List.Difference(required, Table.ColumnNames(headers))
        in
            if not List.IsEmpty(missing) then
                error (source & " CSV: missing header(s) " & Text.Combine(missing, ", ") &
                       ". Check the file path, header row and comma delimiter.")
            else Table.Buffer(headers),

    SqlInput = ReadExport(SqlFile, "SQL", {"CONTENT_ID", "SAP_ID_RTN"}),
    SfInput = ReadExport(SnowflakeFile, "Snowflake", {"ARCHER_CONTENT_AUTHORIZATION_PACKAGE_CONTENT_ID", "SAP_ID_RTN"}),

    // Keep identifiers as text so leading zeros are preserved.
    NormalizeKey = (value as any) as nullable text =>
        let
            s = if value = null then "" else Text.Trim(Text.From(value))
        in
            if s = "" or Text.Upper(s, "en-US") = "NULL" then null else s,

    GroupKeys = (input as table, column as text, keepRow as logical) as table =>
        let
            keyed = Table.AddColumn(input, "__uat_key", each NormalizeKey(Record.Field(_, column)), type nullable text),
            present = Table.SelectRows(keyed, each [__uat_key] <> null),
            grouped = Table.Group(present, {"__uat_key"}, {
                {"n", each Table.RowCount(_), Int64.Type},
                {"sample", each if keepRow and Table.RowCount(_) = 1 then Record.RemoveFields(_{0}, {"__uat_key"}) else null, type nullable record}
            })
        in
            Table.Buffer(Table.RenameColumns(grouped, {{"__uat_key", "id"}})),

    SqlTotal = Table.RowCount(SqlInput),
    SfTotal = Table.RowCount(SfInput),
    CountValues = (values as list) as number => if List.IsEmpty(values) then 0 else List.Sum(values),
    SampleKeys = (keys as list) as text =>
        if List.IsEmpty(keys) then "none" else Text.Combine(List.FirstN(List.Sort(keys, Comparer.Ordinal), 3), ", "),

    ProfileKey = (label as text, sqlColumn as text, sfColumn as text, keepRow as logical) as record =>
        let
            left = Table.RenameColumns(GroupKeys(SqlInput, sqlColumn, keepRow), {{"n", "ln"}, {"sample", "L"}}),
            right = Table.RenameColumns(GroupKeys(SfInput, sfColumn, keepRow), {{"id", "rid"}, {"n", "rn"}, {"sample", "R"}}),
            joined = Table.Buffer(Table.Join(left, {"id"}, right, {"rid"}, JoinKind.FullOuter)),
            sqlOnly = Table.SelectRows(joined, each [rn] = null),
            sfOnly = Table.SelectRows(joined, each [ln] = null),
            bothSides = Table.SelectRows(joined, each [ln] <> null and [rn] <> null),
            paired = Table.SelectColumns(Table.SelectRows(bothSides, each [ln] = 1 and [rn] = 1), {"id", "L", "R"}),
            sqlDistinct = Table.RowCount(left),
            sfDistinct = Table.RowCount(right),
            sqlPresent = CountValues(left[ln]),
            sfPresent = CountValues(right[rn]),
            sqlDupIds = Table.SelectRows(left, each [ln] > 1)[id],
            sfDupIds = Table.SelectRows(right, each [rn] > 1)[rid],
            rows = {
                {"Keys", label & " | distinct populated IDs", sqlDistinct, sfDistinct, null, null, null},
                {"Keys", label & " | missing ID rows", SqlTotal - sqlPresent, SfTotal - sfPresent, null, null, "Includes empty, spaces and NULL markers in the exports."},
                {"Keys", label & " | extra duplicate rows", sqlPresent - sqlDistinct, sfPresent - sfDistinct, null, null, "Repeated IDs: SQL " & SampleKeys(sqlDupIds) & " / SF " & SampleKeys(sfDupIds)},
                {"Keys", label & " | IDs on one side only", Table.RowCount(sqlOnly), Table.RowCount(sfOnly), null, null, "SQL-only: " & SampleKeys(sqlOnly[id]) & " / SF-only: " & SampleKeys(sfOnly[rid])},
                {"Keys", label & " | shared distinct IDs", null, null, Table.RowCount(bothSides), null, null}
            }
        in
            [Rows = rows, Pairs = paired],

    ContentProfile = ProfileKey("Content_ID", "CONTENT_ID", "ARCHER_CONTENT_AUTHORIZATION_PACKAGE_CONTENT_ID", true),
    SapProfile = ProfileKey("SAP_ID_RTN", "SAP_ID_RTN", "SAP_ID_RTN", false),
    UniquePairs = Table.Buffer(ContentProfile[Pairs]),
    BadPairs = Table.SelectRows(UniquePairs, each
        let
            l = NormalizeKey(Record.Field([L], "SAP_ID_RTN")),
            r = NormalizeKey(Record.Field([R], "SAP_ID_RTN"))
        in
            l = null or r = null or l <> r
    ),
    KeyRows = List.Combine({
        {{"Keys", "Total export rows", SqlTotal, SfTotal, null, null, "All rows in the two files."}},
        ContentProfile[Rows], SapProfile[Rows],
        {{"Keys", "SAP_ID_RTN on unique shared Content_ID", null, null, Table.RowCount(UniquePairs), Table.RowCount(BadPairs),
            "Missing or different SAP_ID_RTN. Example Content_IDs: " & SampleKeys(BadPairs[id])}}
    }),

    MappingRows = {
        {"Content_ID", "ARCHER_CONTENT_AUTHORIZATION_PACKAGE_CONTENT_ID", "text", "Record identifier"},
        {"Last_Updated", "ARCHER_CONTENT_AUTHORIZATION_PACKAGE_LAST_UPDATED", "date", "BaseTable path"},
        {"SAP_ID", "SAP_ID", "text", "Outside main/role-UID projection"},
        {"Authorization_Package_Name", "AUTHORIZATION_PACKAGE_NAME", "text", "Main and role/UID views"},
        {"Acronym", "ACRONYM", "text", "Main and role/UID views"},
        {"Operational_Status", "OPERATIONAL_STATUS", "text", "Main and role/UID views"},
        {"Information_Classification", "INFORMATION_CLASSIFICATION", "text", "Main and role/UID views"},
        {"ATOIATO_Date", "ATOIATO_DATE", "date", "Main and role/UID views"},
        {"ATOIATO_Expiration_Date", "ATOIATO_EXPIRATION_DATE", "date", "Main and role/UID views"},
        {"Connectivity", "CONNECTIVITY", "text", "Main and role/UID views"},
        {"Information_Owner_IO", "INFORMATION_OWNER_IO", "role", "Outside main/role-UID projection"},
        {"Information_System_Owner_ISO", "INFORMATION_SYSTEM_OWNER_ISO", "role", "Main role expansion and role/UID view"},
        {"Senior_Information_Systems_Security_Officer_SISSO", "SENIOR_INFORMATION_SYSTEMS_SECURITY_OFFICER_SISSO", "role", "Outside main/role-UID projection"},
        {"Authorizing_Official_AO", "AUTHORIZING_OFFICIAL_AO", "role", "Main role expansion and role/UID view"},
        {"Information_System_Security_Officer_ISSO", "INFORMATION_SYSTEM_SECURITY_OFFICER_ISSO", "role", "Main role expansion and role/UID view"},
        {"Security_Control_Assessor_SCA", "SECURITY_CONTROL_ASSESSOR_SCA", "role", "Main role expansion and role/UID view"},
        {"Authorizing_Official_Designated_Representative_AODR", "AUTHORIZING_OFFICIAL_DESIGNATED_REPRESENTATIVE_AODR", "role", "Outside main/role-UID projection"},
        {"Package_Acronym", "PACKAGE_ACRONYM", "text", "BaseTable path"},
        {"_Compliant", "PCT_COMPLIANT", "number", "Outside main/role-UID projection"},
        {"HRC_Related_Package_Acronyms", "HRC_RELATED_PACKAGE_ACRONYMS", "text", "BaseTable path"},
        {"SAP_ID_RTN", "SAP_ID_RTN", "text", "Main and role/UID views"},
        {"Count_of_Controls_with_Open_POAMs", "COUNT_OF_CONTROLS_WITH_OPEN_POAMS", "number", "Outside main/role-UID projection"},
        {"Additional_Visibility", "ADDITIONAL_VISIBILITY", "role", "Outside main/role-UID projection"},
        {"Alternate_Information_System_Owner_ISO", null, "role", "Main role expansion and role/UID view"},
        {"Alternate_Security_Control_Assessor_SCA", "ALTERNATE_SECURITY_CONTROL_ASSESSOR_SCA", "role", "Outside main/role-UID projection"},
        {"Alternate_Information_System_Security_Officer_ISSO", "ALTERNATE_INFORMATION_SYSTEM_SECURITY_OFFICER_ISSO", "role", "Main role expansion and role/UID view"},
        {"Alternate_System_Administrator_SA", "ALTERNATE_SYSTEM_ADMINISTRATOR_SA", "role", "Main role expansion and role/UID view"},
        {"Information_System_Security_Manager_ISSM", "INFORMATION_SYSTEM_SECURITY_MANAGER_ISSM", "role", "Outside main/role-UID projection"},
        {"System_Administrator_SA", "SYSTEM_ADMINISTRATOR_SA", "role", "Main role expansion and role/UID view"},
        {"Entity", "ENTITY_NAME", "text", "Main and role/UID views"},
        {"Business", "BUSINESS_NAME", "text", "Main and role/UID views"},
        {"Information_Owner_IO_Name", "INFORMATION_OWNER_IO_NAME_ONLY", "role", "Outside main/role-UID projection"},
        {"Information_Owner_IO_UID", "INFORMATION_OWNER_IO_ID_ONLY", "role", "Outside main/role-UID projection"},
        {"Information_System_Owner_ISO_Name", "INFORMATION_SYSTEM_OWNER_ISO_NAME_ONLY", "role", "BaseTable path"},
        {"Information_System_Owner_ISO_UID", "INFORMATION_SYSTEM_OWNER_ISO_ID_ONLY", "role", "Role/UID view"},
        {"Senior_Information_Systems_Security_Officer_SISSO_Name", "SENIOR_INFORMATION_SYSTEMS_SECURITY_OFFICER_SISSO_NAME_ONLY", "role", "Outside main/role-UID projection"},
        {"Senior_Information_Systems_Security_Officer_SISSO_UID", "SENIOR_INFORMATION_SYSTEMS_SECURITY_OFFICER_SISSO_ID_ONLY", "role", "Outside main/role-UID projection"},
        {"Authorizing_Official_AO_Name", "AUTHORIZING_OFFICIAL_AO_NAME_ONLY", "role", "BaseTable path"},
        {"Authorizing_Official_AO_UID", "AUTHORIZING_OFFICIAL_AO_ID_ONLY", "role", "Role/UID view"},
        {"Information_System_Security_Officer_ISSO_Name", "INFORMATION_SYSTEM_SECURITY_OFFICER_ISSO_NAME_ONLY", "role", "BaseTable path"},
        {"Information_System_Security_Officer_ISSO_UID", "INFORMATION_SYSTEM_SECURITY_OFFICER_ISSO_ID_ONLY", "role", "Role/UID view"},
        {"Security_Control_Assessor_SCA_Name", "SECURITY_CONTROL_ASSESSOR_SCA_NAME_ONLY", "role", "BaseTable path"},
        {"Security_Control_Assessor_SCA_UID", "SECURITY_CONTROL_ASSESSOR_SCA_ID_ONLY", "role", "Role/UID view"},
        {"Authorizing_Official_Designated_Representative_AODR_Name", "AUTHORIZING_OFFICIAL_DESIGNATED_REPRESENTATIVE_AODR_NAME_ONLY", "role", "Outside main/role-UID projection"},
        {"Authorizing_Official_Designated_Representative_AODR_UID", "AUTHORIZING_OFFICIAL_DESIGNATED_REPRESENTATIVE_AODR_ID_ONLY", "role", "Outside main/role-UID projection"},
        {"Additional_Visibility_Name", "ADDITIONAL_VISIBILITY_NAME_ONLY", "role", "Outside main/role-UID projection"},
        {"Additional_Visibility_UID", "ADDITIONAL_VISIBILITY_ID_ONLY", "role", "Outside main/role-UID projection"},
        {"Alternate_Information_System_Owner_ISO_Name", null, "role", "Inventory only"},
        {"Alternate_Information_System_Owner_ISO_UID", null, "role", "Role/UID view"},
        {"Alternate_Security_Control_Assessor_SCA_Name", "ALTERNATE_SECURITY_CONTROL_ASSESSOR_SCA_NAME_ONLY", "role", "Outside main/role-UID projection"},
        {"Alternate_Security_Control_Assessor_SCA_UID", "ALTERNATE_SECURITY_CONTROL_ASSESSOR_SCA_ID_ONLY", "role", "Outside main/role-UID projection"},
        {"Alternate_Information_System_Security_Officer_ISSO_Name", "ALTERNATE_INFORMATION_SYSTEM_SECURITY_OFFICER_ISSO_NAME_ONLY", "role", "Outside main/role-UID projection"},
        {"Alternate_Information_System_Security_Officer_ISSO_UID", "ALTERNATE_INFORMATION_SYSTEM_SECURITY_OFFICER_ISSO_ID_ONLY", "role", "Role/UID view"},
        {"Alternate_System_Administrator_SA_Name", "ALTERNATE_SYSTEM_ADMINISTRATOR_SA_NAME_ONLY", "role", "Outside main/role-UID projection"},
        {"Alternate_System_Administrator_SA_UID", "ALTERNATE_SYSTEM_ADMINISTRATOR_SA_ID_ONLY", "role", "Role/UID view"},
        {"Information_System_Security_Manager_ISSM_Name", "INFORMATION_SYSTEM_SECURITY_MANAGER_ISSM_NAME_ONLY", "role", "Outside main/role-UID projection"},
        {"Information_System_Security_Manager_ISSM_UID", "INFORMATION_SYSTEM_SECURITY_MANAGER_ISSM_ID_ONLY", "role", "Outside main/role-UID projection"},
        {"System_Administrator_SA_Name", "SYSTEM_ADMINISTRATOR_SA_NAME_ONLY", "role", "BaseTable path"},
        {"System_Administrator_SA_UID", "SYSTEM_ADMINISTRATOR_SA_ID_ONLY", "role", "Role/UID view"},
        {"ConfirmedInArcher", "ARCHER_CONTENT_AUTHORIZATION_PACKAGE_CONFIRMED_IN_ARCHER", "date", "BaseTable path"},
        {"Count_of_Controls", null, "number", "Main and role/UID views"},
        {"Baseline_Recommendation", null, "text", "Main and role/UID views"},
        {"Control_Set_Version_Number", null, "text", "Main and role/UID views"},
        {"Stub_Entry_RTN", null, "text", "Main and role/UID views"},
        {"SAP_Inactive", null, "text", "Main and role/UID views"},
        {"In_Continuous_Monitoring", "IN_CONTINUOUS_MONITORING", "text", "Main and role/UID views"},
        {"Authorization_Package_Source", null, "text", "Main and role/UID views"},
        {"First_Published", "ARCHER_CONTENT_AUTHORIZATION_PACKAGE_FIRST_PUBLISHED", "date", "Inventory only"}
    },

    // Each record carries the original CSV cells for one shared, unique Content_ID.
    PairedRecords = List.Buffer(Table.ToRecords(UniquePairs)),
    PairedCount = List.Count(PairedRecords),
    SqlFieldNames = List.Buffer(Table.ColumnNames(SqlInput)),
    SfFieldNames = List.Buffer(Table.ColumnNames(SfInput)),

    LooksMissing = (cell as nullable text) as logical =>
        if cell = null then true else Text.Trim(cell) = "" or Text.Upper(Text.Trim(cell), "en-US") = "NULL",

    CompareMappedField = (mapping as list) as list =>
        let
            sqlName = mapping{0},
            sfName = mapping{1},
            fieldKind = mapping{2},
            scope = mapping{3},
            sqlHeader = Text.Upper(Text.Trim(sqlName), "en-US"),
            sfHeader = if sfName = null then null else Text.Upper(Text.Trim(sfName), "en-US"),
            hasSql = List.Contains(SqlFieldNames, sqlHeader),
            hasSf = sfHeader <> null and List.Contains(SfFieldNames, sfHeader),
            hasSameNameSf = List.Contains(SfFieldNames, sqlHeader),
            fieldSection =
                if fieldKind = "date" then "05 Dates"
                else if fieldKind = "number" then "06 Numbers"
                else if fieldKind = "role" then "07 Roles"
                else "04 Fields",
            issue =
                if sfHeader = null then
                    if hasSameNameSf then "Same-name Snowflake column found. Confirm the mapping before comparing."
                    else "No mapped Snowflake column. The SQL name is also absent from this Snowflake export."
                else if not hasSf then "Snowflake CSV is missing " & sfName & "."
                else null,
            headerIssues = List.RemoveNulls({
                if hasSql then null else "SQL CSV is missing " & sqlName & ".",
                issue
            }),
            result =
                if not List.IsEmpty(headerIssues) then
                    {"08 Missing or unmapped", sqlName, null, null, 0, null, scope & ". " & Text.Combine(headerIssues, " ")}
                else if PairedCount = 0 then
                    {fieldSection, sqlName, 0, 0, 0, null, "No shared, unique Content_ID to test."}
                else
                    let
                        observations = List.Buffer(List.Transform(PairedRecords, (pair) =>
                            let
                                left = Record.Field(pair[L], sqlHeader),
                                right = Record.Field(pair[R], sfHeader),
                                leftMissing = LooksMissing(left),
                                rightMissing = LooksMissing(right)
                            in
                                [
                                    Id = pair[id], LeftPresent = not leftMissing, RightPresent = not rightMissing,
                                    BothMissing = leftMissing and rightMissing, Diff = left <> right
                                ]
                        )),
                        sqlPopulated = List.Count(List.Select(observations, each [LeftPresent])),
                        sfPopulated = List.Count(List.Select(observations, each [RightPresent])),
                        bothMissing = List.Count(List.Select(observations, each [BothMissing])),
                        differences = List.Buffer(List.Select(observations, each [Diff])),
                        markerDiffs = List.Select(differences, each [BothMissing]),
                        valueDiffs = List.Select(differences, each not [BothMissing]),
                        examples = List.Transform(
                            List.FirstN(List.Combine({valueDiffs, markerDiffs}), 3), each [Id]
                        ),
                        notes = List.RemoveNulls({
                            scope,
                            if bothMissing = 0 then null else "Both missing-looking " & Text.From(bothMissing),
                            if List.IsEmpty(markerDiffs) then null else "Marker-only differences " & Text.From(List.Count(markerDiffs)),
                            if List.IsEmpty(examples) then null else "IDs " & Text.Combine(examples, ", ")
                        })
                    in
                        {fieldSection, sqlName, sqlPopulated, sfPopulated, PairedCount, List.Count(differences),
                         Text.Combine(notes, " | ")}
        in
            result,
    FieldRows = List.Buffer(List.Transform(MappingRows, CompareMappedField)),

    // This section counts fields. The detail sections count package rows.
    FieldSummary = (group as text, label as text) as list =>
        let
            fields = List.Select(FieldRows, each _{0} = group),
            tested = List.Select(fields, each _{4} > 0),
            changed = List.Select(tested, each _{5} > 0),
            noPopulatedPair = List.Select(tested, each _{2} = 0 and _{3} = 0),
            firstChanged = Text.Combine(List.Transform(List.FirstN(changed, 3), each _{1}), ", "),
            missingGroup = group = "08 Missing or unmapped",
            notes = if missingGroup then
                Text.From(List.Count(fields)) & " columns to review. See 08 Missing or unmapped."
                else if List.IsEmpty(tested) then "No fields tested in this group."
                else Text.Combine(List.RemoveNulls({
                    if firstChanged = "" then null else "Examples: " & firstChanged,
                    if List.IsEmpty(noPopulatedPair) then null else
                        Text.From(List.Count(noPopulatedPair)) & " fields had no populated values on either side."
                }), " | ")
        in
            {"Field summary", label, null, null, if missingGroup then null else List.Count(tested),
             if missingGroup or List.IsEmpty(tested) then null else List.Count(changed), notes},

    SummaryRows = {
        FieldSummary("04 Fields", "Package fields and IDs"),
        FieldSummary("05 Dates", "Dates"),
        FieldSummary("06 Numbers", "Control counts and compliance"),
        FieldSummary("07 Roles", "Roles and visibility"),
        FieldSummary("08 Missing or unmapped", "Missing or unmapped columns")
    },
    AllRows = List.Combine({KeyRows, SummaryRows, FieldRows}),
    Output = #table(
        type table [Section = text, Check = text, SQL = nullable number, Snowflake = nullable number,
                    Compared = nullable number, Differences = nullable number, #"Examples or notes" = nullable text],
        AllRows
    )
in
    Output
