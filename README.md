# ClickUp Sprint Readiness Compliance Agent

Automated compliance auditing for engineering sprint readiness across 8 product teams. Validates epic setup, backlog hygiene, sprint field completeness, and next-sprint planning in ClickUp.

## Overview

This is a compliance automation system for **Habuild Engineering** that audits the sprint readiness of 8 engineering PODs (Product-Owning Teams):

- **QA** • **Platform** • **Experience** • **Internal Tools** • **Beetu** • **Infra** • **Growth** • **Mobile**

The system runs four automated compliance checks and updates a central dashboard in ClickUp with compliance scores and audit summaries.

### What It Does

Each POD is scored on four compliance dimensions:

1. **Epics Setup** — All in-progress epics must have an assignee
2. **Backlog Hygiene** — All active backlog tasks must be connected to an epic
3. **Key Fields Updated** — All sprint tasks must have 6 required fields set (Assignee, Due Date, Priority, Time Estimate, Sprint Points, Sprint Type)
4. **Sprint N+1 Readiness** — Tasks ready for the next sprint must have all 6 fields set (runs only in the 5 days before sprint end)

Results are color-coded by compliance percentage:
- **≥97%** = Done ✅
- **11–96%** = In Progress 🟨
- **≤10%** = Not Started 🔴

## Architecture

```
User Command (/audit-all or /audit-pod)
  ↓
[Skill: audit-pod / audit-all]
  ├─ For each POD:
  │  ├─ [pod-auditor] discovers active/next sprint lists
  │  ├─ [epics-setup-agent] validates epic assignees
  │  ├─ [backlog-hygiene-and-sprint-fields-agent] validates backlog epic connections + sprint field completeness
  │  ├─ [next-sprint-readiness-agent] validates next-sprint readiness (timing-gated, currently disabled)
  │  └─ [doc-updater] writes results to Sprint Readiness dashboard
  │
  └─ Prints audit summary table
```

**Key design:** Each check runs in an isolated sub-agent context, keeping model prompts focused and costs low. All data flows through the ClickUp MCP server.

## Prerequisites

