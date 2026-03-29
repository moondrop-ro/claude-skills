---
name: catch-up
description: >
  Load context about what changed across your projects since you last checked.
  Use when starting a session and you want Claude to be aware of recent activity
  from agents, teammates, CI, and deploys. Invoke with /catch-up.
---

# Catch Up

Load context about what happened across the user's projects since they last checked. Your primary goal is to **understand what changed so you can work with full awareness** for the rest of this session. The summary you present to the user is secondary — your own context-loading is the point.

## 1. Parse Arguments

The user invokes this skill with optional arguments:

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

Extract from the arguments:
- **Targets**: `--all`, specific repo names, or none (current repo)
- **Time window**: a duration like `3d`, `12h`, `1w` — or `--full` for entire history — or none (use last check timestamp, falling back to 24h)

One time flag applies to all targets. No per-repo time overrides.

## 2. Read Config

Read `~/.claude/catch-up.json`. If it does not exist, proceed with **zero-config mode**: use git adapter only on the current working directory.

Config schema:

```json
{
  "repos": {
    "<name>": {
      "path": "<absolute path>",
      "adapters": ["git", "paperclip", "vercel", "supabase"]
    }
  },
  "lastCheck": {
    "<name>": "<ISO 8601 timestamp>"
  }
}
```

## 3. Resolve Targets

Determine which repos and adapters to run:

| Argument | Resolution |
|----------|-----------|
| No args | Current working directory. If its path matches a repo in config, use that repo's adapters. Otherwise, use git adapter only. |
| `--all` | Every repo in `config.repos`. If no config exists, error: "No repos configured. Run /catch-up in a git repo, or create ~/.claude/catch-up.json." |
| Named repos (e.g., `ops moon-drop`) | Match each name against keys in `config.repos`. If a name doesn't match, error: "Repo '<name>' not found in ~/.claude/catch-up.json." |

Determine the time boundary for each repo:
- If `--full` flag: no time boundary (all history)
- If duration given (e.g., `3d`): calculate the cutoff timestamp
- If neither: use `config.lastCheck[repoName]`, or 24h ago if no prior check

## 4. Run Adapters

For each target repo, read and follow the adapter instruction files in `adapters/`. Run adapters for a single repo in parallel where possible (they check independent sources).

Available adapters — only run those listed in the repo's config (or just `git` in zero-config mode):

| Adapter | File | Requires |
|---------|------|----------|
| `git` | `adapters/git.md` | Git repo |
| `paperclip` | `adapters/paperclip.md` | Paperclip CLI (`npx paperclipai`) |
| `vercel` | `adapters/vercel.md` | Vercel CLI (`vercel`) |
| `supabase` | `adapters/supabase.md` | Supabase CLI (`npx supabase`) |

For each adapter, read the corresponding `.md` file from the `adapters/` directory relative to this skill file, then follow its instructions. Pass the adapter:
- The repo path
- The time boundary (cutoff timestamp, or "full")

If an adapter's required CLI tool is not available, skip it and note: "Skipped [adapter] — CLI not found."

## 5. Synthesize

Combine all adapter findings into a structured context model. Organize by repo, then within each repo group findings into:

- **What changed** — commits, deploys, agent work completed, migrations applied
- **What's in flight** — open PRs, active tasks, running deploys, pending migrations
- **What looks wrong** — failed CI, blocked tasks, reverted commits, failed deploys

This synthesis is for YOUR understanding. Internalize it. You will carry this context for the rest of the session.

## 6. Present Summary

Show the user a brief summary. Keep it scannable — not a wall of text. Format:

```
## Catch-Up Summary

### <repo-name>
- [2-4 bullet points of the most important findings]
- Flag anything that looks wrong with a warning indicator

### <repo-name>
- ...

Anything I should know before we proceed? (corrections, context, things to ignore)
```

If only one repo was checked, skip the repo heading.

## 7. User Checkpoint

Wait for the user to respond. They may:
- Correct your understanding ("that CI failure is a known flake")
- Add context ("we're in a code freeze until Thursday")
- Flag things to ignore ("don't worry about the blocked tasks")
- Say nothing / "looks good" / "no"

Incorporate everything they say into your context model.

## 8. Save Session Context

Write the final synthesized context (including user corrections) to the current project's memory directory:

**Path:** `~/.claude/projects/<current-project-path>/memory/catch-up-session.md`

Use the standard memory file format:

```markdown
---
name: catch-up-session
description: Active session context from last /catch-up run
type: project
---

## Context loaded: <current datetime>

### <repo-name>
- <findings>

### User notes
- <corrections and context from the user checkpoint>
```

This file is overwritten on each `/catch-up` run — it is a session-scoped snapshot, not an accumulation.

## 9. Update Timestamps

Update `~/.claude/catch-up.json` with the current timestamp for each repo that was checked:

```json
{
  "lastCheck": {
    "<repo-name>": "<current ISO 8601 timestamp>"
  }
}
```

Merge into the existing config — do not overwrite other fields. If no config file exists (zero-config mode), do not create one.

## 10. Continue Session

After saving context and updating timestamps, you are done. Do not summarize again. The user will proceed with their next request, and you will use the loaded context to inform your work throughout the session.
