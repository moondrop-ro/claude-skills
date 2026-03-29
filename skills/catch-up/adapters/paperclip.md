# Paperclip Adapter

Gather recent agent activity from a Paperclip-managed company.

## Inputs

- **Repo path**: absolute path to the repository (used to find Paperclip config)
- **Time boundary**: ISO 8601 cutoff timestamp, or "full" for entire history

## Prerequisites

This adapter requires:
- Paperclip CLI available (`npx paperclipai`)
- A Paperclip company ID — check the repo's `CLAUDE.md` for a Company ID field, or look for a `config/company.md` file in the repo

If no company ID can be found, skip this adapter and note: "Paperclip adapter skipped — no company ID found."

## What to Check

### 1. Recent activity

```bash
npx paperclipai activity list --company-id <company-id>
```

This returns a chronological feed of agent actions: task status changes, comments, assignments, completions.

Filter to entries after the cutoff timestamp. If `--full`, include all activity.

### 2. Current task statuses

```bash
npx paperclipai issue list --company-id <company-id>
```

Report tasks grouped by status: `todo`, `in_progress`, `blocked`, `done`, `in_review`.

### 3. Agent roster

```bash
npx paperclipai agent list --company-id <company-id>
```

Cross-reference with activity to identify which agents have been active vs idle.

### 4. Dashboard overview

```bash
npx paperclipai dashboard --company-id <company-id>
```

Captures high-level company health: budget utilization, active agents, task throughput.

## How to Interpret

- **Completed tasks**: what agents shipped since the cutoff. Note the deliverables.
- **In-progress tasks**: what agents are currently working on. Note any that have been in-progress for a long time without updates.
- **Blocked tasks**: flag prominently — these need human or upstream intervention.
- **Agent activity patterns**: which agents are active, which are idle. Idle agents with assigned tasks may indicate a problem.
- **Budget**: if any agent is above 80% budget utilization, flag it.

## Output Shape

Report back to the orchestrator:

- **Completed work**: what agents delivered since the cutoff
- **In-flight work**: what agents are currently doing
- **Blocked items**: tasks that need intervention, with the blocking reason
- **Agent health**: budget warnings, idle agents with work assigned
- **Overall status**: 1-2 sentence summary of the org's progress
