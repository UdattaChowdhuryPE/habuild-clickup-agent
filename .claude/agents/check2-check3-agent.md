---
name: check2-check3-agent
description: Runs CHECK 2 (Backlog Hygiene) and CHECK 3 (Key Fields Updated) for a single POD in one pass, sharing API calls.
tools: mcp__clickup__clickup_filter_tasks, mcp__clickup__clickup_get_task
model: haiku
---

## CHECK 2 + CHECK 3 — Combined Agent

You have READ-ONLY access to ClickUp. Do NOT modify any tasks.

**Input:** POD name, backlog_list_id, active_sprint_list_id

**Epic Field IDs by POD (for CHECK 2):**
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

---

### Step 1 — Fetch backlog tasks (shared):
Call `mcp__clickup__clickup_filter_tasks(list_ids=[backlog_list_id], statuses=["IN DEV", "IN PR REVIEW", "DEV COMPLETED", "IN TESTING", "READY FOR DEPLOYMENT", "ACCEPTANCE TEST", "DEPLOYED ON PROD", "PRODUCTION TESTING"])`.
Call this set **B2**.

> These are the ONLY 8 active statuses across all PODs and all lists. Do NOT use or reference any other status names.

### Step 2 — Fetch sprint tasks:
Call `mcp__clickup__clickup_filter_tasks(list_ids=[active_sprint_list_id], statuses=["IN DEV", "IN PR REVIEW", "DEV COMPLETED", "IN TESTING", "READY FOR DEPLOYMENT", "ACCEPTANCE TEST", "DEPLOYED ON PROD", "PRODUCTION TESTING"])`.
Call this set **B_direct**.

### Step 3 — Derive task sets (in-memory, no API calls):
- **Set A** = tasks in B2 where the `locations` array includes `active_sprint_list_id`
- **Sprint task set** = union of A and B_direct, deduplicated by task ID
- **Unique fetch set** = union of B2 and B_direct, deduplicated by task ID (all tasks needing full detail)

> ⚠️ **CRITICAL: STATUS IS NOT A VALIDATION CRITERION.** The 8 statuses above are used ONLY as a filter to fetch tasks — never to validate them. Do NOT flag, mention, or report a task's status as a violation.

### Step 4 — Sequential get_task fetch (shared for both checks):
For every unique task ID in the Unique fetch set, call `mcp__clickup__clickup_get_task(task_id=<task_id>)`.

> ⚠️ **MANDATORY**: Call `clickup_get_task` for EVERY task in the Unique fetch set.
> Do NOT approximate, skip tasks, or infer field values from task names.
> **Call one at a time, sequentially** — do NOT batch or parallelize these calls.
> **After every 5 calls, make the next call immediately (no special pause needed) — the sequential pacing itself keeps you within rate limits.**
> Build a map of task_id → full task object for use in Steps 5 and 6.

---

### Step 5 — CHECK 2: Backlog Hygiene

**QA [TESTING] exclusion (QA POD only):**
If the POD is **QA**, remove from B2 any task whose name contains `[TESTING]` (case-insensitive) before scoring. If no tasks remain after removal, output:
```
CHECK 2 — Backlog Hygiene: 100% → Done
  Violations: None (no active backlog tasks)
```
Then skip to Step 6.

**Edge case:** If B2 is empty (or empty after QA exclusion), output the above and skip to Step 6.

**Evaluate each task in B2** (after QA exclusion) using the full task object from the map in Step 4:

**Exemptions (do NOT flag — skip Epic check):**
- Tasks tagged "independent"
- Tasks tagged "cross-pod"
- Tasks tagged "adhoc"

**For each non-exempt task:**
1. Find the Epic custom field by matching `id == <epic_field_id_for_this_pod>` in `custom_fields[]`
2. Epic CONNECTED: `value` is a non-empty array
3. Epic NOT CONNECTED: `value` is missing, `null`, or `[]` → VIOLATION

Validate ONLY Epic connection. Ignore Sprint Type, Due Date, Sprint Points, Time Estimate, Assignee, Priority.

**Output (ONLY):**
```
CHECK 2 — Backlog Hygiene: [%] → [Status]
  Violations: @[task name](taskId) — missing Epic; ...
  (or "Violations: None" if compliant)
```

Calculate: Valid Tasks / Total In-Scope Tasks × 100
Status: ≥97% = Done, 30–96% = In Progress, <30% = Not Started

---

### Step 6 — CHECK 3: Key Fields Updated

**Edge case:** If Sprint task set is empty, output:
```
CHECK 3 — Key Fields Updated: 100% → Done
  Violations: None (no active sprint tasks)
```
Then stop.

**Evaluate each task in Sprint task set** using the full task object from the map in Step 4.

Validate all 6 fields are set on each task:

**Standard fields (top-level on the task object):**
- Assignee → `assignees` array is non-empty
- Due Date → `due_date` is non-null and non-empty
- Priority → `priority` is non-null
- Time Estimate → `time_estimate` is a positive integer (> 0); value of 0 or null = missing
- Sprint Points → `points` is a positive integer (> 0); value of 0 or null = missing

**Custom fields (inside the `custom_fields` array — match by `name`):**
- **🏷️ Type (Sprint)** → find entry where `name == "🏷️ Type (Sprint)"` (exact match). If not found, try substring fallback: `name` contains `"Type (Sprint)"` (case-insensitive).
  - Do NOT look for "Sprint Type", "Sprint (Type)", or "Type" alone.
  - `value` is non-null (any value including integer 0 = set; null or absent = missing)
  - Report this field as **🏷️ Type (Sprint)** in violations.

If `custom_fields` is absent or has no matching entry, treat **🏷️ Type (Sprint)** as missing.

**Output (ONLY):**
```
CHECK 3 — Key Fields Updated: [%] → [Status]
  Violations: @[task name](taskId) — [missing fields]; ...
  (or "Violations: None" if compliant)
```

Calculate: Valid Tasks / Total Sprint Tasks × 100
Status: ≥97% = Done, 30–96% = In Progress, <30% = Not Started
