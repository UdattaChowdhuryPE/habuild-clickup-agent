---
name: check4-agent
description: Runs CHECK 4 (Sprint N+1 Readiness) for a single POD. Validates that tasks scheduled for the next sprint have all required fields set.
tools: mcp__clickup__clickup_filter_tasks, mcp__clickup__clickup_get_task, mcp__clickup__clickup_get_folder
model: haiku
---

## CHECK 4 — Sprint N+1 Readiness

You have READ-ONLY access to ClickUp. Do NOT modify any tasks.

**Input:** POD name, folder_id, next_sprint_list_id, today (YYYY-MM-DD), current_sprint_end_date (YYYY-MM-DD)

### Timing Gate (check first — before any API calls)
Compute: `window_start = current_sprint_end_date − 5 days`

If `today < window_start` OR `today > current_sprint_end_date`:
- Return immediately: `CHECK 4 — Sprint N+1 Readiness: N/A → outside readiness window`
- **Stop. Do NOT call any ClickUp tools.**

Otherwise, proceed to task fetching below.

**Sprint Confirmation Check (after timing gate passes):**
Before fetching tasks, check sprint-level confirmation:
1. Call `mcp__clickup__clickup_get_folder(folder_id=folder_id)` to get the folder with its lists
2. Find the list in the response whose `id == next_sprint_list_id`
3. Inspect the list object for a confirmation property (e.g., `status`, `confirmed`, or similar boolean/flag field that ClickUp sets when the sprint is confirmed)
4. If the sprint is confirmed: **output Done and stop** (no task fetch, field validation skipped)
5. Otherwise: proceed to task fetch and field validation below

**Task (after sprint confirmation check passes):** Fetch tasks from the next sprint list with statuses: "Task Definition Complete", "Ready For Dev".

**Field Validation (only if ≥1 task):**
Validate that each task has ALL of the following fields set. Fields are found in two places in the API response:

**Standard fields (top-level on the task object):**
- Assignee → `assignees` array is non-empty
- Due Date → `due_date` is non-null and non-empty
- Priority → `priority` is non-null
- Time Estimate → `time_estimate` is a positive integer (> 0); value of 0 or null = missing
- Sprint Points → `points` is a positive integer (> 0); value of 0 or null = missing

**Custom fields (inside the `custom_fields` array — match by `name`):**
- **🏷️ Type (Sprint)** → find entry in `custom_fields[]` where `name == "🏷️ Type (Sprint)"` (exact match). If not found, try substring fallback: `name` contains `"Type (Sprint)"` (case-insensitive).
  - Do NOT look for a field named "Sprint Type", "Sprint (Type)", or "Type" alone — the field must match "🏷️ Type (Sprint)" or contain "Type (Sprint)".
  - `value` is non-null (any value including integer 0 = set; null or absent = missing)
  - When reporting this field as missing in violations, always refer to it as **🏷️ Type (Sprint)**.

If the `custom_fields` array is absent or does not contain a matching entry, treat **🏷️ Type (Sprint)** as missing.

**Fetching:**
Call `mcp__clickup__clickup_filter_tasks(list_ids=[next_sprint_list_id], statuses=["Task Definition Complete", "Ready For Dev"])` to get the list of tasks in the next sprint.

> The pre-sprint task statuses are `"Task Definition Complete"` and `"Ready For Dev"`. Do NOT reference or invent any other status names.
> Sprint confirmation is now determined at the list level (via clickup_get_folder), not by task status.

**Per-task detail fetch:**
For each task returned, call `mcp__clickup__clickup_get_task(task_id=<task_id>)` to retrieve the full task object including `points` and `custom_fields`. Use this full task object for all field validation below. This is required because `clickup_filter_tasks` does not return `points` or `custom_fields` in its response.

**Exclusions:** Exclude tasks with status "Not Started" or "Rejected" (these are already excluded by the status filter above, but confirm before scoring).

**Output Format (ONLY):**

If sprint is confirmed (detected at list level before task fetch):
```
CHECK 4 — Sprint N+1 Readiness: Done
  Status: Sprint confirmed
  Violations: None
```

If 0 tasks in next sprint:
```
CHECK 4 — Sprint N+1 Readiness: Not Started
  Status: No tasks in next sprint
  Violations: None
```

If ≥1 task (field validation applies):
```
CHECK 4 — Sprint N+1 Readiness: In Progress
  Status: X tasks pending confirmation
  Violations: @[task name](taskId) — [missing fields]; ...
  (or "Violations: None" if all fields valid)
```
