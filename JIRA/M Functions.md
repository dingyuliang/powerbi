## GetScrumSprintReport
```
let
    GetScrumSprintReport = (url as text, boardId as number, sprintId as number, optional update_issues as any) =>
    let
        update_issues = if update_issues = true then true else false, 
        // Step 1: get sprint issues from jira sprint report - no 50 limitation
        // https://dingyuliang.atlassian.net/rest/greenhopper/latest/rapid/charts/sprintreport?rapidViewId=1&sprintId=1
        rawContents = Web.Contents(url&"/rest/greenhopper/latest/rapid/charts/sprintreport", [Query = [sprintId = Number.ToText(sprintId), rapidViewId = Number.ToText(boardId)], Headers=[#"Accept" = "application/json", #"Authorization"="Bearer "&JIRAPersonalAccessToken]]),

        /*
        {"contents":{"completedIssues":[{"id":10000,"key":"POW-1","hidden":false,"typeName":"Story","typeId":"10001","typeHierarchyLevel":0,"summary":"Test User Story","typeUrl":"https://dingyuliang.atlassian.net/rest/api/2/universal_avatar/view/type/issuetype/avatar/10315?size=medium","priorityUrl":"https://dingyuliang.atlassian.net/images/icons/priorities/medium.svg","priorityName":"Medium","done":true,"assignee":"60b3df488cf91900693ad834","assigneeKey":"60b3df488cf91900693ad834","assigneeAccountId":"60b3df488cf91900693ad834","assigneeName":"Yuliang Ding","avatarUrl":"https://secure.gravatar.com/avatar/a33656f903cb26c5534705e783f23887?d=https%3A%2F%2Favatar-management--avatars.us-west-2.prod.public.atl-paas.net%2Finitials%2FYD-0.png","hasCustomUserAvatar":false,"flagged":false,"currentEstimateStatistic":{"statFieldId":"customfield_10030","statFieldValue":{"value":8.0}},"estimateStatisticRequired":false,"estimateStatistic":{"statFieldId":"customfield_10030","statFieldValue":{"value":5.0}},"statusId":"10001","statusName":"Done","statusUrl":"https://dingyuliang.atlassian.net/","status":{"id":"10001","name":"Done","description":"","iconUrl":"https://dingyuliang.atlassian.net/","statusCategory":{"id":"3","key":"done","colorName":"green"}},"fixVersions":[],"projectId":10000,"linkedPagesCount":0}],"issuesNotCompletedInCurrentSprint":[{"id":10001,"key":"POW-2","hidden":false,"typeName":"Story","typeId":"10001","typeHierarchyLevel":0,"summary":"Test User Story - Added New","typeUrl":"https://dingyuliang.atlassian.net/rest/api/2/universal_avatar/view/type/issuetype/avatar/10315?size=medium","priorityUrl":"https://dingyuliang.atlassian.net/images/icons/priorities/medium.svg","priorityName":"Medium","done":true,"hasCustomUserAvatar":false,"flagged":false,"currentEstimateStatistic":{"statFieldId":"customfield_10030","statFieldValue":{"value":2.0}},"estimateStatisticRequired":false,"estimateStatistic":{"statFieldId":"customfield_10030","statFieldValue":{"value":2.0}},"statusId":"10000","statusName":"To Do","statusUrl":"https://dingyuliang.atlassian.net/","status":{"id":"10000","name":"To Do","description":"","iconUrl":"https://dingyuliang.atlassian.net/","statusCategory":{"id":"2","key":"new","colorName":"blue-gray"}},"fixVersions":[],"projectId":10000,"linkedPagesCount":0}],"puntedIssues":[],"issuesCompletedInAnotherSprint":[],"completedIssuesInitialEstimateSum":{"value":5.0,"text":"5.0"},"completedIssuesEstimateSum":{"value":8.0,"text":"8.0"},"issuesNotCompletedInitialEstimateSum":{"value":2.0,"text":"2.0"},"issuesNotCompletedEstimateSum":{"value":2.0,"text":"2.0"},"allIssuesEstimateSum":{"value":12.0,"text":"12.0"},"puntedIssuesInitialEstimateSum":{"text":"null"},"puntedIssuesEstimateSum":{"text":"null"},"issuesCompletedInAnotherSprintInitialEstimateSum":{"text":"null"},"issuesCompletedInAnotherSprintEstimateSum":{"text":"null"},"issueKeysAddedDuringSprint":{"POW-2":true}},"sprint":{"id":1,"sequence":1,"name":"POW Sprint 1","state":"CLOSED","linkedPagesCount":0,"goal":"","startDate":"28/Jul/22 7:49 AM","endDate":"29/Jul/22 7:49 AM","isoStartDate":"2022-07-28T07:49:36+0000","isoEndDate":"2022-07-29T07:49:00+0000","completeDate":"28/Jul/22 7:50 AM","isoCompleteDate":"2022-07-28T07:50:49+0000","canUpdateSprint":true,"remoteLinks":[],"daysRemaining":0},"lastUserToClose":"<a class=\"user-hover\" rel=\"60b3df488cf91900693ad834\" id=\"_60b3df488cf91900693ad834\" data-user=\"{&quot;accountId&quot;: &quot;60b3df488cf91900693ad834&quot;}\" href=\"/secure/ViewProfile.jspa?accountId=60b3df488cf91900693ad834\">Yuliang Ding</a>","supportsPages":false}
        */
        json = Json.Document(rawContents),
        
        contents = json[contents],
        sprint = json[sprint],

        completed_issues_0 = contents[completedIssues],
        issuesNotCompletedInCurrentSprint_0 = contents[issuesNotCompletedInCurrentSprint],
        puntedIssues_0 = contents[puntedIssues],
        issuesCompletedInAnotherSprint_0 = contents[issuesCompletedInAnotherSprint],
        
        #"all issues"  = List.Combine({completed_issues_0, issuesNotCompletedInCurrentSprint_0, puntedIssues_0, issuesCompletedInAnotherSprint_0}), // this include added_issues
        #"all issues typed" = Table.FromRecords(#"all issues" , type table[ id =  Int32.Type, key = Text.Type, typeId =  Text.Type, summary =  Any.Type, currentEstimateStatistic = Record.Type, estimateStatistic = Record.Type]),
        #"issue id" = Table.AddColumn(#"all issues typed", "issue_id", each [id]),
        #"sprint id" = Table.AddColumn(#"issue id", "sprint_id", each sprintId),
        #"board id" = Table.AddColumn(#"sprint id", "board_id", each boardId),

        #"original estimate" = Table.AddColumn(#"board id", "initial_estimate", each try Double.From([estimateStatistic][statFieldValue][value]) otherwise 0.0),
        #"current estimate" = Table.AddColumn(#"original estimate", "current_estimate", each try Double.From([currentEstimateStatistic][statFieldValue][value]) otherwise 0.0),
        all_issues_record_table = #"current estimate",
        all_issues_record_list = Table.ToRecords(all_issues_record_table),
        added_issues_table = Table.Join(all_issues_record_table, 
                                                            "key",
                                                                Table.FromList(List.Select(
                                                                                        Record.FieldNames(contents[issueKeysAddedDuringSprint]), 
                                                                                        each Record.Field(contents[issueKeysAddedDuringSprint], _) = true
                                                                                    ), 
                                                                            null, 
                                                                            {"issue_key"}
                                                                            ), 
                                                                "issue_key",
                                                                JoinKind.Inner),
        added_issues = Table.ToRecords(added_issues_table),

        completed_issues = List.Select(all_issues_record_list, each List.Contains(List.Transform(completed_issues_0, each [key]), [key])),
        issuesNotCompletedInCurrentSprint = List.Select(all_issues_record_list, each List.Contains(List.Transform(issuesNotCompletedInCurrentSprint_0, each [key]), [key])),
        puntedIssues = List.Select(all_issues_record_list, each List.Contains(List.Transform(puntedIssues_0, each [key]), [key])),
        issuesCompletedInAnotherSprint = List.Select(all_issues_record_list, each List.Contains(List.Transform(issuesCompletedInAnotherSprint_0, each [key]), [key])),

        // Committed -> planned_points -> (get_all_issues - added_issues).original_estimate
        // Delivered -> completed_points -> (completed_issues +  issuesCompletedInAnotherSprint).current_estimate
        // Not Delivered -> incomplete_points -> issuesNotCompletedInCurrentSprint.current_estimate
        // Removed -> removed_points -> puntedIssues.current_estimate
        // Added -> added_points -> added_issues.initial_estimate
        // Changed -> changed_points -> (completed_issues + completed_outside_issues + incomplete_issues + puntedIssues).(current_estimate-initial_estimate)

        // Committed + Added + Changed = Delivered + Not Delivered + Removed
        
        /*
        completed_issues = contents[completedIssuesInitialEstimateSum],
        completed_issues = contents[completedIssuesEstimateSum],
        completed_issues = contents[issuesNotCompletedInitialEstimateSum],
        completed_issues = contents[issuesNotCompletedEstimateSum],
        completed_issues = contents[allIssuesEstimateSum],
        completed_issues = contents[puntedIssuesInitialEstimateSum],
        completed_issues = contents[puntedIssuesEstimateSum],
        completed_issues = contents[issuesCompletedInAnotherSprintInitialEstimateSum],
        completed_issues = contents[issuesCompletedInAnotherSprintEstimateSum],
        completed_issues = contents[issueKeysAddedDuringSprint],
        */

        #"Initial Value" = [ 
                    board_id = boardId,
                    sprint_id = sprintId, 
                    sequence = sprint[sequence], 
                    name = sprint[name], 
                    state = sprint[state], 
                    iso_start_date = DateTime.FromText(sprint[startDate]), 
                    iso_end_date = DateTime.FromText(sprint[endDate]),
                    completed_initial_estimate = try contents[completedIssuesInitialEstimateSum][value] otherwise 0.0,
                    completed_estimate = try contents[completedIssuesEstimateSum][value] otherwise 0.0,
                    completed_issues_count = List.Count(contents[completedIssues]),
                    incompleted_initial_estimate = try contents[issuesNotCompletedInitialEstimateSum][value] otherwise 0.0,
                    incompleted_estimate = try contents[issuesNotCompletedEstimateSum][value] otherwise 0.0,
                    incompleted_issues_count = List.Count(contents[issuesNotCompletedInCurrentSprint]),
                    completed_outside_sprint_initial_estimate = try contents[issuesCompletedInAnotherSprintInitialEstimateSum][value] otherwise 0.0,
                    completed_outside_sprint_estimate = try contents[issuesCompletedInAnotherSprintEstimateSum][value] otherwise 0.0,
                    completed_outside_sprint_issues_count = List.Count(contents[issuesCompletedInAnotherSprint]),
                    removed_from_sprint_initial_estimate = try contents[puntedIssuesInitialEstimateSum][value] otherwise 0.0,
                    removed_from_sprint_estimate = try contents[puntedIssuesEstimateSum][value] otherwise 0.0,
                    removed_from_sprint_issues_count =List.Count(contents[puntedIssues]),
                    added_during_sprint_initial_estimate = try List.Accumulate(added_issues, 0.0, (state, current) => state + current[initial_estimate])
                                                            otherwise 0.0,
                    added_during_sprint_estimate = try List.Accumulate(added_issues, 0.0, (state, current) => state + current[current_estimate])
                                                            otherwise 0.0,
                    /*
                    added_during_sprint_estimate = 
                                                       Table.Join(Table.FromList(all_issues), 
                                                       "key",
                                                        Table.FromList(List.Select(
                                                                                Record.FieldNames(contents[issueKeysAddedDuringSprint]), 
                                                                                each Record.Field(contents[issueKeysAddedDuringSprint], _) = true
                                                                            ), 
                                                                    null, 
                                                                    {"issue_key"}
                                                                    ), 
                                                        "issue_key",
                                                        JoinKind.Inner), */
                    added_during_sprint_issues_count = try List.Count(List.Select(Record.FieldNames(contents[issueKeysAddedDuringSprint]), each Record.Field(contents[issueKeysAddedDuringSprint], _) = true))
                                                            otherwise 0,
                    changed_during_sprint_initial_estimate = 0.0,
                    changed_during_sprint_estimate = 0.0,
                    changed_during_sprint_issues_count = 0
                ],
        
        // Committed -> planned_points -> (get_all_issues - added_issues).initial_estimate
        
        #"Committed Points" = try Record.AddField(#"Initial Value", "committed_points", List.Accumulate(
                                                                                                        List.Select(all_issues_record_list, each List.Contains(List.Transform(added_issues, each [key]), [key]) = false ),
                                                                                                        0.0,
                                                                                                        (state, current) => state + current[initial_estimate]
                                                                                                    )
                                             ) 
                              otherwise [],
        //#"Committed Points" = #"Initial Value",
        // Added -> added_points -> added_issues.initial_estimate
        #"Added Points" = try Record.AddField(#"Committed Points", "added_points", List.Accumulate(
                                                                                                    added_issues,
                                                                                                    0.0,
                                                                                                    (state, current) => state + current[initial_estimate]
                                                                                                )
                                           )
                              otherwise [],
        // Changed -> changed_points -> (completed_issues + completed_outside_issues + incomplete_issues + puntedIssues).(current_estimate-initial_estimate)
        #"Changed Points" = try Record.AddField(#"Added Points", "changed_points",  List.Accumulate(
                                                                                                    all_issues_record_list,
                                                                                                    0.0,
                                                                                                    (state, current) => state + current[current_estimate] - current[initial_estimate]
                                                                                                )
                                           )
                              otherwise [],
        // Delivered -> completed_points -> (completed_issues +  issuesCompletedInAnotherSprint).current_estimate
        #"Delivered Points" = try Record.AddField(#"Changed Points", "delivered_points", List.Accumulate(
                                                                                                    List.Combine({completed_issues, issuesCompletedInAnotherSprint}),
                                                                                                    0.0,
                                                                                                    (state, current) => state + current[current_estimate]
                                                                                                )
                                           )
                              otherwise [],
        // Not Delivered -> incomplete_points -> issuesNotCompletedInCurrentSprint.current_estimate
        #"Not Delivered Points" = try Record.AddField(#"Delivered Points", "not_delivered_points", List.Accumulate(
                                                                                                    issuesNotCompletedInCurrentSprint,
                                                                                                    0.0,
                                                                                                    (state, current) => state + current[current_estimate]
                                                                                                )
                                           )
                              otherwise [],
        // Removed -> removed_points -> puntedIssues.current_estimate        
        #"Removed Points" = Record.AddField(#"Not Delivered Points", "removed_points", List.Accumulate(
                                                                                                    puntedIssues,
                                                                                                    0.0,
                                                                                                    (state, current) => state + current[current_estimate]
                                                                                                )
                                           ),

                    
        // ToDo: if sprint has more than 50 issues, then we will have issue, we need to implement pagination
        // Step 2: get sprint issues detailed data
        // https://developer.atlassian.com/cloud/jira/software/rest/api-group-sprint/#api-rest-agile-1-0-sprint-sprintid-issue-get
        // https://a.jira.com/rest/agile/1.0/sprint/5509/issue?fields=id,key,customfield_16052,customfield_15351,customfield_15155
        issues_details_content = Web.Contents(url&"/rest/agile/1.0/sprint/"&Text.From(sprintId)&"/issue", [Query = [ 
                                                                                            startAt = "0",
                                                                                            maxResults = "100",
                                                                                            fields = {
                                                                                                "id", 
                                                                                                "key",
                                                                                                "assignee",           // assignee.name, emailAddress, key, displayName
                                                                                                "priority",           // priority.name
                                                                                                "customfield_11350",  // epic id
                                                                                                "customfield_15351",  // value stream  ->.value
                                                                                                "customfield_15155",  // granicus product is an array ->.value
                                                                                                "customfield_11850"   // scrum team -> .value
                                                                                                }], 
                                                                                      Headers=[#"Accept" = "application/json", #"Authorization"="Bearer "&JIRAPersonalAccessToken]]),
         
        issues_details_json = Json.Document(issues_details_content),

        #"Detailed Issues" = Table.FromRecords(issues_details_json[issues], type table[ id =  Int32.Type, key = Text.Type, fields = Any.Type ]),
        #"Detailed Issues Renamed" = Table.RenameColumns(
                                     #"Detailed Issues",
                                     {
                                         {"id","right_id"},
                                         {"key", "right_key"}
                                         //{"customfield_11350", "epic_id"},
                                         //{"customfield_15351", "value_stream"},
                                         //{"customfield_15155", "product"},
                                         //{"customfield_11850", "scrum_team"}
                                     }
                                ),
        #"Epic ID Column" = Table.AddColumn(#"Detailed Issues Renamed", "epic_key", each try [fields][customfield_11350] otherwise null),
        #"Value Stream Column" = Table.AddColumn(#"Epic ID Column", "value_stream", each try [fields][customfield_15351][value] otherwise null),
        #"Sub Value Stream Column" = Table.AddColumn(#"Value Stream Column", "sub_value_stream", each try [fields][customfield_15351][child][value] otherwise null),
        #"Product Column" = Table.AddColumn(#"Sub Value Stream Column", "product", each try List.First([fields][customfield_15155])[value] otherwise null),
        #"Scrum Team Column" = Table.AddColumn(#"Product Column", "scrum_team", each try [fields][customfield_11850][value] otherwise null),

        //Table.ExpandRecordColumn(#"Expanded reporter", "aggregateprogress", {"progress", "total"}, {"aggregateprogress.progress", "aggregateprogress.total"}),

        // due to max result 50 litation
        #"Left Join Issues Table" =  Table.Join(all_issues_record_table, "key", #"Scrum Team Column", "right_key", JoinKind.LeftOuter),
        
        // If an issue is removed from a sprint, we can see the issue is in sprint report API, but we won't be able to find the ticket in the sprint issues via API.
        // So, we have to fix these issues.
        #"Final Issues Records" = if update_issues = true then Table.TransformRows(#"Left Join Issues Table", (row) as record => if row[right_key] = null 
                                                                                                    then GetIssueDetails(url, row, row[id])  //Record.TransformFields(row, {"epic_key", (v)=>"ddd"}) 
                                                                                                    else row)
                                                          else Table.ToRecords(#"Left Join Issues Table")
                                            

    in
        [ sprintReport = #"Removed Points", sprintIssues = #"Final Issues Records" ]
        //#"Initial Value"
in
    GetScrumSprintReport
```

