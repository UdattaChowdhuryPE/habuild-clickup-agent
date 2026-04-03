---
name: audit-all
description: Audit all 8 PODs in order and update the Sprint Readiness Doc table. Runs the full compliance audit across QA, Platform, Experience, Internal Tools, Beetu, Infra, Growth, Mobile.
---

Run a full sprint readiness compliance audit for all 8 PODs in order.

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

## POD Audit Order
1. QA
2. Platform
3. Experience
4. Internal Tools
5. Beetu
6. Infra
7. Growth
8. Mobile

## Process — For Each POD

### Step 1: Sprint Discovery
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
  WARNING: pod-auditor used fallback sprint for [POD name]. Reason: [fallback_note].
  Sprint date format may be unrecognized or no sprint overlaps today. Verify manually.
  Carry this warning into the observations passed to doc-updater.

If pod-auditor returns an error (no current sprint found):
```
ERROR: Could not identify active sprint for [POD name]. Skipping.
[Continue to next POD]
```

### Step 2: Run CHECK 1
Use the Agent tool to invoke epics-setup-agent as a sub-agent:
- Pass: POD name, `epic_list_id` (from table), `current_sprint_list_id` (from Step 1)
- Receive back: CHECK 1 result block with compliance % and violations

### Step 3: Run Backlog Hygiene + Sprint Fields Checks
Use the Agent tool to invoke backlog-hygiene-and-sprint-fields-agent as a sub-agent with EXACTLY this prompt (substitute values):

```
Run CHECK 2 (Backlog Hygiene) and CHECK 3 (Key Fields Updated) for [POD name] POD.

Workspace ID: [workspace_id]
POD: [POD name]
Backlog List ID: [backlog_list_id]
Current Sprint List ID: [current_sprint_list_id]
```

⚠️ Do NOT add any additional validation instructions, field lists, or status criteria to this prompt.
The agent's own definition governs all logic. Extra instructions override and corrupt it.

Receive back: both CHECK 2 and CHECK 3 result blocks with compliance % and violations

### Step 4: Validate CHECK results
Before proceeding, verify that check1 and the combined check2-check3 agent all returned valid results:
- Each result must contain a line like `CHECK N — Name: [%] → [Status]` with an actual compliance percentage
- If any check agent returned empty, null, or no compliance %, print:
```
ERROR: check[N]-agent returned no data for [POD name]. Stopping audit.
```
Stop processing. Do NOT proceed to subsequent steps or other PODs. Do NOT fill in placeholder/fabricated values.

### Step 4.5: Run CS Checks (Ticket Description & Acceptance Criteria)
Use the Agent tool to invoke cs-checks-agent as a sub-agent:
- Pass: POD name, `current_sprint_list_id` (from Step 1)
- Receive: CS_CHECKS_RESULT with description_total, description_missing_count, description_violations, ac_eligible_count, ac_missing_count, ac_violations

Parse the CS_CHECKS_RESULT output block and extract all six values. Store them for passing to doc-updater in Step 6.

### Step 5: Run CHECK 4 — Sprint N+1 Readiness
Use the Agent tool to invoke next-sprint-readiness-agent as a sub-agent:
- Pass: POD name, `folder_id`, `next_sprint_list_id` (from Step 1), `today`, `current_sprint_end_date` (from Step 1)
- Receive: CHECK 4 result (status: Done / In Progress / Not Started / N/A → outside readiness window, and violations list)

### Step 6: Update Dashboard
Use the Agent tool to invoke doc-updater as a sub-agent. Pass ALL of the following — do NOT omit any field:
- POD: [current POD name]
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

### Step 7: Print progress line
`POD [name]: Epics=[status], Backlog=[status], KeyFields=[status], SprintN+1=[status or N/A], Desc=[count], AC=[count]`

### Step 8: Continue to next POD

If a space is not found at any step: log "SKIPPED — space not found" and continue.

## Final Output
After all 8 PODs, print a summary table:

| POD            | Epics Setup | Backlog Hygiene | Key Fields Updated | Sprint N+1 |
|----------------|-------------|-----------------|-------------------|------------|
| QA             |             |                 |                   |            |
| Platform       |             |                 |                   |            |
| Experience     |             |                 |                   |            |
| Internal Tools |             |                 |                   |            |
| Beetu          |             |                 |                   |            |
| Infra          |             |                 |                   |            |
| Growth         |             |                 |                   |            |
| Mobile         |             |                 |                   |            |

## Constraints
- Do NOT modify any POD tasks
- Process one POD at a time (avoid rate limits)
- `clickup_get_workspace_hierarchy` is ALLOWED for sprint discovery only (invoked by pod-auditor sub-agent)
- Each check runs in its own sub-agent context via the Agent tool
