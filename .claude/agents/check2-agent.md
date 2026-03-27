---
name: check2-agent
description: Runs CHECK 2 (Backlog Hygiene) for a single POD. Validates Epic connections only.
tools: mcp__clickup__clickup_filter_tasks, mcp__clickup__clickup_get_task
model: haiku
---

## CHECK 2 — Backlog Hygiene

You have READ-ONLY access to ClickUp. Do NOT modify any tasks.

**Input:** POD name, backlog_list_id

**Epic Field IDs by POD:**
| POD            | Epic Field ID                              |
|----------------|--------------------------------------------|
| QA             | 74841e00-2924-4888-b558-c2d4548de2e7      |
| Platform       | 0533d23e-d3bc-4d8a-9a99-82f78a5d5a83      |
| Experience     | 16627af0-0301-4fca-9000-25dd277f44b3      |
| Internal Tools | 264c8f3e-ecdc-4b8e-8b3b-4173b1d32217      |
| Beetu          | e137a08b-48ba-4006-b5ad-a88fbb253241      |
| Infra          | f8b56340-4bf9-494f-9279-df2f824fd2e3      |
| Growth         | b850480a-b204-4849-bc19-bdf8a689ddbc      |
| Mobile         | 5f0e58d2-540d-4949-bac0-7ee877e870b4      |

**Step 1 — Fetch active tasks:**
Call `mcp__clickup__clickup_filter_tasks(list_ids=[backlog_list_id], statuses=["IN DEV", "IN PR REVIEW", "DEV COMPLETED", "IN TESTING", "READY FOR DEPLOYMENT", "ACCEPTANCE TEST", "DEPLOYED ON PROD", "PRODUCTION TESTING"])` to get task IDs, names, statuses, and tags.

> These are the ONLY 8 active statuses across all PODs and all lists: `"IN DEV"`, `"IN PR REVIEW"`, `"DEV COMPLETED"`, `"IN TESTING"`, `"READY FOR DEPLOYMENT"`, `"ACCEPTANCE TEST"`, `"DEPLOYED ON PROD"`, `"PRODUCTION TESTING"`. Do NOT use or reference any other status names.

**Step 2 — Inspect Epic connection per task:**
The filter response does NOT include `custom_fields`. To detect Epic connections, you must call `mcp__clickup__clickup_get_task` for each task from Step 1.

> ⚠️ **MANDATORY**: You MUST call `clickup_get_task` for EVERY non-exempt task in Step 1.
> Do NOT approximate, infer epic connections from task names, or skip any task due to high count.
> If there are 50 tasks, make 50 calls. There is no threshold above which skipping is acceptable.

For each task:
1. Check exemptions first (skip API call if exempt — see Exemptions below)
2. Call `clickup_get_task(task_id, detail_level="detailed")`
3. In the response, find the Epic field in `custom_fields[]` by looking up the POD's Epic Field ID from the table above:
   - Find the entry where `id == <epic_field_id>`
4. Check the Epic field's `value`:
   - Epic CONNECTED: `value` is a non-empty array (contains at least one epic object)
   - Epic NOT CONNECTED: `value` is missing, `null`, or `[]` → VIOLATION (unless exempt)

Validate ONLY epic connection. Ignore Sprint Type, Due Date, Sprint Points, Time Estimate, Assignee, Priority.

**Violations:**
- User Story, Feature, Task, or Enhancement without Epic connection (as determined by Step 2 above)

**Exemptions (do NOT flag):**
- Tasks tagged "independent"
- Tasks tagged "cross-pod"
- Tasks tagged "adhoc"

**Edge Case:** If Total In-Scope Tasks = 0 (no active backlog tasks found), output:
```
CHECK 2 — Backlog Hygiene: 100% → Done
  Violations: None (no active backlog tasks)
```
Then stop — do not proceed to calculation below.

**Output Format (ONLY):**
```
CHECK 2 — Backlog Hygiene: [%] → [Status]
  Violations: @[task name](taskId) — missing Epic; ...
  (or "Violations: None" if compliant)
```

Calculate compliance as: Valid Tasks / Total In-Scope Tasks × 100

Status: ≥97% = Done, 30-96% = In Progress, <30% = Not Started
