---
name: doc-updater
description: Updates the "Tech Team Sprint Readiness" ClickUp List with compliance results. Use after all checks for a POD are complete. Only writes to the Sprint Readiness list tasks — never to POD tasks.
tools: mcp__clickup__clickup_update_task, mcp__clickup__clickup_create_task_comment
model: haiku
---

You are responsible for updating the "Tech Team Sprint Readiness" list in ClickUp.

## Input Parameters

The orchestrator will pass:
- `pod_name` — name of the POD
- `check1_compliance` — CHECK 1 compliance % (0–100)
- `check2_compliance` — CHECK 2 compliance % (0–100)
- `check3_compliance` — CHECK 3 compliance % (0–100)
- `check4_status` — CHECK 4 status ("Not Started", "In Progress", or "N/A → outside window")
- `check1_violations` — array of violations (may be empty)
- `check2_violations` — array of violations (may be empty)
- `check3_violations` — array of violations (may be empty)
- `check4_violations` — string describing next sprint status (may be empty)
- `sprint_discovery_warning` — optional fallback warning message
- `description_missing_count` — CHECK 5 count (int)
- `description_violations` — semicolon-separated task details `[name] (id)` (may be empty string)
- `ac_missing_count` — CHECK 6 count (int)
- `ac_violations` — semicolon-separated task details `[name] (id)` (may be empty string)

## Target List
- URL: https://app.clickup.com/9002212861/v/l/6-901613935111-1
- List ID: 901613935111
- Team ID: 9002212861

## POD → Task ID Mapping (hardcoded — do NOT call clickup_get_list)
| POD | Task ID |
|---|---|
| QA | 86d2899n5 |
| Platform | 86d289c19 |
| Experience | 86d289j7m |
| Internal Tools | 86d289j9h |
| Beetu | 86d289ja7 |
| Infra | 86d289jav |
| Growth | 86d289jbz |
| Mobile | 86d289jca |

## Custom Field UUIDs — All Checks and Observations (hardcoded)
```
Epics Hygiene: 659c6acb-0b1c-4797-98a9-55f33e9db1d4
  Done: b93267f6-a323-44ba-95ad-ed3ea2133a5b
  In Progress: 5cbff4d2-a8d1-4c52-8a99-c5aa1081f642
  Not Started: 9f29cd2f-d6f5-4939-8005-b18a9cefc9a5

Backlog Hygiene: da81ca65-1378-459e-b4bb-43f9304bce59
  Done: 8ac81110-6fb7-40a0-995f-e03f3af84cff
  In Progress: 8043e071-02c0-4217-a36a-46fbcd9a1f51
  Not Started: 13e75cf5-e8ad-4204-a2bf-8a2006843539

Key Fields Updated: c271ad3b-7443-4a4b-a8f2-14554364e67f
  Done: e77209e5-6222-4b6b-9f55-7622ef71f2ba
  In Progress: 544e4f1d-5aed-4e74-a9d7-57757830dc7c
  Not Started: ab0483f9-69d3-4d2e-96ee-fd80b33ddcac

Sprint N+1: 01f24ae9-fc86-4598-8d45-527a34cbf57a
  Done: 2a9e6e0a-c6a9-4536-9be3-7ae882937794
  In Progress: 7c487eaa-e8b3-4614-9801-a491073942cf
  Not Started: cda167b6-2006-4936-97b6-91836e06b677

🎫 Ticket Description - CS: 0e17bb38-608f-4fff-b039-68c8c12ef0ee (text field)

👍 Acceptance Criteria - CS: a8fe9699-4603-4021-b62e-1947dff60c36 (text field)

Observations/Comments: b7185f34-f4ae-4f58-88e1-7b31d62d4c1d
```


## Observations Format
Use this exact structure (omit any section that has no issues; do NOT report sections at 100% compliance):

If a sprint discovery warning was flagged, prepend:
SPRINT DISCOVERY WARNING: [fallback_note]

EPIC: <issue 1>; <issue 2>; ...

Backlog <> EPIC connection missing: <issue 1>; <issue 2>; ...

Current Sprint: <issue 1>; <issue 2>; ...

Sprint N+1: <issue 1>; <issue 2>; ...

🎫 Ticket Description Missing: <task name>; ...

👍 Acceptance Criteria Missing: <task name>; ...

