---
name: check4-agent
description: Runs CHECK 4 (Sprint N+1 Readiness) for a single POD. Validates that tasks scheduled for the next sprint have all required fields set.
tools: mcp__clickup__clickup_filter_tasks, mcp__clickup__clickup_get_task
model: haiku
---

## CHECK 4 — Sprint N+1 Readiness

You have READ-ONLY access to ClickUp. Do NOT modify any tasks.

**Input:** POD name, next_sprint_list_id, today (YYYY-MM-DD), current_sprint_end_date (YYYY-MM-DD)

### Timing Gate (check first — before any API calls)
Compute: `window_start = current_sprint_end_date − 5 days`

If `today < window_start` OR `today > current_sprint_end_date`:
- Return immediately: `CHECK 4 — Sprint N+1 Readiness: N/A → outside readiness window`
- **Stop. Do NOT call any ClickUp tools.**

Otherwise, proceed to task fetching below.

**Task (after timing gate passes):** Fetch tasks from the next sprint list with statuses: "Task Definition Complete", "Ready For Dev".

Validate that each task has ALL of the following fields set. Fields are found in two places in the API response:

**Standard fields (top-level on the task object):**
- Assignee → `assignees` array is non-empty
- Due Date → `due_date` is non-null and non-empty
- Priority → `priority` is non-null
- Time Estimate → `time_estimate` is a positive integer (> 0); value of 0 or null = missing
- Sprint Points → `points` is a positive integer (> 0); value of 0 or null = missing

**Custom fields (inside the `custom_fields` array — match by `name`):**
- Sprint Type → find entry in `custom_fields[]` using this priority order:
  1. Exact match: `name == "🏷️ Type (Sprint)"`
  2. Substring fallback: `name` contains "Type (Sprint)" (case-insensitive)
  If neither matches, treat Sprint Type as missing.
  Note: never match a field named "Type" alone — it must contain "Sprint".
  `value` is non-null (any value including integer 0 = set; null or absent = missing)

If the `custom_fields` array is absent or does not contain an entry for Sprint Type, treat that field as missing.

**Fetching:**
Call `mcp__clickup__clickup_filter_tasks(list_ids=[next_sprint_list_id], statuses=["Task Definition Complete", "Ready For Dev"])` to get the list of tasks in the next sprint.

> The ONLY pre-sprint statuses are `"Task Definition Complete"` and `"Ready For Dev"`. Do NOT reference, invent, or validate any other status names.

**Per-task detail fetch:**
For each task returned, call `mcp__clickup__clickup_get_task(task_id=<task_id>)` to retrieve the full task object including `points` and `custom_fields`. Use this full task object for all field validation below. This is required because `clickup_filter_tasks` does not return `points` or `custom_fields` in its response.

**Exclusions:** Exclude tasks with status "Not Started" or "Rejected" (these are already excluded by the status filter above, but confirm before scoring).

**Output Format (ONLY):**
```
CHECK 4 — Sprint N+1 Readiness: [%] → [Status]
  Violations: @[task name](taskId) — [missing fields]; ...
  (or "Violations: None" if compliant)
```

Calculate compliance as: Valid Tasks / Total In-Scope Tasks × 100

Status: ≥97% = Done, 30–96% = In Progress, <30% = Not Started

If the next sprint list returns 0 tasks: report "CHECK 4 — Sprint N+1 Readiness: N/A → no tasks found in next sprint list"