## GetScrumSprintReports
```
let
    GetScrumSprintReports = (sprintReport_table_temp as table, 
                             sprintIssues_table_temp as table,
                             url as text, 
                             boardId as number, 
                             optional startAt as number, 
                             optional sprint_matches_text as text, 
                             optional update_sprints as any,
                             optional update_issues as any) =>
    let
        replaceIfExists = ReplaceIfExists,
        sprint_matches_text = if sprint_matches_text = null then "" else sprint_matches_text,
        update_sprints = if update_sprints = true then true else false,
        update_issues = if update_issues = true then true else false,
        startAt = if startAt = null then 0 else startAt,
        maxResults = 50,

        // https://yourjira.com/rest/greenhopper/1.0/integration/teamcalendars/sprint/list?jql=project+%3D+YOURPROJECTKEY
        // https://dingyuliang.atlassian.net/rest/greenhopper/1.0/sprintquery/1
        // https://dingyuliang.atlassian.net//rest/agile/1.0/board/1/sprint
        /*
        {"maxResults":50,"startAt":0,"isLast":true,"values":[{"id":1,"self":"https://dingyuliang.atlassian.net/rest/agile/1.0/sprint/1","state":"closed","name":"POW Sprint 1","startDate":"2022-07-28T07:49:36.309Z","endDate":"2022-07-29T07:49:00.000Z","completeDate":"2022-07-28T07:50:49.734Z","originBoardId":1,"goal":""},{"id":2,"self":"https://dingyuliang.atlassian.net/rest/agile/1.0/sprint/2","state":"closed","name":"POW Sprint 2","startDate":"2022-07-28T14:08:32.636Z","endDate":"2022-07-29T07:49:00.000Z","completeDate":"2022-07-28T14:09:42.234Z","originBoardId":1,"goal":""},{"id":3,"self":"https://dingyuliang.atlassian.net/rest/agile/1.0/sprint/3","state":"active","name":"POW Sprint 3","startDate":"2022-07-29T06:03:06.319Z","endDate":"2022-07-29T23:44:00.000Z","originBoardId":1,"goal":""}]}
        */
        sprints_info = if update_sprints = true then 
                            Json.Document(
                                        Web.Contents(url&"/rest/agile/1.0/board/"&Number.ToText(boardId)&"/sprint", [Query = [maxResults = Number.ToText(maxResults), startAt = Number.ToText(startAt)], Headers=[Accept = "application/json", Authorization="Bearer "&JIRAPersonalAccessToken]])
                                        )
                        else 
                            [
                                isLast = true, 
                                // Don't use ToList, which will cause issues
                                values = Table.ToRecords(Table.AddColumn(
                                                                    sprintReport_table_temp,
                                                                    "id",
                                                                    each [sprint_id]
                                                                    )
                                                     )
                            ],
    
        isLast = sprints_info[isLast],
        sprint_list = sprints_info[values],

        closed_sprints = List.Select(sprint_list, each Text.Lower(_[state]) = "closed" and ( if sprint_matches_text = "" then true else Text.Contains(_[name], sprint_matches_text))), 
        // GetScrumSprintReport -> [ sprintReport = #"Removed Points", sprintIssues = all_issues_record_list ]
        sprint_rowvalue_list = List.Transform(closed_sprints, each GetScrumSprintReport(url, boardId, _[id], update_issues)),
        
        #"Current Sprint Reports" = List.Accumulate(sprint_rowvalue_list, 
                                                    [ sprintReport = sprintReport_table_temp, sprintIssues = sprintIssues_table_temp ],
                                                    (state, current) => if update_sprints then 
                                                                                (
                                                                                // update sprints
                                                                                        if Table.Contains(state[sprintReport], 
                                                                                        [
                                                                                            board_id = current[sprintReport][board_id],
                                                                                            sprint_id = current[sprintReport][sprint_id]
                                                                                        ]) then 
                                                                                            [ 
                                                                                                sprintReport = if replaceIfExists = true then Table.ReplaceRows(state[sprintReport], Table.PositionOf(state[sprintReport],current[sprintReport]), 1, {current[sprintReport]}) else state[sprintReport],
                                                                                                sprintIssues = state[sprintIssues]
                                                                                            ] 
                                                                                        else
                                                                                            [ 
                                                                                                sprintReport = Table.InsertRows(state[sprintReport], 0, {current[sprintReport]}),
                                                                                                sprintIssues = if update_issues then
                                                                                                                (
                                                                                                                    List.Accumulate(
                                                                                                                                    current[sprintIssues], 
                                                                                                                                    state[sprintIssues], 
                                                                                                                                    (_state, _current) => 
                                                                                                                                            if Table.Contains(_state, 
                                                                                                                                            [
                                                                                                                                                board_id = _current[board_id],
                                                                                                                                                sprint_id = _current[sprint_id],
                                                                                                                                                issue_id = _current[issue_id]
                                                                                                                                            ]) then 
                                                                                                                                                if replaceIfExists = true then Table.ReplaceRows(_state, Table.PositionOf(_state,_current), 1, {_current}) else _state
                                                                                                                                            else
                                                                                                                                                Table.InsertRows(_state,0, {_current})
                                                                                                                                    )
                                                                                                                 )
                                                                                                                 else state[sprintIssues]
                                                                                            ] 
                                                                                )
                                                                            else (
                                                                                // not update sprints
                                                                                            [ 
                                                                                                sprintReport = state[sprintReport],
                                                                                                sprintIssues = if update_issues then
                                                                                                                (
                                                                                                                    List.Accumulate(
                                                                                                                                    current[sprintIssues], 
                                                                                                                                    state[sprintIssues], 
                                                                                                                                    (_state, _current) => 
                                                                                                                                            if Table.Contains(_state, 
                                                                                                                                            [
                                                                                                                                                board_id = _current[board_id],
                                                                                                                                                sprint_id = _current[sprint_id],
                                                                                                                                                issue_id = _current[issue_id]
                                                                                                                                            ]) then 
                                                                                                                                                if replaceIfExists = true then Table.ReplaceRows(_state, Table.PositionOf(_state,_current), 1, {_current}) else _state
                                                                                                                                            else
                                                                                                                                                Table.InsertRows(_state,0, {_current})
                                                                                                                                    )
                                                                                                                 )
                                                                                                                 else state[sprintIssues]
                                                                                            ]
                                                                            )
                                                                        
                                                    ),

        #"Sprint Reports" = if isLast then #"Current Sprint Reports" else GetScrumSprintReports(#"Current Sprint Reports"[sprintReport], #"Current Sprint Reports"[sprintIssues], url, boardId, startAt + maxResults, sprint_matches_text, update_sprints, update_issues)
              
    in
        #"Sprint Reports"
in
    GetScrumSprintReports
```

