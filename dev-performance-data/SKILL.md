---
name: dev-performance-data
description: |
  Use when the user asks to "fetch dev performance data", "update sprint data",
  "refresh performance analytics", "pull jira sprint data", or invokes /dev-performance-data.
  Fetches {{PROJECT_KEY}} Jira tickets and changelogs via MCPs and saves to the dev-team-performance-analytics data folder.
---

## Language
- Write in simple, plain language. Short sentences, common words.
- No filler, no preamble, no marketing phrasing. Every line must add information.
- One idea per line; prefer bullets over paragraphs.

# Dev Performance Data Fetcher

Fetch Jira ticket data and changelogs for the {{PROJECT_KEY}} project and save them as raw JSON files
for the dev-team-performance-analytics dashboard.

## CRITICAL: Data Preservation Rule

**NEVER overwrite or replace existing data files.** Always MERGE new data with existing data.

When updating data:
1. Read the existing `sprints.json`, `tickets.json`, and `changelogs.json` first
2. Merge new entries into the existing arrays (deduplicate by sprint id, ticket key, or issue_key)
3. Only then write the combined result back

The ONLY exception is if the user explicitly instructs you to replace/overwrite the files.

## Configuration

- **Jira Project**: {{PROJECT_KEY}}
- **Board ID**: 5 ({{PROJECT_KEY}} board)
- **Sprint field**: customfield_10020
- **Output directory**: `C:\Users\ChrisWeigelt\Documents\Claude Code\dev-team-performance-analytics\data\raw\`
- **Processing script**: `C:\Users\ChrisWeigelt\Documents\Claude Code\dev-team-performance-analytics\scripts\process_data.py`
- **Python path**: `C:\Users\ChrisWeigelt\AppData\Local\Python\bin\python.exe`

## Execution Steps

### Step 1: Determine Sprint Scope

Ask the user which sprints to fetch. Offer these options:
- **Default**: Active sprint + last 5 closed sprints
- **Custom range**: User specifies sprint numbers or count (e.g., "last 10 sprints")
- **All**: Fetch all available sprints (warn: may be slow for 70+ sprints)

Use `jira_get_sprints_from_board` with board_id "5" to list sprints:
- state="active" for current sprint
- state="closed" for past sprints (paginate with start_at/limit to get the desired count, fetching from most recent)

Save the sprint list to `data/raw/sprints.json` (merged with existing).

### Step 2: Fetch Tickets

For **closed sprints**, fetch DONE tickets using `jira_search` with JQL:

```
project = {{PROJECT_KEY}} AND sprint = {sprintId} AND status = "DONE" ORDER BY created ASC
```

For **active sprints**, fetch ALL tickets (DONE + in-progress) using JQL:

```
project = {{PROJECT_KEY}} AND sprint = {sprintId} AND status != "To Do" ORDER BY created ASC
```

This ensures in-progress tickets appear in the dashboard when the "Include in-progress" toggle is used.

Use pagination (limit=50, follow next_page_token) to get all tickets per sprint.

For each ticket, capture these fields: `summary,status,assignee,issuetype,parent,created,updated,customfield_10020,subtasks`

Also check each parent ticket for subtasks. If a ticket has subtasks (visible in the issue data),
fetch those subtask details individually using `jira_get_issue`.

Collect all tickets (parents + subtasks) into a single array. Each ticket object should contain:
```json
{
  "key": "{{PROJECT_KEY}}-1807",
  "id": "21990",
  "summary": "...",
  "status": "Done",
  "assignee": "Piotr Rudzki",
  "issue_type": "Task",
  "is_subtask": false,
  "parent_key": null,
  "created": "2026-02-23T...",
  "sprint_name": "{{PROJECT_KEY}} Sprint 70",
  "sprint_id": 987
}
```

For tickets in multiple sprints, use the **last** sprint in `customfield_10020` array.

Save to `data/raw/tickets.json` (merged with existing).

### Step 3: Fetch Changelogs

**DO NOT use `jira_batch_get_changelogs`.** Batch fetching has caused widespread data corruption
(empty `from_string`/`to_string` fields) and lost inline responses. Instead, fetch changelogs
**individually per ticket** using `jira_get_issue` with `expand="changelog"`.

**CRITICAL: Always re-fetch changelogs for ALL tickets in the current fetch scope**, even if they
already have entries in changelogs.json. Tickets may have changed status since the last fetch
(e.g., moved from "READY TO DEPLOY" to "Done"), and stale changelogs will cause them to appear
incomplete in the dashboard. Do NOT skip tickets just because they already exist in changelogs.json.

For each ticket key collected in Step 2:
1. Call `jira_get_issue` with `issue_key=<KEY>`, `fields="summary"`, `expand="changelog"`
2. Extract the changelog histories from the response
3. Filter to only `status` and `assignee` field changes
4. Build the changelog entry for that ticket

You can call multiple `jira_get_issue` requests in parallel (up to 5-6 at a time) to speed things up.

Save all changelogs to `data/raw/changelogs.json` (merged with existing). When merging, **replace**
any existing changelog entry for a ticket key with the freshly fetched one. The structure should be:
```json
[
  {
    "issue_key": "{{PROJECT_KEY}}-1807",
    "changelogs": [
      {
        "items": [{"field": "status", "from_string": "To Do", "to_string": "In Progress"}],
        "author": {"display_name": "..."},
        "created": "2026-02-23T..."
      }
    ]
  }
]
```

**Verification step:** After all changelogs are fetched, compare the set of ticket keys in
`tickets.json` against `changelogs.json`. Any missing tickets must be re-fetched.
Also scan for corrupted entries (status changes with empty `from_string`/`to_string`) and re-fetch those.

### Step 4: Run Processing Script

After all raw data is saved, run the Python processing script:

```bash
cd "C:\Users\ChrisWeigelt\Documents\Claude Code\dev-team-performance-analytics"
"C:\Users\ChrisWeigelt\AppData\Local\Python\bin\python.exe" scripts/process_data.py
```

This transforms raw data into dashboard-ready JSON files in `data/processed/`.

Report to the user:
- Number of sprints fetched
- Number of tickets collected
- Number of changelog entries
- Confirmation that processing completed successfully
- Remind them to open dashboard.html to view results

## Error Handling

- If a Jira MCP call fails, report the error and continue with remaining items
- If the Python script fails, show the error output and suggest manual fixes
- If no DONE tickets found for a sprint, skip it and note in the summary

## Important Notes

- **Never use `jira_batch_get_changelogs`** — it produces corrupted data with empty status strings.
  Always use individual `jira_get_issue` with `expand="changelog"` instead.
- Tickets may appear in multiple sprints. Always use the last sprint in `customfield_10020`.
- Some tickets may have been created directly in "In Progress" status — this is normal.
  The processing script handles this edge case.
- For active sprints, non-DONE tickets will have `is_current: true` in their last segment.
  The dashboard's "Include in-progress" toggle shows/hides these.
