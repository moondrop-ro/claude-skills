# moondrop-skills

Shared skill library for Claude Code. Skills install to `~/.claude/skills/` (global) or `<project>/.claude/skills/` (per-project) and are invoked as slash commands.

**Repo**: `https://github.com/moondrop-ro/moondrop-skills`
**Owner**: moondrop-ro

## Architecture

```
moondrop-skills/
├── VERSION              # Semver, single line (e.g. "0.1.0")
├── CHANGELOG.md         # Release notes, one ## heading per version
├── install.sh           # Copies skills/ to target dir + VERSION into each skill
├── README.md            # Public-facing docs with skill descriptions
└── skills/
    ├── wrap-up/         # /wrap-up — end-of-session checklist
    │   └── SKILL.md
    └── memory-dream/    # /memory-dream — memory consolidation
        └── SKILL.md
```

## Skill file format

Every skill is a folder inside `skills/` containing a `SKILL.md`. The folder name becomes the slash command.

```markdown
---
name: my-skill
version: 0.1.0
description: >
  One-paragraph description. This is what Claude Code shows in the skill list
  and what determines when the skill triggers.
---

# /my-skill

> **Display Name** — Short tagline.

Description paragraph for humans reading the file.

### What it does
| Step | Action |
|------|--------|
| ... | ... |

### Usage
...

---

[Actual skill instructions for the AI agent follow after the --- separator]
```

### Required sections

1. **YAML frontmatter** — `name`, `version`, `description` (all required)
2. **Description homepage** — human-readable overview with tagline, feature table, usage examples. Separated from instructions by `---`
3. **Preamble — Version Check** — update check block (see below)
4. **Skill instructions** — the actual steps the AI agent follows

## Versioning and update system

### How it works

Every skill includes a `## Preamble — Version Check` section that runs before the skill starts. The flow:

1. Check `~/.moondrop-skills/update-check` — skip if set to `false`
2. Check `~/.moondrop-skills/update-snoozed` — skip if within snooze window
3. Read installed version from `~/.claude/skills/<skill-name>/VERSION`
4. Check cache at `~/.moondrop-skills/last-update-check` (24h TTL)
5. If cache expired: find the git repo, `git fetch origin`, compare `origin/main:VERSION` against installed
6. If update available: read `origin/main:CHANGELOG.md`, compose a 20-word summary of what's new
7. Prompt user with AskUserQuestion (or auto-upgrade if enabled)
8. On upgrade: `git pull`, `bash install.sh`, show full changelog summary

### State files (in `~/.moondrop-skills/`)

| File | Purpose |
|------|---------|
| `last-update-check` | Line 1: unix timestamp. Line 2: `UP_TO_DATE {ver}` or `UPGRADE_AVAILABLE {old} {new} {repo}` |
| `auto-upgrade` | Contains `true` to skip prompts and upgrade automatically |
| `update-check` | Contains `false` to disable all update checking |
| `update-snoozed` | `{version} {level} {timestamp}` — escalating backoff (1=24h, 2=48h, 3+=1wk) |
| `just-upgraded-from` | Previous version, written after successful upgrade |

### install.sh behavior

- Copies each `skills/<name>/` folder to the target skills directory
- Copies `VERSION` into each installed skill folder for version tracking
- Target: `~/.claude/skills/` (default) or `<project>/.claude/skills/` (with `--project`)

## Releasing a new version

1. Make your changes to skill files
2. Update `CHANGELOG.md` — add a new `## X.Y.Z` section at the top with user-facing changes
3. Update `VERSION` to the new version number
4. Update `version:` in each skill's YAML frontmatter to match
5. Commit and push to `main`

Users will be prompted on next skill invocation. The 20-word summary in the prompt is generated from the CHANGELOG diff between their version and the new one.

## Contributing a new skill

1. Create `skills/<skill-name>/SKILL.md`
2. Follow the skill file format above (frontmatter, description homepage, preamble, instructions)
3. Copy the version check preamble from an existing skill — change only the `_INSTALLED` line to read from `$HOME/.claude/skills/<skill-name>/VERSION`
4. Add the skill to `README.md`
5. Test with `bash install.sh` then invoke the slash command

## Rules for AI agents working on this repo

- **Never commit or push automatically** — wait for explicit instruction
- **Keep the preamble identical across skills** except for the skill name in the `_INSTALLED` path
- **Frontmatter version must match VERSION file** — they drift easily, keep them in sync
- **CHANGELOG entries are user-facing** — write them for the person using the skill, not the developer
- **Description homepages matter** — they're the first thing a user sees. Keep them concise, specific, and scannable with tables
- **20-word changelog summary** — when updating the preamble prompt, the summary is composed at runtime from CHANGELOG.md, not hardcoded
