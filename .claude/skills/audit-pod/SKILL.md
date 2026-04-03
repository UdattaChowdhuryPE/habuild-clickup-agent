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
Use the Agent tool to invoke epics-setup-agent as a sub-agent:
- Pass: POD name, `epic_list_id`, `current_sprint_list_id`
- Receive: CHECK 1 result (compliance %, violations list)

### Step 4: Run Backlog Hygiene + Sprint Fields Checks
Use the Agent tool to invoke backlog-hygiene-and-sprint-fields-agent as a sub-agent:
- Pass: POD name, `backlog_list_id`, `current_sprint_list_id`
- Receive: both CHECK 2 result and CHECK 3 result (compliance %, violations list for each)

### Step 5: Run CS Checks (Ticket Description & Acceptance Criteria)
Use the Agent tool to invoke cs-checks-agent as a sub-agent:
- Pass: POD name, `current_sprint_list_id`
- Receive: CS_CHECKS_RESULT with description_total, description_missing_count, description_violations, ac_eligible_count, ac_missing_count, ac_violations

Parse the CS_CHECKS_RESULT output block and extract all six values. Store them for passing to doc-updater in Step 9.

### Step 6: Validate CHECK results
Before proceeding, verify that all three check agents (CHECK 1, 2, 3) returned valid results:
- Each result must contain a line like `CHECK N — Name: [%] → [Status]` with an actual compliance percentage
- If any check agent returned empty, null, or no compliance %, print:
```
ERROR: check[N]-agent returned no data for $ARGUMENTS. Stopping audit.
```
Stop processing. Do NOT proceed to subsequent PODs or steps. Do NOT fill in placeholder/fabricated compliance values.

Also verify that cs-checks-agent returned a valid CS_CHECKS_RESULT block with all six required fields.

### Step 7: Run CHECK 4 — Sprint N+1 Readiness
Use the Agent tool to invoke next-sprint-readiness-agent as a sub-agent:
- Pass: POD name, `folder_id`, `next_sprint_list_id` (from Step 2), `today`, `current_sprint_end_date` (from Step 2)
- Receive: CHECK 4 result (status: Done / In Progress / Not Started / N/A → outside readiness window, and violations list)

### Step 8: Compute compliance and statuses
For each check, compute compliance % and determine status:
- Done (≥97%) / In Progress (30–96%) / Not Started (<30%)

### Step 10: Update Dashboard
Use the Agent tool to invoke doc-updater as a sub-agent. Pass ALL of the following — do NOT omit any field:
- POD: [pod name from $ARGUMENTS]
- Check 1 — Epics Setup: [compliance %] → [status label: Done / In Progress / Not Started]
- Check 2 — Backlog Hygiene: [compliance %] → [status label]
- Check 3 — Key Fields Updated: [compliance %] → [status label]
- Check 4 — Sprint N+1: [compliance % or "N/A"] → [status label or "N/A"]
- Check 5 — Ticket Description: [missing_count], [violations list] (from cs-checks-agent)
- Check 6 — Acceptance Criteria: [missing_count], [violations list] (from cs-checks-agent)
- Observations (EPIC section): [verbatim violations from epics-setup-agent, or "None"]
- Observations (Backlog section): [verbatim violations from backlog-hygiene-and-sprint-fields-agent CHECK 2 block, or "None"]
- Observations (Current Sprint section): [verbatim violations from backlog-hygiene-and-sprint-fields-agent CHECK 3 block, or "None"]
- Observations (Sprint N+1 section): [verbatim violations from next-sprint-readiness-agent, or "None", or "N/A — outside window"]
- Observations (🎫 Ticket Description section): [violations from cs-checks-agent, or "None"]
- Observations (👍 Acceptance Criteria section): [violations from cs-checks-agent, or "None"]

doc-updater will look up the sprint_readiness_task_id, map each % to an option ID, and post the mandatory audit summary comment.

### Step 11: Print result
`POD $ARGUMENTS: Epics=[status], Backlog=[status], KeyFields=[status], SprintN+1=[status or N/A], Desc=[count], AC=[count]`

## Constraints
- Do NOT modify any POD tasks
- `clickup_get_workspace_hierarchy` is ALLOWED for sprint discovery only (to enumerate lists in the folder)
- Current sprint tasks only (ignore future sprints, archived, previous sprint completed tasks)
- Backlog checks use backlog list only; sprint checks use current sprint list only
- All sub-agent invocations MUST use the Agent tool — never inline a check

## Task Fetching Rule (Token-efficient)
Always pass `statuses` to `filter_tasks` — never fetch everything then filter.
- CHECK 1: `filter_tasks(list_ids=[epic_list_id], ...)` — no per-task get_task needed (assignees in response)
- CHECK 2+3: backlog-hygiene-and-sprint-fields-agent shares one backlog filter + one sprint filter; calls `clickup_get_task` per unique task sequentially (one at a time) — used for both Epic detection (check2) and field validation (check3)
- CHECK 4: `filter_tasks(list_ids=[next_sprint_list_id], statuses=["Task Definition Complete", "Ready For Dev"])`, then `clickup_get_task` per task sequentially — filter_tasks omits points and custom_fields
  - Sprint confirmation is determined at the list level (via `clickup_get_folder`), not by task status
