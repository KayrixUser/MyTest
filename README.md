let
    SqlFile = "C:\YourFolder\SQL POAMs.csv",
    SnowflakeFile = "C:\YourFolder\Snowflake POAMs.csv",
    Output = "Summary",

    /*
    Optional read-only export queries. Save results WITH column headers.
    SQL Server (ISO text for the four native datetime fields; keep the two source text dates as stored):
    SELECT Content_ID, Last_Updated, POAM_ID,
           CONVERT(varchar(33), First_Published, 126) AS First_Published, Name,
           CONVERT(varchar(33), Milestone_Estimated_Completion_Date, 126) AS Milestone_Estimated_Completion_Date,
           Overall_Status, POAM_Owner, Authorization_Package, hRTN_First_Published,
           POAM_Owner_Name, POAM_Owner_UID, CONVERT(varchar(33), ConfirmedInArcher, 126) AS ConfirmedInArcher,
           CONVERT(varchar(33), POAM_Target_Completion_Date, 126) AS POAM_Target_Completion_Date,
           Related_Allocated_Controls
    FROM ACP_Connection.dbo.ArcherMetrics_POAMs_PROD_All;

    Snowflake:
    SELECT * FROM RTX_ENTERPRISESERVICES.ES_ESC_GRC_PUBLISHED.C2S_POAM;
    */

    // Options: Summary, FieldResults, FieldDifferences, UnmatchedIDs, SnapshotDates.
    Fields = #table({"SQL","SF","Rule"}, {
        {"Content_ID","ARCHER_CONTENT_POAMS_CONTENT_ID","Key"},
        {"POAM_ID","POAM_ID","Text"},
        {"Name","NAME","Text"},
        {"Overall_Status","OVERALL_STATUS","Text"},
        {"Authorization_Package","AUTHORIZATION_PACKAGE_NAME","Text"},
        {"POAM_Owner","POAM_OWNER","Text"},
        {"POAM_Owner_Name","POAM_OWNER_NAME_ONLY","List"},
        {"POAM_Owner_UID","POAM_OWNER_ID_ONLY","List"},
        {"Related_Allocated_Controls","RELATED_ALLOCATED_CONTROLS","Text"},
        {"Last_Updated","ARCHER_CONTENT_POAMS_LAST_UPDATED","Date"},
        {"ConfirmedInArcher","ARCHER_CONTENT_POAMS_CONFIRMED_IN_ARCHER","Date"},
        {"First_Published","ARCHER_CONTENT_POAMS_FIRST_PUBLISHED","Date"},
        {"hRTN_First_Published","HRTN_FIRST_PUBLISHED","Date"},
        {"Milestone_Estimated_Completion_Date","MILESTONE_ESTIMATED_COMPLETION_DATE","Date"},
        {"POAM_Target_Completion_Date","POAM_TARGET_COMPLETION_DATE","Date"}
    }),
    Restored = {"POAM_ID","hRTN_First_Published","POAM_Target_Completion_Date","Related_Allocated_Controls"},
    Clean = (v as nullable text) as nullable text =>
        let s = if v = null then "" else Text.Trim(v)
        in if List.Contains({"","NULL","(NULL)","\N"},Text.Upper(s)) then null else s,

    ReadCsv = (path as text, required as list) as record =>
        let
            bytes = Binary.Buffer(File.Contents(path)),
            bom = if Binary.Length(bytes) < 2 then "" else Text.Upper(Binary.ToText(Binary.Range(bytes,0,2),BinaryEncoding.Hex)),
            enc = if bom = "FFFE" then 1200 else if bom = "FEFF" then 1201 else
                let utf8 = try Text.FromBinary(bytes,65001)
                in if utf8[HasError] then 1252 else if Text.Contains(utf8[Value],Character.FromNumber(65533)) then 1252 else 65001,
            csv = Csv.Document(bytes,[Delimiter=",",Encoding=enc,QuoteStyle=QuoteStyle.Csv]),
            headers = Table.TransformColumnNames(Table.PromoteHeaders(csv,[PromoteAllScalars=true]),
                each Text.Upper(Text.Trim(Text.Replace(_,Character.FromNumber(65279),"")))),
            missing = List.Difference(required,Table.ColumnNames(headers)),
            selected = Table.SelectColumns(headers,required,MissingField.UseNull),
            renamed = Table.RenameColumns(selected,List.Select(List.Zip({required,Fields[SQL]}), each _{0} <> _{1})),
            data = Table.TransformColumns(renamed,List.Transform(Fields[SQL],each {_,Clean,type nullable text}))
        in [Data=Table.Buffer(data),Missing=missing],
    Sql = ReadCsv(SqlFile,List.Transform(Fields[SQL],each Text.Upper(_))),
    Sf = ReadCsv(SnowflakeFile,Fields[SF]),
    A = Sql[Data], B = Sf[Data],
    TotalA = Table.RowCount(A), TotalB = Table.RowCount(B),
    HaveRows = TotalA > 0 and TotalB > 0,
    Has = (col as text) as logical => not List.Contains(Sql[Missing],Text.Upper(col))
        and not List.Contains(Sf[Missing],Fields{[SQL=col]}[SF]),
    KeysReady = Has("Content_ID") and HaveRows,
    N = (v as number) as text => Number.ToText(v,"#,##0","en-US"),
    Sample = (ids as list) as text => if List.IsEmpty(ids) then "none" else Text.Combine(List.FirstN(ids,3),", "),
    Profile = (data as table, col as text, countLabel as text) as table =>
        Table.Buffer(Table.Group(Table.SelectRows(Table.SelectColumns(data,{col}),each Record.Field(_,col) <> null),
            {col},{{countLabel,each Table.RowCount(_),Int64.Type}})),
    CountA = Profile(A,"Content_ID","SQLRows"), CountB = Profile(B,"Content_ID","SFRows"),
    Joined = Table.Join(CountA,{"Content_ID"},Table.RenameColumns(CountB,{{"Content_ID","SFKey"}}),{"SFKey"},JoinKind.FullOuter),
    OnlyA = Table.Buffer(Table.SelectRows(Joined,each [SFRows] = null)),
    OnlyB = Table.Buffer(Table.SelectRows(Joined,each [SQLRows] = null)),
    Shared = Table.SelectRows(Joined,each [SQLRows] <> null and [SFRows] <> null),
    Unique = Table.RenameColumns(Table.SelectColumns(Table.SelectRows(Shared,each [SQLRows]=1 and [SFRows]=1),{"Content_ID"}),{{"Content_ID","JoinKey"}}),
    Linked = Table.Join(Unique,{"JoinKey"},A,{"Content_ID"},JoinKind.Inner),
    Pairs = Table.Buffer(Table.Join(Linked,{"Content_ID"},Table.PrefixColumns(B,"SF"),{"SF.Content_ID"},JoinKind.Inner)),
    SharedN = Table.RowCount(Shared), PairN = Table.RowCount(Pairs),

    // Interpret timezone-free date/time exports in en-US. Unreadable or zoned dates need review.
    AsDate = (s as nullable text) as nullable datetime =>
        if s = null then null else
        let
            tail = if Text.Length(s) > 10 then Text.Range(s,10) else "",
            zoned = Text.EndsWith(Text.Upper(s),"Z") or Text.Contains(tail,"+") or Text.Contains(tail,"-"),
            parsed = if zoned then null else try DateTime.FromText(s,[Culture="en-US"]) otherwise null
        in parsed,
    Stamp = (d as nullable datetime) as nullable text => if d=null then null else DateTime.ToText(d,[Format="yyyy-MM-dd HH:mm:ss.fffffff",Culture="en-US"]),
    Members = (s as text) as list => List.Sort(List.Transform(Text.Split(s,";"),each Text.Trim(_)),Comparer.Ordinal),
    Judge = (a as nullable text,b as nullable text,rule as text) as text =>
        if a=null and b=null then "Both blank"
        else if rule="Date" and ((a<>null and AsDate(a)=null) or (b<>null and AsDate(b)=null)) then "Unreadable date"
        else if a=null or b=null then "Different"
        else if rule="Date" then (if AsDate(a)=AsDate(b) then "Match" else "Different")
        else if rule="List" then (if Members(a)=Members(b) then "Match" else "Different")
        else if a=b then "Match" else "Different",
    // Verify the historical override/fallback independently on each export, on the same calendar day.
    Today = Date.From(DateTime.FixedLocalNow()),
    ChosenDateText = (r as record, prefix as text) as nullable text =>
        let hist = Record.Field(r,prefix & "hRTN_First_Published")
        in if hist=null then Record.Field(r,prefix & "First_Published") else hist,
    DayText = (s as nullable text) as nullable text =>
        if s=null then null else let d=AsDate(s) in if d=null then "[Unreadable date]" else N(Duration.Days(Today-Date.From(d))),
    Effective = Table.AddColumn(Table.AddColumn(Pairs,"Effective first published",each ChosenDateText(_,""),type nullable text),
        "SF.Effective first published",each ChosenDateText(_,"SF."),type nullable text),
    Derived = Table.AddColumn(Table.AddColumn(Effective,"DaysOpen at same cutoff",each DayText([Effective first published]),type nullable text),
        "SF.DaysOpen at same cutoff",each DayText([#"SF.Effective first published"]),type nullable text),
    DerivedReady = Has("hRTN_First_Published") and Has("First_Published"),
    DerivedFields = {"Effective first published","DaysOpen at same cutoff"},
    Spec = List.Transform(Table.ToRecords(Table.SelectRows(Fields,each [Rule]<>"Key")),
        each [Name=_[SQL],Rule=_[Rule],Available=Has(_[SQL])])
        & {[Name=DerivedFields{0},Rule="Date",Available=DerivedReady],[Name=DerivedFields{1},Rule="Days",Available=DerivedReady]},
    Compare = (f as record) as record =>
        let
            col=f[Name], source=if List.Contains(DerivedFields,col) then Derived else Pairs,
            selected=Table.RenameColumns(Table.SelectColumns(source,{"Content_ID",col,"SF." & col}),{{col,"SQL value"},{"SF." & col,"Snowflake value"}}),
            checked=Table.Buffer(Table.AddColumn(selected,"Finding",each
                if f[Rule]="Days" and List.Contains({[SQL value],[Snowflake value]},"[Unreadable date]") then "Unreadable date"
                else Judge([SQL value],[Snowflake value],f[Rule]),type text)),
            ready=f[Available] and KeysReady and PairN>0,
            issues=Table.AddColumn(Table.SelectRows(checked,each [Finding]="Different" or [Finding]="Unreadable date"),"Field",each col,type text),
            readableDiff=Table.RowCount(Table.SelectRows(issues,each [Finding]="Different")),
            bad=Table.RowCount(issues)-readableDiff,
            pa=List.NonNullCount(checked[SQL value]), pb=List.NonNullCount(checked[Snowflake value]),
            status=if ready and pa+pb>0 and readableDiff+bad=0 then "Pass" else "Needs Review",
            note=if not f[Available] then "Missing required header."
                else if not KeysReady or PairN=0 then "No unique shared records to compare."
                else if pa+pb=0 then "Blank on both sides for every shared record."
                else N(readableDiff) & " different" & (if bad=0 then "" else ", " & N(bad) & " unreadable")
                    & (if Table.IsEmpty(issues) then "." else ". Example Content_IDs: " & Sample(issues[Content_ID]) & ".")
        in [Field=col,Result=status,Compared=if ready then PairN else null,SQLPopulated=if ready then pa else null,
            SFPopulated=if ready then pb else null,Differences=if ready then readableDiff else null,Unreadable=if ready then bad else null,
            Evidence=note,Details=if ready then issues else Table.FirstN(issues,0)],
    Stats=Table.Buffer(Table.FromRecords(List.Transform(Spec,Compare))),
    FieldResults=Table.RemoveColumns(Stats,{"Details"}),
    FieldDifferences=Table.ReorderColumns(Table.Combine(Stats[Details]),{"Field","Content_ID","SQL value","Snowflake value","Finding"}),
    UnmatchedIDs=if not KeysReady then error "Both exports need data and Content_ID headers to list unmatched IDs." else Table.Sort(Table.Combine({
        Table.AddColumn(Table.SelectColumns(OnlyA,{"Content_ID"}),"Found in",each "SQL Server only",type text),
        Table.AddColumn(Table.RenameColumns(Table.SelectColumns(OnlyB,{"SFKey"}),{{"SFKey","Content_ID"}}),"Found in",each "Snowflake only",type text)
    }),{{"Found in",Order.Ascending},{"Content_ID",Order.Ascending}}),

    Grade=(ok as logical) as text => if ok then "Pass" else "Needs Review",
    Row=(label as text,id as text,result as text,evidence as text,action as text) as list => {label,id,result,evidence,action,""},
    FieldRow=(label as text,id as text,cols as list,action as text) as list =>
        let relevant=Table.SelectRows(Stats,each List.Contains(cols,[Field])),
            note=Text.Combine(List.Transform(Table.ToRecords(relevant),each _[Field] & ": " & _[Evidence])," ")
        in Row(label,id,Grade(List.AllTrue(List.Transform(relevant[Result],each _="Pass"))),note,action),
    KeyRow=(col as text,id as text,x as table,y as table) as list =>
        let
            aCount=List.Sum({0} & x[SQLRows]), bCount=List.Sum({0} & y[SFRows]),
            missA=TotalA-aCount, missB=TotalB-bCount, dupA=aCount-Table.RowCount(x), dupB=bCount-Table.RowCount(y)
        in Row(col & " completeness and duplicates",id,Grade(HaveRows and Has(col) and missA+missB+dupA+dupB=0),
            if not Has(col) then "Header missing. See required fields." else
            "SQL: " & N(Table.RowCount(x)) & " distinct, " & N(missA) & " blank, " & N(dupA) & " extra duplicate rows. Snowflake: "
            & N(Table.RowCount(y)) & " distinct, " & N(missB) & " blank, " & N(dupB) & " extra duplicate rows.","IDs are counted as text."),
    Missing=(names as list) as text => if List.IsEmpty(names) then "none" else Text.Combine(names,", "),
    RestoredReady=List.Select(Restored,each Has(_)),
    DateProfile=(data as table,col as text) as record =>
        let raw=List.RemoveNulls(Table.Column(data,col)), parsed=List.RemoveNulls(List.Transform(raw,AsDate))
        in [Latest=if List.IsEmpty(parsed) then "(none readable)" else Stamp(List.Max(parsed)),Bad=List.Count(raw)-List.Count(parsed)],
    LatestCheck=(col as text,id as text) as list =>
        let x=DateProfile(A,col),y=DateProfile(B,col),
            ok=Has(col) and x[Bad]+y[Bad]=0 and x[Latest]<>"(none readable)" and x[Latest]=y[Latest]
        in Row("Latest " & col,id,Grade(ok),
            "SQL: " & x[Latest] & ". Snowflake: " & y[Latest] & ". Unreadable populated dates (SQL / Snowflake): " & N(x[Bad]) & " / " & N(y[Bad]) & ".",
            "Maximum across each full export. Use the refresh logs to confirm the actual cutoff."),
    SnapshotCounts=(data as table,side as text) as table =>
        let dates=Table.AddColumn(Table.SelectColumns(data,{"ConfirmedInArcher"}),"Confirmation day",each
                if [ConfirmedInArcher]=null then "(blank)" else let d=AsDate([ConfirmedInArcher]) in
                if d=null then "(unreadable)" else Date.ToText(Date.From(d),[Format="yyyy-MM-dd",Culture="en-US"]),type text),
            grouped=Table.Group(dates,{"Confirmation day"},{{"Rows",each Table.RowCount(_),Int64.Type}})
        in Table.AddColumn(grouped,"Source",each side,type text),
    SnapshotDates=if not Has("ConfirmedInArcher") then error "Both exports need their confirmation-date headers for this view." else Table.Sort(Table.Combine({SnapshotCounts(A,"SQL Server"),SnapshotCounts(B,"Snowflake")}),{{"Source",Order.Ascending},{"Confirmation day",Order.Descending}}),

    OnlyARows=Table.Join(Table.RenameColumns(Table.SelectColumns(OnlyA,{"Content_ID"}),{{"Content_ID","JoinKey"}}),{"JoinKey"},A,{"Content_ID"},JoinKind.Inner),
    BlankOnSQLOnly=(col as text) as text => if not KeysReady or List.Contains(Sql[Missing],Text.Upper(col)) then "unknown" else
        N(Table.RowCount(Table.SelectRows(OnlyARows,each Record.Field(_,col)=null))),
    Checks={
        Row("Required POAM fields","PO01",Grade(List.IsEmpty(Sql[Missing]) and List.IsEmpty(Sf[Missing])),
            "Expected 15. SQL missing: " & Missing(Sql[Missing]) & ". Snowflake missing: " & Missing(Sf[Missing]) & ".","Compare the S5 PROD_All export to the published C2S_POAM view."),
        Row("Previously missing fields present","PO02",Grade(List.Count(RestoredReady)=4),
            N(List.Count(RestoredReady)) & " of 4 present on both sides. Still missing on either side: " & Missing(List.Difference(Restored,RestoredReady)) & ".",
            "Value checks below cover the POAM ID, historical publish date, target date and related controls."),
        Row("Total POAM records","PO03",Grade(HaveRows and TotalA=TotalB),"SQL Server: " & N(TotalA) & ". Snowflake: " & N(TotalB) & ".","Both full exports, before the C2S status filter. SnapshotDates lists rows by confirmation day if one side contains more history."),
        Row("Same POAMs in both files","PO04",Grade(KeysReady and Table.IsEmpty(OnlyA) and Table.IsEmpty(OnlyB) and PairN=TotalA and PairN=TotalB and PairN>0),
            if not KeysReady then "Need both populated exports and their Content_ID headers." else
            N(SharedN) & " IDs in both. " & N(Table.RowCount(OnlyA)) & " SQL only (" & Sample(OnlyA[Content_ID]) & "). "
            & N(Table.RowCount(OnlyB)) & " Snowflake only (" & Sample(OnlyB[SFKey]) & "). " & N(PairN) & " shared IDs occur once on each side and were used for field checks.",
            "SQL-only rows with blank raw owner: " & BlankOnSQLOnly("POAM_Owner") & "; with blank SQL package: " & BlankOnSQLOnly("Authorization_Package")
            & ". The groups may overlap. Check both inner joins in Snowflake for unmatched lookups."),
        KeyRow("Content_ID","PO05",CountA,CountB),
        KeyRow("POAM_ID","PO06",Profile(A,"POAM_ID","SQLRows"),Profile(B,"POAM_ID","SFRows")),
        FieldRow("POAM ID on the same record","PO07",{"POAM_ID"},"Match by Content_ID, then compare POAM_ID. SAP_ID identifies a package."),
        FieldRow("POAM name and status","PO08",{"Name","Overall_Status"},"Compare the full source population. Check the downstream Completed/NULL filter separately."),
        FieldRow("Authorization package","PO09",{"Authorization_Package"},"Compare SQL package text with the resolved Snowflake package names. Exact text matters to the existing equality join."),
        FieldRow("POAM owner fields","PO10",{"POAM_Owner","POAM_Owner_Name","POAM_Owner_UID"},"Compare raw owner text separately from names and UIDs. Names/UIDs use semicolon lists, ignoring order and retaining duplicates; confirm owner pairing for multi-owner records."),
        FieldRow("Related allocated controls","PO11",{"Related_Allocated_Controls"},"Compare the restored controls field on the same POAM."),
        LatestCheck("Last_Updated","PO12"),
        LatestCheck("ConfirmedInArcher","PO13"),
        FieldRow("Update and confirmation on matched records","PO14",{"Last_Updated","ConfirmedInArcher"},"Inspect unreadable dates separately from actual value differences."),
        FieldRow("Original and historical publish dates","PO15",{"First_Published","hRTN_First_Published"},"Compare each field separately. A blank historical date can be expected."),
        FieldRow("Milestone and target completion dates","PO16",{"Milestone_Estimated_Completion_Date","POAM_Target_Completion_Date"},"A milestone date and a POAM target date are separate checks."),
        FieldRow("Effective publish date and DaysOpen","PO17",DerivedFields,"Recalculated using the historical date when present, otherwise First_Published. Calendar-day cutoff: "
            & Date.ToText(Today,[Format="yyyy-MM-dd",Culture="en-US"]) & ". Verify the actual C2S output at its load date.")
    },
    Summary=#table({"Check","Test ID","Result","Evidence","Details or Action","Comments(Grady)"},
        Checks & {Row("POAM export comparison","PO18",Grade(List.AllTrue(List.Transform(Checks,each _{2}="Pass"))),
            "Compared fields on " & N(PairN) & " unique shared Content_IDs. All-blank fields and unreadable dates require review.",
            "For downstream sign-off, confirm the Matillion/C2S refresh order, owner/package join eligibility, status filter, dimPOAMs, factCyberEvents package links and the separate S6 BaseTable path.")}),
    Report=if Output="Summary" then Summary else if Output="FieldResults" then FieldResults
        else if Output="FieldDifferences" then FieldDifferences else if Output="UnmatchedIDs" then UnmatchedIDs
        else if Output="SnapshotDates" then SnapshotDates else error "Choose Summary, FieldResults, FieldDifferences, UnmatchedIDs, or SnapshotDates."
in
    Report
