# Claude Code Skills

Shared skill library for [Claude Code](https://claude.ai/claude-code). Install globally or per project, then use via slash commands.

## Available Skills

| Command | Description |
|---------|-------------|
| `/wrap-up` | End-of-session checklist — commits & pushes uncommitted work, checks service deployments (Vercel, Sanity, Supabase), reviews conversation for memories to persist, updates CLAUDE.md, marks completed checklist items, captures deferred work to backlog, and presents a scannable summary |
| `/catch-up` | Load context about what changed across your projects — recent commits, agent activity, deployments, migrations. Configurable adapters (git, Paperclip, Vercel, Supabase). Zero-config default works with just git. Supports `--all`, `--full`, time windows, and named repos |
| `/memory-dream` | Memory consolidation — merges near-duplicates, prunes stale entries, clusters related feedback, cross-references CLAUDE.md. Auto mode by default (plan-and-wait on first run) |

## Install

### Option A: Global (all projects)

Skills go to `~/.claude/skills/` and are available in every Claude Code session.

```bash
git clone https://github.com/moondrop-ro/claude-skills.git
cd claude-skills
bash install.sh
```

### Option B: Per project

Skills go to `<project>/.claude/skills/` and are only available in that project.

```bash
git clone https://github.com/moondrop-ro/claude-skills.git
cd claude-skills
bash install.sh --project /path/to/your/project
```

### Manual install

Copy any skill folder into your skills directory:

```bash
# Global
cp -r skills/wrap-up ~/.claude/skills/

# Per project
cp -r skills/wrap-up /path/to/project/.claude/skills/
```

## Updating

Pull the latest and re-run install:

```bash
cd claude-skills
git pull
bash install.sh
```

## Adding Skills

Each skill is a folder inside `skills/` containing a `SKILL.md` file:

```
skills/
└── my-skill/
    └── SKILL.md
```

The folder name becomes the slash command (`/my-skill`). Run `bash install.sh` after adding new skills.
