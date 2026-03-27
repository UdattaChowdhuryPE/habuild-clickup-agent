---
name: check-compliance
description: Quickly check the current state of the Sprint Readiness list without running a new audit. Reads and displays the current compliance status for all PODs.
---

Read and display the current "Tech Team Sprint Readiness" list without running any new audits.

## Steps
1. Fetch tasks from list `901613935111` at: https://app.clickup.com/9002212861/v/l/6-901613935111-1
   - Use `clickup_get_list` to get list metadata, then `clickup_get_task` for each of the 8 whitelisted task IDs:
     `86d2899n5`, `86d289c19`, `86d289j7m`, `86d289j9h`, `86d289ja7`, `86d289jav`, `86d289jbz`, `86d289jca`
2. Parse each task's custom fields and/or description for compliance values
3. Display the current compliance status for all 8 PODs in a clean table format:

| POD | Epics Setup | Backlog Hygiene | Key Fields Updated | Observations |
|-----|-------------|-----------------|-------------------|--------------|

4. Highlight any PODs with "Not Started" status
5. Count total: Done / In Progress / Not Started across all checks

## No writes — read only.