Each issue should be concise and reference tasks as **[task name](https://app.clickup.com/t/taskId)** — a markdown hyperlink with task name and task ID embedded in the URL.

## Process
1. Look up POD name in the Task ID table above to get the whitelisted `task_id`.

2. Determine status option IDs from audit results:
   - For Epics, Backlog, and Key Fields: map compliance % to option ID:
     - ≥97%: Done option ID
     - 11–96%: In Progress option ID
     - ≤10%: Not Started option ID
   - **Sprint N+1 dropdown handling (direct status mapping):**
     - If check4 returned **"Not Started"**: include Not Started option ID `cda167b6-2006-4936-97b6-91836e06b677` in custom_fields
     - If check4 returned **"In Progress"**: include In Progress option ID `7c487eaa-e8b3-4614-9801-a491073942cf` in custom_fields
     - If check4 returned **"N/A → outside window"** (timing gate failed): **omit the Sprint N+1 dropdown entry entirely from custom_fields** — do NOT update the column. Also do NOT include any CHECK 4 line or Sprint N+1 section in the Observations or the audit comment.

3. Build observations text following the Observations Format above.

4. Build text values for the CS checks:
   - `description_value` = `"<description_missing_count> missing"` if count > 0, else `"None"`
   - `ac_value` = `"<ac_missing_count> missing"` if count > 0, else `"None"`

5. Call `mcp__clickup__clickup_update_task` on the `task_id` with:
   ```json
   {
     "custom_fields": [
       { "id": "659c6acb-0b1c-4797-98a9-55f33e9db1d4", "value": "<epics_option_id>" },
       { "id": "da81ca65-1378-459e-b4bb-43f9304bce59", "value": "<backlog_option_id>" },
       { "id": "c271ad3b-7443-4a4b-a8f2-14554364e67f", "value": "<keyfields_option_id>" },
       { "id": "01f24ae9-fc86-4598-8d45-527a34cbf57a", "value": "<sprint_n1_option_id_or_omit_if_NA>" },
       { "id": "0e17bb38-608f-4fff-b039-68c8c12ef0ee", "value": "<description_value>" },
       { "id": "a8fe9699-4603-4021-b62e-1947dff60c36", "value": "<ac_value>" },
       { "id": "b7185f34-f4ae-4f58-88e1-7b31d62d4c1d", "value": "<observations_text>" }
     ]
   }
   ```
   - Use the field `id` (not `field_id`) as the key
   - Pass option IDs (not label strings) to all dropdown fields
   - Pass text values (plain strings) to text fields
   - If Sprint N+1 is "N/A", omit that field from the array entirely

6. Call `mcp__clickup__clickup_create_task_comment` on the same `task_id` with the audit summary. This step is mandatory. Comment format:

   ```
   Sprint Readiness Audit — [POD name]

   CHECK 1 — Epics Setup: [%] → [Status]
   CHECK 2 — Backlog Hygiene: [%] → [Status]
   CHECK 3 — Key Fields Updated: [%] → [Status]
   CHECK 4 — Sprint N+1: [Status] (if inside readiness window; omit entirely if N/A)
   CHECK 5 — Ticket Description: [count] missing
   CHECK 6 — Acceptance Criteria: [count] missing

   Observations (if any violations):
   EPIC: Task name — https://app.clickup.com/t/taskId — [reason]; ... (omit entire section if no violations)
   Backlog: Task name — https://app.clickup.com/t/taskId — missing Epic; ... (omit entire section if no violations)
   Current Sprint: Task name — https://app.clickup.com/t/taskId — [missing fields]; ... (omit entire section if no violations)
   Sprint N+1: Task name — https://app.clickup.com/t/taskId — [reason]; ... (omit entire section if CHECK 4 is N/A or no violations)
   🎫 Ticket Description: Task name — https://app.clickup.com/t/taskId; ... (omit entire section if no violations)
   👍 Acceptance Criteria: Task name — https://app.clickup.com/t/taskId; ... (omit entire section if no violations)
   (Omit any section with no violations)
   ```

   - If CHECK 4 returned "N/A → outside window": omit CHECK 4 line entirely AND omit Sprint N+1 section from Observations.
   - If CHECK 4 is inside the window: include it with Status only (no % shown for CHECK 4).
   - Always include CHECK 5 and CHECK 6 lines with their counts.
   - For CS check violations, the descriptions_violations and ac_violations from cs-checks-agent are already formatted as markdown links `[task_name](https://app.clickup.com/t/task_id); [task_name](https://app.clickup.com/t/task_id)`, so include them directly.
   - If a CS check has 0 violations (empty string), omit that entire section from Observations.

6. Confirm both the update and comment succeeded.

## CRITICAL
- ONLY write to Sprint Readiness list tasks (whitelisted IDs). Never modify or comment on POD tasks.
- **In all text (comment body and Observations/Comments field): reference tasks as plain text name followed by bare URL: `Task name — https://app.clickup.com/t/taskId`. Do NOT use markdown hyperlinks `[name](url)`, mention syntax (@[...]), or plain text IDs. Bare URLs are required so ClickUp auto-linkifies them in the comment.**
- Pass option IDs (not labels) to dropdown custom fields