## GetAllScrumSprintReports
```
let
    GetAllScrumSprintReports = (url as text, boardIds as list, optional sprint_matches_text as text, optional update_sprints as any, optional update_issues as any) =>
    let    
        update_sprints = if update_sprints = true then true else false,
        update_issues = if update_issues = true then true else false,
        /*
            # JIRA PowerBI Connector OData metadata
            <edmx:Edmx xmlns:edmx="http://docs.oasis-open.org/odata/ns/edmx" Version="4.0">
            <edmx:DataServices>
            <Schema xmlns="http://docs.oasis-open.org/odata/ns/edm" Namespace="Alpha-Serve-Power-BI">
            <EntityType Name="Issues">
            <Key>
            <PropertyRef Name="ISSUE_ID"/>
            </Key>
            <Property Name="ISSUE_ID" Type="Edm.Int64"/>
            <Property Name="ISSUE_KEY" Type="Edm.String"/>
            </EntityType>
            <EntityType Name="SprintReports">
            <Property Name="BOARD_ID" Type="Edm.Int64"/>
            <Property Name="BOARD_NAME" Type="Edm.String"/>
            <Property Name="SPRINT_ID" Type="Edm.Int64"/>
            <Property Name="SPRINT_NAME" Type="Edm.String"/>
            <Property Name="COMPLETED_INITIAL_ESTIMATE" Type="Edm.Double"/>
            <Property Name="COMPLETED_ESTIMATE" Type="Edm.Double"/>
            <Property Name="COMPLETED_ISSUES_COUNT" Type="Edm.Int64"/>
            <Property Name="INCOMPLETED_INITIAL_ESTIMATE" Type="Edm.Double"/>
            <Property Name="INCOMPLETED_ESTIMATE" Type="Edm.Double"/>
            <Property Name="INCOMPLETED_ISSUES_COUNT" Type="Edm.Int64"/>
            <Property Name="COMPLETED_OUTSIDE_SPRINT_INITIAL_ESTIMATE" Type="Edm.Double"/>
            <Property Name="COMPLETED_OUTSIDE_SPRINT_ESTIMATE" Type="Edm.Double"/>
            <Property Name="COMPLETED_OUTSIDE_SPRINT_ISSUES_COUNT" Type="Edm.Int64"/>
            <Property Name="REMOVED_FROM_SPRINT_INITIAL_ESTIMATE" Type="Edm.Double"/>
            <Property Name="REMOVED_FROM_SPRINT_ESTIMATE" Type="Edm.Double"/>
            <Property Name="REMOVED_FROM_SPRINT_ISSUES_COUNT" Type="Edm.Int64"/>
            <Property Name="ADDED_DURING_SPRINT_ISSUES_COUNT" Type="Edm.Int64"/>
            </EntityType>
            <EntityType Name="SprintReportIssues">
            <Property Name="BOARD_ID" Type="Edm.Int64"/>
            <Property Name="BOARD_NAME" Type="Edm.String"/>
            <Property Name="SPRINT_ID" Type="Edm.Int64"/>
            <Property Name="SPRINT_NAME" Type="Edm.String"/>
            <Property Name="ISSUE_ID" Type="Edm.Int64"/>
            <Property Name="ISSUE_KEY" Type="Edm.String"/>
            <Property Name="INITIAL_ESTIMATE" Type="Edm.Double"/>
            <Property Name="CURRENT_ESTIMATE" Type="Edm.Double"/>
            <Property Name="SPRINT_REPORT_STATUS" Type="Edm.String"/>
            <Property Name="IS_ADDED_DURING_SPRINT" Type="Edm.Boolean"/>
            </EntityType>
            <EntityContainer Name="Container">
            <EntitySet EntityType="Alpha-Serve-Power-BI.Issues" Name="Issues"/>
            <EntitySet EntityType="Alpha-Serve-Power-BI.SprintReports" Name="SprintReports"/>
            <EntitySet EntityType="Alpha-Serve-Power-BI.SprintReportIssues" Name="SprintReportIssues"/>
            </EntityContainer>
            </Schema>
            </edmx:DataServices>
            </edmx:Edmx>
            */
        // https://gorilla.bi/power-query/creating-tables/#table-fromcolumns
    
        sprintReport_table_temp = if update_sprints = true then
                                            Table.AddKey(
                                                #table(type table[ board_id = Int64.Type, 
                                                            sprint_id = Int64.Type, 
                                                            sequence = Int64.Type,
                                                            name = Text.Type,
                                                            state = Text.Type,
                                                            iso_start_date = DateTime.Type,
                                                            iso_end_date = DateTime.Type,
                                                            completed_initial_estimate = Double.Type,
                                                            completed_estimate = Double.Type,
                                                            completed_issues_count = Int32.Type,
                                                            incompleted_initial_estimate = Double.Type,
                                                            incompleted_estimate = Double.Type,
                                                            incompleted_issues_count = Int32.Type,
                                                            completed_outside_sprint_initial_estimate = Double.Type,
                                                            completed_outside_sprint_estimate = Double.Type,
                                                            completed_outside_sprint_issues_count = Int32.Type,
                                                            removed_from_sprint_initial_estimate = Double.Type,
                                                            removed_from_sprint_estimate = Double.Type,
                                                            removed_from_sprint_issues_count = Int32.Type,
                                                            added_during_sprint_initial_estimate = Double.Type,
                                                            added_during_sprint_estimate = Double.Type,
                                                            added_during_sprint_issues_count = Int32.Type,
                                                            changed_during_sprint_initial_estimate = Double.Type,
                                                            changed_during_sprint_estimate = Double.Type,
                                                            changed_during_sprint_issues_count = Int32.Type,
                                                            committed_points = Double.Type,
                                                            added_points = Double.Type,
                                                            changed_points = Double.Type,
                                                            delivered_points = Double.Type,
                                                            not_delivered_points = Double.Type,
                                                            removed_points = Double.Type
                                                    ],
                                                {}), 
                                                {"board_id", "sprint_id"}, 
                                                true)
                                        else SprintReports,         

        sprintIssues_table_temp = Table.AddKey(
                                            #table(type table[ 
                                                    issue_id = Int64.Type, 
                                                    board_id = Int64.Type, 
                                                    sprint_id = Int64.Type,  
                                                    summary = Text.Type,
                                                    key = Text.Type,
                                                    typeId = Text.Type,
                                                    epic_key = Text.Type,
                                                    value_stream = Text.Type,
                                                    sub_value_stream = Text.Type,
                                                    product = Text.Type,
                                                    scrum_team = Text.Type,
                                                    initial_estimate = Double.Type,
                                                    current_estimate = Double.Type
                                            ],
                                            {}), 
                                            {"issue_id", "board_id", "sprint_id"}, 
                                            true), 
    
        // GetAllScrumSprintReports -> [ sprintReport = current_sprintReport_table, sprintIssues = current_sprintIssues_table ]
        #"Result" = List.Accumulate(boardIds, 
                                [ sprintReport = sprintReport_table_temp, sprintIssues = sprintIssues_table_temp ], 
                                (state, boardId) => GetScrumSprintReports(state[sprintReport], state[sprintIssues], url, boardId, 0, sprint_matches_text, update_sprints, update_issues))
    in
        #"Result" 
in
    GetAllScrumSprintReports
```

