# Claude Code Skills

Shared skill library for [Claude Code](https://claude.ai/claude-code). Install globally or per project, then use via slash commands.

Skills check for updates automatically — you'll be prompted when a new version is available.

---

## `/wrap-up` — Session Wrap-Up

> Never leave a session with loose ends.

Everything that should happen before you close a session, automated into one command. Commits work, checks deploys, surfaces what it learned about you, captures deferred work, and presents a scannable summary.

| Step | Action |
|------|--------|
| Git | Commits uncommitted work, pushes to remote |
| Services | Checks Vercel, Sanity, Supabase deploy status |
| About you | Surfaces observations about your working style — you approve what gets remembered |
| Memory | Saves project context, feedback, and references |
| CLAUDE.md | Updates project instructions if the session revealed new patterns |
| Backlog | Captures deferred work, TODOs, and ideas you didn't act on |
| Decisions | Logs product/design/technical decisions in a client-readable format |
| Summary | What was done, what's next, what's open |

```
/wrap-up
```

---

## `/catch-up` — Start With Full Context

> Start every session with full awareness.

When you open a new session, Claude has no idea what happened since last time. `/catch-up` loads recent commits, agent activity, deployments, and migrations so Claude works with full context from the first message.

| Source | What it checks |
|--------|---------------|
| Git | Commits, branches, merges, tags since last check |
| Paperclip | Agent activity — tasks completed, in progress, blocked |
| Vercel | Recent deployments, status, failures |
| Supabase | Database migrations applied or pending |

```
/catch-up                          # Current repo, since last check
/catch-up 3d                       # Last 3 days
/catch-up --all                    # All registered repos
/catch-up ops moon-drop            # Named repos only
```

Zero-config works out of the box (git only). For multi-repo setups, create `~/.claude/catch-up.json`.

---

## `/memory-dream` — Memory Consolidation

> Defragment your AI's memory.

Over time, Claude Code accumulates memory files — observations about you, your projects, your preferences. Some go stale. Some overlap. Some contradict each other. `/memory-dream` tidies everything up so future sessions orient quickly and accurately.

| Action | Description |
|--------|-------------|
| Merge | Combines near-duplicate memory files covering the same topic |
| Prune | Removes stale memories (referenced files moved/deleted, facts outdated) |
| Cluster | Groups scattered small files into coherent topic files |
| Cross-reference | Checks memories against CLAUDE.md — removes redundant entries |
| Verify | Greps the codebase to confirm memories still reflect reality |

```
/memory-dream              # Current project, auto mode
/memory-dream --plan       # Show proposed changes before applying
/memory-dream --all        # All projects
```

---

## Install

### Global (all projects)

Skills go to `~/.claude/skills/` and are available in every Claude Code session.

```bash
git clone https://github.com/moondrop-ro/moondrop-skills.git
cd moondrop-skills
bash install.sh
```

### Per project

Skills go to `<project>/.claude/skills/` and are only available in that project.

```bash
git clone https://github.com/moondrop-ro/moondrop-skills.git
cd moondrop-skills
bash install.sh --project /path/to/your/project
```

### Manual install

Copy any skill folder into your skills directory:

```bash
cp -r skills/wrap-up ~/.claude/skills/       # Global
cp -r skills/wrap-up .claude/skills/          # Per project
```

## Updating

Every skill checks for updates automatically when invoked. If a new version is available, you'll be prompted:

- **Yes, upgrade now** — pulls and re-installs immediately
- **Always keep me up to date** — enables auto-upgrade for future updates
- **Not now** — snoozes with escalating backoff (24h, 48h, 1 week)
- **Never ask again** — disables update checks

Preferences are stored in `~/.moondrop-skills/` and can be changed manually:

```bash
echo true > ~/.moondrop-skills/auto-upgrade     # Enable auto-upgrade
echo false > ~/.moondrop-skills/update-check    # Disable update checks
rm ~/.moondrop-skills/update-check              # Re-enable update checks
```

Manual update:

```bash
cd moondrop-skills && git pull && bash install.sh
```

## Adding Skills

Each skill is a folder inside `skills/` containing a `SKILL.md` file:

```
skills/
└── my-skill/
    └── SKILL.md
```

The folder name becomes the slash command (`/my-skill`). Run `bash install.sh` after adding new skills.
