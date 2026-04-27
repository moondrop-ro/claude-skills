---
name: catch-up
version: 0.1.0
description: >
  Load context about what changed across your projects since you last checked.
  Use when starting a session and you want Claude to be aware of recent activity
  from agents, teammates, CI, and deploys. Invoke with /catch-up.
---

# /catch-up

> **Catch Up** — Start every session with full awareness.

When you open a new session, Claude has no idea what happened since last time. `/catch-up` fixes that. It loads recent commits, agent activity, deployments, and migrations so Claude works with full context from the first message.

### What it does

| Source | What it checks |
|--------|---------------|
| Git | Commits, branches, merges, tags since last check |
| Paperclip | Agent activity — tasks completed, in progress, blocked |
| Vercel | Recent deployments, status, failures |
| Supabase | Database migrations applied or pending |

### Usage

```
/catch-up                          # Current repo, since last check
/catch-up 3d                       # Last 3 days
/catch-up --full                   # Entire history
/catch-up --all                    # All registered repos
/catch-up --all 3d                 # All repos, last 3 days
/catch-up ops moon-drop            # Named repos only
```

### Configuration

Zero-config works out of the box (git only). For multi-repo + multi-adapter setup, create `~/.claude/catch-up.json`:

```json
{
  "repos": {
    "my-app": {
      "path": "/home/user/projects/my-app",
      "adapters": ["git", "vercel", "supabase"]
    }
  }
}
```

---

# Catch Up

Load context about what happened across the user's projects since they last checked. Your primary goal is to **understand what changed so you can work with full awareness** for the rest of this session. The summary you present to the user is secondary — your own context-loading is the point.

## Preamble — Version Check

Before starting the skill, check if moondrop-skills has an update available. This should be fast and non-blocking — use cached results when possible.

```bash
_MS_STATE="$HOME/.moondrop-skills"
mkdir -p "$_MS_STATE"

# Skip if checks disabled
if [ -f "$_MS_STATE/update-check" ] && [ "$(cat "$_MS_STATE/update-check")" = "false" ]; then
  echo "UPDATE_CHECK_DISABLED"
  exit 0
fi

# Skip if snoozed
if [ -f "$_MS_STATE/update-snoozed" ]; then
  _SV=$(awk '{print $1}' "$_MS_STATE/update-snoozed")
  _SL=$(awk '{print $2}' "$_MS_STATE/update-snoozed")
  _ST=$(awk '{print $3}' "$_MS_STATE/update-snoozed")
  _NOW=$(date +%s)
  case "$_SL" in 1) _D=86400;; 2) _D=172800;; *) _D=604800;; esac
  if [ $((_NOW - _ST)) -lt "$_D" ]; then
    echo "SNOOZED until $(date -d @$((_ST + _D)) '+%Y-%m-%d %H:%M' 2>/dev/null || echo 'later')"
    exit 0
  fi
fi

# Get installed version
_INSTALLED=$(cat "$HOME/.claude/skills/catch-up/VERSION" 2>/dev/null || echo "0.0.0")

# Use cache if fresh (24h TTL)
_CACHE="$_MS_STATE/last-update-check"
if [ -f "$_CACHE" ]; then
  _CT=$(awk 'NR==1{print $1}' "$_CACHE")
  _NOW=$(date +%s)
  if [ $((_NOW - _CT)) -lt 86400 ]; then
    cat "$_CACHE" | tail -n +2
    exit 0
  fi
fi

# Find repo and fetch
_MS_REPO=""
for _C in "$HOME/.moondrop-skills/repo" "$HOME/moondrop-skills" "$HOME/projects/moondrop-skills" "$HOME/dev/moondrop-skills" "$HOME/src/moondrop-skills" "$HOME/code/moondrop-skills"; do
  [ -d "$_C/.git" ] && _MS_REPO="$_C" && break
done

if [ -z "$_MS_REPO" ]; then
  echo "NO_REPO"
  exit 0
fi

cd "$_MS_REPO"
git fetch origin --quiet 2>/dev/null
_REMOTE=$(git show origin/main:VERSION 2>/dev/null | tr -d '[:space:]')
_NOW=$(date +%s)

if [ -n "$_REMOTE" ] && [ "$_INSTALLED" != "$_REMOTE" ]; then
  _RESULT="UPGRADE_AVAILABLE $_INSTALLED $_REMOTE $_MS_REPO"
else
  _RESULT="UP_TO_DATE $_INSTALLED"
fi

printf "%s\n%s\n" "$_NOW" "$_RESULT" > "$_CACHE"
echo "$_RESULT"
```

**If `UPGRADE_AVAILABLE {old} {new} {repo}`:**

First, read the changelog to build a short summary of what's new:
```bash
cd "{repo}"
git show origin/main:CHANGELOG.md 2>/dev/null || echo "(no changelog)"
```
From the changelog, extract changes between v{old} and v{new}. Compose a **one-line summary of 20 words or fewer** describing the most interesting user-facing changes. This is the `{whats_new}` summary.

Check auto-upgrade preference:
```bash
_AUTO=""
[ -f "$HOME/.moondrop-skills/auto-upgrade" ] && _AUTO=$(cat "$HOME/.moondrop-skills/auto-upgrade")
echo "AUTO_UPGRADE=$_AUTO"
```

**If `AUTO_UPGRADE=true`:** Log "Auto-upgrading moondrop-skills v{old} -> v{new}... ({whats_new})" and run the upgrade (see below). If it fails, warn and continue with the skill.

**Otherwise**, use AskUserQuestion:
- Question: "moondrop-skills **v{new}** is available (you're on v{old}): *{whats_new}*. Upgrade now?"
- Options: ["Yes, upgrade now", "Always keep me up to date", "Not now", "Never ask again"]

| Response | Action |
|----------|--------|
| **Yes, upgrade now** | Run upgrade below |
| **Always keep me up to date** | `echo "true" > "$HOME/.moondrop-skills/auto-upgrade"` — tell user auto-upgrade enabled, then run upgrade |
| **Not now** | Write snooze: `echo "{new} {level} $(date +%s)" > "$HOME/.moondrop-skills/update-snoozed"` (level 1=24h, 2=48h, 3+=1wk). Tell user when next reminder will be. Continue with the skill. |
| **Never ask again** | `echo "false" > "$HOME/.moondrop-skills/update-check"` — tell user checks disabled. Continue with the skill. |

**Upgrade steps:**
```bash
cd "{repo}"
_OLD=$(cat VERSION 2>/dev/null || echo "unknown")
git stash 2>&1
git pull origin main --ff-only 2>&1 || git reset --hard origin/main
bash install.sh
rm -f "$HOME/.moondrop-skills/update-snoozed"
echo "$_OLD" > "$HOME/.moondrop-skills/just-upgraded-from"
```

After upgrading, read `{repo}/CHANGELOG.md`, summarize changes between old and new versions as 3-7 bullets, then display:
```
moondrop-skills v{new} — upgraded from v{old}!

What's new:
- ...

Happy skilling!
```

Then continue with the skill below.

**If `UP_TO_DATE`, `SNOOZED`, `UPDATE_CHECK_DISABLED`, or `NO_REPO`:** Continue silently with the skill.

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
