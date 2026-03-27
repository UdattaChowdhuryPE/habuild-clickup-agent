---
name: audit-pod
description: Audit a single POD's sprint readiness and update the Doc table. Pass the POD name as argument.
argument-hint: "[pod-name]"
---

Run a full sprint readiness compliance audit for POD: **$ARGUMENTS**

Follow the rules in CLAUDE.md exactly.

## Pre-Known IDs (do NOT discover dynamically)

| POD            | Backlog List ID  | Folder ID    | Epic List ID     | Sprint Readiness Task ID |
|----------------|------------------|--------------|------------------|--------------------------|
| QA             | 901613221638     | 90168264586  | 901613221607     | 86d2899n5                |
| Platform       | 901613091713     | 90168160491  | 901613091701     | 86d289c19                |
| Experience     | 901613363633     | 90168380918  | 901613363614     | 86d289j7m                |
| Internal Tools | 901613486023     | 90168482660  | 901613486013     | 86d289j9h                |
| Beetu          | 901613520227     | 90168513028  | 901613520195     | 86d289ja7                |
| Infra          | 901613554762     | 90168540627  | 901613554735     | 86d289jav                |
| Growth         | 901613519642     | 90168512663  | 901613520075     | 86d289jbz                |
| Mobile         | 901613593087     | 90168572617  | 901613593121     | 86d289jca                |

## Steps

### Step 1: Look up IDs
Look up `$ARGUMENTS` in the table above to get `backlog_list_id`, `folder_id`, `epic_list_id`, and `sprint_readiness_task_id`.

### Step 2: Sprint Discovery
Use the Agent tool to invoke pod-auditor as a sub-agent:
- Pass: `folder_id`, `today` (read from the `# currentDate` system context — format YYYY-MM-DD.
  If `# currentDate` is not present in context, stop and ask the user for today's date before proceeding.)
- Receive: SPRINT_DISCOVERY_RESULT with current_sprint_list_id, current_sprint_end_date, next_sprint_list_id

The pod-auditor agent uses `clickup_get_workspace_hierarchy` to enumerate lists in the folder and parses sprint date formats:
- QA, Platform use `D/M` format: e.g., `(16/3 - 30/3)` for dates within current year
- Experience, Mobile use `M/D` format: e.g., `(3/16 - 3/30)` for dates within current year
- Internal Tools, Beetu, Infra, Growth use `D/M/YY` format: e.g., `(16/3/26 - 30/3/26)` with two-digit year

After receiving SPRINT_DISCOVERY_RESULT:
- If `fallback_used: true`, print:
  WARNING: pod-auditor used fallback sprint for $ARGUMENTS. Reason: [fallback_note].
  Sprint date format may be unrecognized or no sprint overlaps today. Verify manually.
  Carry this warning into the observations passed to doc-updater.

If pod-auditor returns an error (no current sprint found):
```
ERROR: Could not identify active sprint for $ARGUMENTS. Skipping.
[Continue to next POD or end if this is the last one]
```

### Step 3: Run CHECK 1
Use the Agent tool to invoke check1-agent as a sub-agent:
- Pass: POD name, `epic_list_id`, `current_sprint_list_id`
- Receive: CHECK 1 result (compliance %, violations list)

### Step 4: Run CHECK 2
Use the Agent tool to invoke check2-agent as a sub-agent:
- Pass: POD name, `backlog_list_id`
- Receive: CHECK 2 result (compliance %, violations list)

### Step 5: Run CHECK 3
Use the Agent tool to invoke check3-agent as a sub-agent:
- Pass: POD name, `backlog_list_id`, `current_sprint_list_id`
- Receive: CHECK 3 result (compliance %, violations list)

### Step 5.5: Validate CHECK results
Before proceeding, verify that all three check agents returned valid results:
- Each result must contain a line like `CHECK N — Name: [%] → [Status]` with an actual compliance percentage
- If any check agent returned empty, null, or no compliance %, print:
```
ERROR: check[N]-agent returned no data for $ARGUMENTS. Stopping audit.
```
Stop processing. Do NOT proceed to subsequent PODs or steps. Do NOT fill in placeholder/fabricated compliance values.

### Step 6: Sprint N+1 Check (timing-gated)
Compute the window: `[current_sprint_end_date - 5 days, current_sprint_end_date]`
- If today is within this window AND `next_sprint_list_id` is not null:
  - Use the Agent tool to invoke check4-agent as a sub-agent
  - Pass: POD name, `folder_id`, `next_sprint_list_id`, `today`, `current_sprint_end_date`
  - Receive: CHECK 4 result (compliance %, violations list)
- Otherwise: Sprint N+1 result = "N/A — outside readiness window"

### Step 7: Compute compliance and statuses
For each check, compute compliance % and determine status:
- Done (≥97%) / In Progress (30–96%) / Not Started (<30%)

### Step 8: Update Dashboard
Use the Agent tool to invoke doc-updater as a sub-agent. Pass ALL of the following — do NOT omit any field:
- POD: [pod name from $ARGUMENTS]
- Check 1 — Epics Setup: [compliance %] → [status label: Done / In Progress / Not Started]
- Check 2 — Backlog Hygiene: [compliance %] → [status label]
- Check 3 — Key Fields Updated: [compliance %] → [status label]
- Check 4 — Sprint N+1: [compliance % or "N/A"] → [status label or "N/A"]
- Observations (EPIC section): [verbatim violations from check1-agent, or "None"]
- Observations (Backlog section): [verbatim violations from check2-agent, or "None"]
- Observations (Current Sprint section): [verbatim violations from check3-agent, or "None"]
- Observations (Sprint N+1 section): [verbatim violations from check4-agent, or "None", or "N/A — outside window"]

doc-updater will look up the sprint_readiness_task_id, map each % to an option ID, and post the mandatory audit summary comment.

### Step 9: Print result
`POD $ARGUMENTS: Epics=[status], Backlog=[status], KeyFields=[status], SprintN+1=[status or N/A]`

## Constraints
- Do NOT modify any POD tasks
- `clickup_get_workspace_hierarchy` is ALLOWED for sprint discovery only (to enumerate lists in the folder)
- Current sprint tasks only (ignore future sprints, archived, previous sprint completed tasks)
- Backlog checks use backlog list only; sprint checks use current sprint list only
- All sub-agent invocations MUST use the Agent tool — never inline a check

## Task Fetching Rule (Token-efficient)
Always pass `statuses` to `filter_tasks` — never fetch everything then filter.
- CHECK 1: `filter_tasks(list_ids=[epic_list_id], ...)`
- CHECK 2: `filter_tasks` to get task IDs (custom_fields not included in response), then `clickup_get_task` per task to inspect Epic custom_field
  - Exception to no-loop rule: filter_tasks response strips custom_fields; per-task lookups required to detect Epic connections
- CHECK 3: `filter_tasks(list_ids=[backlog_list_id], statuses=[8 active statuses])`, then keep only tasks where locations includes active_sprint_list_id
- CHECK 4: `filter_tasks(list_ids=[next_sprint_list_id], statuses=["Task Definition Complete", "Ready For Dev"])`
  - Note: Sprint confirmation is now determined at the list level (via `clickup_get_folder`), not by task status
General principle: Never call `clickup_get_task` in a loop UNLESS the batch tool doesn't return needed data.
- CHECK 2: per-task get_task required — filter_tasks omits custom_fields
- CHECK 3: per-task get_task required — filter_tasks omits time_estimate, points, and custom_fields
- CHECK 4: per-task get_task required — filter_tasks omits points and custom_fields
- CHECK 1: per-task get_task NOT required — filter_tasks returns assignees