## GetIssueDetails
```
let
    GetIssueDetails = (url as text, issueRecord as record, issueId as number) =>
    let
        // https://developer.atlassian.com/cloud/jira/software/rest/api-group-issue/#api-rest-agile-1-0-issue-issueidorkey-get
        // https://a.jira.com/rest/agile/1.0/issue/{issueIdOrKey}?fields=id,key,customfield_16052,customfield_15351,customfield_15155
        issues_details_content = Web.Contents(url&"/rest/agile/1.0/issue/"&Text.From(issueId), [Query 
                                                                                        = [ 
                                                                                            fields = {
                                                                                                "id", 
                                                                                                "key",
                                                                                                "assignee",           // assignee.name, emailAddress, key, displayName
                                                                                                "priority",           // priority.name
                                                                                                "customfield_11350",  // epic id
                                                                                                "customfield_15351",  // value stream  ->.value
                                                                                                "customfield_15155",  // granicus product is an array ->.value
                                                                                                "customfield_11850"   // scrum team -> .value
                                                                                                }], 
                                                                                      Headers=[#"Accept" = "application/json", #"Authorization"="Bearer "&JIRAPersonalAccessToken]]),
         
        issues_details_json = Json.Document(issues_details_content),

        #"Detailed Issue" = issues_details_json,

        #"Source Issue" = issueRecord,

        #"Epic ID Column" = Record.TransformFields(#"Source Issue", {"epic_key", (v) => try #"Detailed Issue"[fields][customfield_11350] otherwise null}),
        #"Value Stream Column" = Record.TransformFields(#"Epic ID Column", {"value_stream",  (v) => try #"Detailed Issue"[fields][customfield_15351][value] otherwise null}),
        #"Sub Value Stream Column" = Record.TransformFields(#"Value Stream Column", {"sub_value_stream",  (v) => try #"Detailed Issue"[fields][customfield_15351][child][value] otherwise null}),
        #"Product Column" = Record.TransformFields(#"Sub Value Stream Column", { "product",  (v) => try List.First(#"Detailed Issue"[fields][customfield_15155])[value] otherwise null}),
        #"Scrum Team Column" = Record.TransformFields(#"Product Column", {"scrum_team", (v) => try #"Detailed Issue"[fields][customfield_11850][value] otherwise null})

    in
        #"Scrum Team Column"
        //#"Initial Value"
in
    GetIssueDetails
```

