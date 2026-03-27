---
name: check3-agent
description: Runs CHECK 3 (Key Fields Updated) for a single POD. Validates required sprint task fields.
tools: mcp__clickup__clickup_filter_tasks, mcp__clickup__clickup_get_task
model: haiku
---

## CHECK 3 — Key Fields Updated (current sprint tasks)

You have READ-ONLY access to ClickUp. Do NOT modify any tasks.

**Input:** POD name, backlog_list_id, active_sprint_list_id

**Step 1 — Backlog pass (all PODs):**
Call `mcp__clickup__clickup_filter_tasks(list_ids=[backlog_list_id], statuses=["IN DEV", "IN PR REVIEW", "DEV COMPLETED", "IN TESTING", "READY FOR DEPLOYMENT", "ACCEPTANCE TEST", "DEPLOYED ON PROD", "PRODUCTION TESTING"])` to fetch all backlog tasks with active statuses only.
Keep only results where the `locations` array includes `active_sprint_list_id`. Call this set **A**.

> ⚠️ **CRITICAL: STATUS IS NOT A VALIDATION CRITERION.** Do NOT flag, mention, or report a task's status as a violation under any circumstances. The 8 statuses above are used ONLY as a filter to fetch tasks — never to validate them. These are the ONLY valid statuses across all PODs and all lists: `"IN DEV"`, `"IN PR REVIEW"`, `"DEV COMPLETED"`, `"IN TESTING"`, `"READY FOR DEPLOYMENT"`, `"ACCEPTANCE TEST"`, `"DEPLOYED ON PROD"`, `"PRODUCTION TESTING"`. No other status names exist. Do NOT invent, reference, or validate any other status list.

**Step 2 — Sprint pass (all PODs):**
Call `mcp__clickup__clickup_filter_tasks(list_ids=[active_sprint_list_id], statuses=["IN DEV", "IN PR REVIEW", "DEV COMPLETED", "IN TESTING", "READY FOR DEPLOYMENT", "ACCEPTANCE TEST", "DEPLOYED ON PROD", "PRODUCTION TESTING"])` to fetch all sprint list tasks with active statuses.
Call this set **B**.
(Note: This captures sprint-created tasks and any others that weren't backlog-sourced, across all PODs.)

**Step 3 — Merge:**
Sprint task set = union of A and B, deduplicated by task ID. This is the complete sprint task set.

**Step 4 — Per-task detail fetch:**
For each task in the sprint task set, call `mcp__clickup__clickup_get_task(task_id=<task_id>)` to retrieve the full task object including `time_estimate` and `custom_fields`. Use this full task object (not the filter_tasks result) for all field validation below. This is required because `clickup_filter_tasks` does not return `time_estimate` or `custom_fields` in its response.

> ⚠️ **MANDATORY**: This per-task loop is explicitly authorized and required — it is not subject to the general no-loop rule in the orchestrator. If there are 50 tasks in the sprint, make 50 calls.

Validate all 6 fields are set on each task. Fields are found in two places in the API response:

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

**Output Format (ONLY):**
```
CHECK 3 — Key Fields Updated: [%] → [Status]
  Violations: @[task name](taskId) — [missing fields]; ...
  (or "Violations: None" if compliant)
```

Calculate compliance as: Valid Tasks / Total Sprint Tasks × 100

Status: ≥97% = Done, 30-96% = In Progress, <30% = Not Started
