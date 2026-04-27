# /catch-up Skill Design

**Date:** 2026-03-29
**Status:** Draft
**Repo:** moondrop-skills

## Purpose

`/catch-up` is a Claude Code skill that loads context about what happened across the user's projects since they last checked. Its primary purpose is **context-loading for the AI session** — so Claude works with full awareness of recent changes. A brief summary for the user is a side effect. After presenting findings, the user gets a checkpoint to correct or annotate before the context is carried forward for the rest of the session.

## Invocation

```
/catch-up                          # Current repo, since last check (24h if first run)
/catch-up 3d                       # Current repo, last 3 days
/catch-up --full                   # Current repo, entire history
/catch-up --all                    # All registered repos, since last check
/catch-up --all 3d                 # All registered repos, last 3 days
/catch-up --all --full             # All registered repos, entire history
/catch-up ops moon-drop            # Named repos, since last check
/catch-up ops moon-drop --full     # Named repos, entire history
/catch-up ops moon-drop 3d         # Named repos, last 3 days
```

One time flag applies to all targets in the call. No per-repo time overrides.

## File Structure

```
skills/catch-up/
├── SKILL.md                # Core skill — arg parsing, config, orchestration, synthesis, user checkpoint
└── adapters/
    ├── git.md              # Commits, branches, PRs, CI status
    ├── paperclip.md        # Agent activity, task status, heartbeats
    ├── vercel.md           # Deployments, build status
    └── supabase.md         # Migrations, schema changes
```

All adapters ship with the skill. Config determines which are active per repo. Unused adapters are just files sitting there — no cost. Following the gstack model: ship everything, toggle via config.

## Config

Located at `~/.claude/catch-up.json`. Read on every run. If it doesn't exist, the skill works with just git on the current repo — zero-config default.

```json
{
  "repos": {
    "my-app": {
      "path": "/home/user/projects/my-app",
      "adapters": ["git", "vercel", "supabase"]
    },
    "backend": {
      "path": "/home/user/projects/backend",
      "adapters": ["git"]
    },
    "agents": {
      "path": "/home/user/projects/agents",
      "adapters": ["git", "paperclip"]
    }
  },
  "lastCheck": {
    "my-app": "2026-03-29T14:30:00Z",
    "backend": "2026-03-28T09:00:00Z",
    "agents": "2026-03-29T10:15:00Z"
  }
}
```

- Repo names in `repos` are the CLI identifiers (`/catch-up my-app backend`)
- Per-repo timestamps in `lastCheck` so each repo tracks independently
- If current directory matches a registered repo path, that config is used
- If current directory isn't registered, git-only fallback with no config needed

## Flow

```
1.  Parse args (targets, time window, flags)
2.  Read config from ~/.claude/catch-up.json
3.  Resolve which repos + adapters to run
    - No args → current repo (config or git-only fallback)
    - --all → everything in config
    - Named repos → match against config keys
4.  For each repo, run its active adapters in parallel
5.  Synthesize findings into a context model:
    - What changed (commits, deploys, agent work)
    - What's in flight (open PRs, active tasks, running deploys)
    - What looks wrong (failed CI, blocked tasks, reverted commits)
6.  Present brief summary to user
7.  User checkpoint — corrections, annotations, flags
8.  Incorporate user input into final context
9.  Save to session memory (carries forward for rest of session)
10. Update lastCheck timestamps in config
```

Step 7 is the key differentiator — the user steers Claude's understanding before it acts on it. Quick and lightweight, not a review process.

## Adapter Contract

Each adapter is a markdown file with instructions for Claude on how to gather information from one source. Every adapter follows the same structure:

1. **What to check** — the specific commands/queries to run
2. **How to interpret** — what the output means, what to flag
3. **Output shape** — what to report back to the orchestrator

Adapters are not code — they are instructions that Claude follows. Same pattern as a skill, but scoped to one source.

### Shipped Adapters

| Adapter | Source | What it checks |
|---------|--------|---------------|
| `git` | Git history | Commits, branches, open PRs (via `gh`), CI status |
| `paperclip` | Paperclip API/CLI | Agent activity feed, task statuses, blocked tasks, heartbeat history |
| `vercel` | Vercel CLI | Recent deployments, build status, failures |
| `supabase` | Supabase CLI | Migrations, schema changes |

New adapters are added by creating a new `.md` file in `adapters/` and documenting it. Users activate them by adding the adapter name to a repo's `adapters` array in config.

## Session Context Persistence

After the user checkpoint, synthesized context is saved to the **current project's** memory directory at `~/.claude/projects/<current-project>/memory/catch-up-session.md`. Even when `--all` checks multiple repos, the session file lives where the session is running. This is a **session-scoped file** — overwritten on each `/catch-up` run, not accumulated.

```markdown
---
name: catch-up-session
description: Active session context from last /catch-up run
type: project
---

## Context loaded: 2026-03-29 15:00 UTC

### agents
- Agent "Frontend Engineer" shipped hero section component (commit abc123)
- Agent "UI Designer" has 2 blocked tasks waiting on approval
- CI green

### my-app
- Vercel deploy failed 2h ago (build error in api/route.ts)
- 3 new commits on feature/auth branch
- Supabase migration pending: add user_preferences table

### User notes
- CI failure on my-app is known, fix incoming — don't flag it
- Blocked agent tasks are expected, waiting on review
```

The "User notes" section captures user corrections and annotations from the checkpoint. Claude references this file throughout the session when making decisions.

## Design Decisions

| Decision | Choice | Why |
|----------|--------|-----|
| Primary audience | Claude (context-loading) | User wants Claude to work with full awareness, not just a status report |
| Config location | `~/.claude/catch-up.json` | User-specific, machine-specific, not committed to any repo |
| Adapter distribution | Ship all, toggle via config | gstack model — simpler than plugin drop-ins, zero cost for unused adapters |
| Time default (first run) | 24h | Full history could be massive and noisy; `--full` is there when needed |
| Time scope | One flag for all targets | Per-repo time overrides adds complexity without clear value |
| User checkpoint | After synthesis, before session context | User can correct misunderstandings before Claude acts on them |
| Session file | Overwritten per run | Accumulation would create stale context; each run is a fresh snapshot |