1. **Claude Code** CLI installed and authenticated
2. **ClickUp MCP server** accessible at `https://mcp.clickup.com/mcp`
3. **ClickUp API token** with workspace access
4. **`jq`** installed (required by the write-guard hooks)
   - macOS: `brew install jq`
   - Linux: `apt-get install jq` or equivalent
   - Windows: Install via `choco install jq` or download from [jqlang.github.io](https://jqlang.github.io)

## Setup

### 1. Configure MCP Connection

Create `.mcp.json` at the repo root:

```json
{
  "mcpServers": {
    "clickup": {
      "type": "http",
      "url": "https://mcp.clickup.com/mcp",
      "headers": {
        "Authorization": "Bearer YOUR_CLICKUP_API_TOKEN_HERE"
      }
    }
  }
}
```

Replace `YOUR_CLICKUP_API_TOKEN_HERE` with your actual token from ClickUp (Settings → Apps → API Token).

### 2. Configure Claude Code Settings

Create `.claude/settings.local.json` inside the `.claude/` directory:

```json
{
  "env": {
    "CLICKUP_API_TOKEN": "YOUR_CLICKUP_API_TOKEN_HERE"
  },
  "enabledMcpjsonServers": ["clickup"],
  "enableAllProjectMcpServers": true,
  "permissions": {
    "allow": [
      "mcp__clickup__clickup_get_workspace_hierarchy",
      "mcp__clickup__clickup_filter_tasks",
      "mcp__clickup__clickup_get_task",
      "mcp__clickup__clickup_update_task",
      "mcp__clickup__clickup_search",
      "mcp__clickup__clickup_get_list",
      "mcp__clickup__clickup_create_task_comment",
      "mcp__clickup__clickup_get_folder",
      "mcp__clickup__clickup_get_custom_fields",
      "Agent(*)",
      "Bash(jq *)"
    ]
  }
}
```

Replace `YOUR_CLICKUP_API_TOKEN_HERE` with your ClickUp API token. This file is gitignored — never commit secrets.

### 3. Verify MCP Connection

```bash
/mcp
```

Should show the `clickup` server as connected. If offline:
- Check network access to `mcp.clickup.com`
- Verify API token is valid in ClickUp workspace settings
- Ensure `.mcp.json` URL is correct

## Usage

### Audit All PODs

```bash
/audit-all
```

Runs the full 4-check compliance audit on all 8 PODs sequentially. Takes ~2–3 minutes. Prints a summary table at the end.

Example output:
```
POD: QA
  CHECK 1 — Epics Setup: 100% → Done
  CHECK 2 — Backlog Hygiene: 95% → In Progress
  CHECK 3 — Key Fields Updated: 98% → Done
  CHECK 4 — Sprint N+1 Readiness: 100% → Done

POD: Platform
  ...
```

### Audit Single POD

```bash
/audit-pod <pod-name>
```

Runs the same 4-check audit on a single POD. Useful for re-auditing after fixes.

```bash
/audit-pod Platform
/audit-pod QA
```

### Check Current Compliance Status

```bash
/check-compliance
```

Read-only. Displays the current compliance status of all 8 PODs **without running a new audit**. Reads the latest values from the Sprint Readiness dashboard.

Use this to quickly check status between audits.

## Compliance Checks Reference

| Check | What It Validates | Exemptions |
|-------|-------------------|-----------|
| **CHECK 1** | Every "IN PROGRESS" epic has an Assignee set | Epics named "Ad-HOC" or starting with "Pending" |
| **CHECK 2** | Every active backlog task is linked to an Epic | Tasks tagged `independent`, `cross-pod`, or `adhoc` |
| **CHECK 3** | Every sprint task has all 6 fields set* | None |
| **CHECK 4** | Every N+1-sprint task has all 6 fields set* | Only runs within 5 days of sprint end; skipped if no tasks or outside window |

*Required fields: Assignee, Due Date, Priority, Time Estimate, Sprint Points, Sprint Type (custom field).

## POD Reference

The 8 PODs audited in order:

| POD | Workspace | Parent Folder |
|-----|-----------|---------------|
| QA | Habuild | Sprint Readiness |
| Platform | Habuild | Sprint Readiness |
| Experience | Habuild | Sprint Readiness |
| Internal Tools | Habuild | Sprint Readiness |
| Beetu | Habuild | Sprint Readiness |
| Infra | Habuild | Sprint Readiness |
| Growth | Habuild | Sprint Readiness |
| Mobile | Habuild | Sprint Readiness |

## Security Model

This agent implements **defense-in-depth** write protection:

### Prompt-Level Constraints
- `CLAUDE.md` defines the root identity and forbidden operations
- All prompts are reviewed to prevent unauthorized writes

### Infrastructure-Level Enforcement
- `.claude/settings.json` uses `PreToolUse` hooks to **physically block** any write operations outside the whitelist
- Only 8 task IDs (the Sprint Readiness dashboard rows) can be updated
- All other ClickUp write operations are rejected at hook time, not by the model

### Secrets Management
- API token stored in `.settings.local.json` (gitignored)
- Internal IDs and field UUIDs are hardcoded in agent files, not exposed to users
- Task IDs and custom field IDs never appear in user-facing output

This design ensures **even a prompt-injected or compromised model cannot modify POD tasks**.

## Project Structure

```
clickup-agent/
├── CLAUDE.md                          # System identity and global constraints
├── README.md                          # This file
├── .gitignore                         # Excludes .mcp.json, settings.local.json
├── .mcp.json                          # MCP server config (API endpoint)
│
└── .claude/
    ├── settings.json                  # Global + project MCP permissions & write-guard hooks
    ├── settings.local.json            # GITIGNORED: API token & local env vars
    │
    ├── agents/                        # Sub-agent definitions
    │   ├── pod-auditor.md             # Sprint discovery (finds active/next sprint)
    │   ├── epics-setup-agent.md       # Epic assignee validation
    │   ├── backlog-hygiene-and-sprint-fields-agent.md # Backlog epic connection + sprint field validation (combined)
    │   ├── next-sprint-readiness-agent.md # Next-sprint field validation (timing-gated, currently disabled)
    │   └── doc-updater.md             # Dashboard writer
    │
    └── skills/                        # User-invocable slash commands
        ├── audit-all/SKILL.md         # /audit-all command
        ├── audit-pod/SKILL.md         # /audit-pod command
        └── check-compliance/SKILL.md  # /check-compliance command
```

## Troubleshooting

### "SKIPPED — space not found"
A POD's ClickUp space is missing or inaccessible. Verify the space exists and your API token has access.

### "Dashboard Doc not found"
The Sprint Readiness dashboard URL is invalid or the doc was deleted. Verify the doc still exists in ClickUp.

### MCP connection timeout
Network issue reaching `mcp.clickup.com`. Check:
- Internet connectivity
- Firewall/proxy rules allowing HTTPS to `mcp.clickup.com`
- API token validity

### BLOCKED: Modifying tasks error
The hook system prevented a write operation. This is expected — verify you're only updating the 8 Sprint Readiness dashboard tasks, not POD tasks.

## Development

### Adding a New POD

1. Add the POD name to `CLAUDE.md` (the "PODs to Audit" section)
2. Add hardcoded IDs to each check agent file:
   - Folder ID (for sprint discovery)
   - Epic list ID (for CHECK 1)
   - Backlog list ID (for CHECK 2)
   - Custom field UUIDs (Epic field, Sprint Type field)
3. Add a new Sprint Readiness dashboard row for the POD (get the task ID)
4. Add the whitelisted task ID to `settings.json`

### Modifying a Compliance Check

Edit the corresponding agent file in `.claude/agents/`. Each agent is self-contained and can be tested independently by invoking it with test data. See the agent file for input/output format.

## Support

For issues or questions:
- Check the **Troubleshooting** section above
- Review `CLAUDE.md` for system constraints
- Run `/check-compliance` to verify dashboard connectivity
- Verify MCP connection with `/mcp`
