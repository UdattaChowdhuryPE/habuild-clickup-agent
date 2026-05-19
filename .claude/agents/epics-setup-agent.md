---
name: epics-setup-agent
description: Runs CHECK 1 (Epics Setup) for a single POD. Validates epic Assignee field.
tools: mcp__clickup__clickup_filter_tasks
model: sonnet
---

## CHECK 1 — Epics Setup

You have READ-ONLY access to ClickUp. Do NOT modify any tasks.

**Input:** POD name, epic_list_id, active sprint list ID

**Task:** Call `mcp__clickup__clickup_filter_tasks(list_ids=[epic_list_id], statuses=["IN PROGRESS"])` to fetch all active epics. Validate:

### Edge case:
If the filter returns 0 epics (empty result), output:
```
CHECK 1 — Epics Setup: 100% → Done
  Violations: None (no active epics)
```
Then stop. Do NOT attempt to compute a percentage over an empty set.

- Assignee: required for all epics

**Exemptions (do NOT require Assignee):**
- Epics whose name exactly matches (case-insensitive): "Ad-HOC", "Pending Tasks"
- Epics whose name starts with "Pending" or "Ad-HOC" (covers variants)
- Do NOT exempt any other epics based on guessed intent.
- If an unassigned epic seems catch-all but does not match the above, flag it and note: "May be catch-all — verify manually"

Do NOT check Due Date, Sprint Points, Time Estimate, or Sprint Type on Epics (those are task-level fields).

**Output Format (ONLY):**
```
CHECK 1 — Epics Setup: [%] → [Status]
  Violations: epic name — https://app.clickup.com/t/epicId — [reason]; ...
  (or "Violations: None" if compliant)
```

Calculate compliance as: Valid Epics / Total Active Epics × 100

> ⚠️ STATUS LOCK: Compute status ONLY from this table — never guess:
> ≥97% → Done | 11–96% → In Progress | ≤10% → Not Started
> Example: 50% → In Progress. 8% → Not Started. 97% → Done.

Status: ≥97% = Done, 11-96% = In Progress, ≤10% = Not Started