## SprintReports
```
let 
    // configure scrum board id list.
    board_ids = List.Transform(Text.Split(BoardIds, ","), each Number.FromText(_)),
    // GetAllScrumSprintReports -> [ sprintReport = current_sprintReport_table, sprintIssues = current_sprintIssues_table ]
    #"Sprint Reports" = GetAllScrumSprintReports(Text.TrimEnd(URL, "/"), board_ids, SprintNameRegex,  true, false)
in
    #"Sprint Reports"[sprintReport]
```

## SprintReportIssues
```
let
    SprintReportIssues = let 
    // Add dependency
    Source = SprintReports,
    //#"Removed Columns" = Table.RemoveColumns(Source,{"sequence", "name", "state", "iso_start_date", "iso_end_date", "completed_initial_estimate", "completed_estimate", "completed_issues_count", "incompleted_initial_estimate", "incompleted_estimate", "incompleted_issues_count", "completed_outside_sprint_initial_estimate", "completed_outside_sprint_estimate", "completed_outside_sprint_issues_count", "removed_from_sprint_initial_estimate", "removed_from_sprint_estimate", "removed_from_sprint_issues_count", "added_during_sprint_initial_estimate", "added_during_sprint_estimate", "added_during_sprint_issues_count", "changed_during_sprint_initial_estimate", "changed_during_sprint_estimate", "changed_during_sprint_issues_count", "committed_points", "added_points", "changed_points", "delivered_points", "not_delivered_points", "removed_points"}),
    //#"Removed Duplicates" = Table.Distinct(#"Removed Columns"),

    /*
    #"Sprint Issues" = Table.AddKey(
                                            #table(type table[ 
                                                    issue_id = Int64.Type, 
                                                    board_id = Int64.Type, 
                                                    sprint_id = Int64.Type,  
                                                    summary = Text.Type,
                                                    key = Text.Type,
                                                    typeId = Text.Type,
                                                    initial_estimate = Double.Type,
                                                    current_estimate = Double.Type
                                            ],
                                            {}), 
                                            {"issue_id", "board_id", "sprint_id"}, 
                                            true)
    */
    
    // configure scrum board id list.
    board_ids = List.Transform(Text.Split(BoardIds, ","), each Number.FromText(_)),
    // GetAllScrumSprintReports -> [ sprintReport = current_sprintReport_table, sprintIssues = current_sprintIssues_table ]
    #"Sprint Reports" = GetAllScrumSprintReports(Text.TrimEnd(URL, "/"), board_ids, SprintNameRegex, false, true)
in  
    #"Sprint Reports"[sprintIssues]
in
    SprintReportIssues
```