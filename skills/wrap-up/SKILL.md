---
name: wrap-up
version: 0.2.0
description: >
  End-of-session checklist — commits & pushes uncommitted work, checks service
  deployments (Vercel, Sanity, Supabase), surfaces what it learned about you,
  reviews conversation for memories to persist, updates CLAUDE.md, captures
  deferred work to backlog, logs client-facing decisions, and presents a
  scannable summary. Use when ending a session.
---

# /wrap-up

> **Session Wrap-Up** — Never leave a session with loose ends.

> _Source: [moondrop-skills](https://github.com/moondrop-ro/moondrop-skills) — the user's personal skill library at `D:\projects\moondrop-skills`. Distinct from any gstack-shipped skill of the same name; do not overwrite during third-party skill-pack reinstalls._

Everything that should happen before you close a session, automated into one command. `/wrap-up` handles the boring-but-critical end-of-session hygiene so nothing falls through the cracks.

### What it does

| Step | Action |
|------|--------|
| Git | Commits uncommitted work, pushes to remote |
| Services | Checks Vercel, Sanity, Supabase deploy status |
| About you | Surfaces observations about your working style — you approve what gets remembered |
| Memory | Saves project context, feedback, and references to memory |
| CLAUDE.md | Updates project instructions if the session revealed new patterns |
| Checklists | Marks completed items in active plans |
| Backlog | Captures deferred work, TODOs, and ideas you didn't act on |
| Decisions | Logs product/design/technical decisions in a client-readable format |
| Summary | What was done, what's next, what's open |

### Usage

```
/wrap-up
```

No arguments. Just run it before closing.

---

End-of-session checklist. Run every check below, execute the actions, and present a summary to the user before they close the session.

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
_INSTALLED=$(cat "$HOME/.claude/skills/wrap-up/VERSION" 2>/dev/null || echo "0.0.0")

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
# Check pointer file first (contains repo path)
if [ -f "$_MS_STATE/repo" ]; then
  _PTR=$(cat "$_MS_STATE/repo" | tr -d '[:space:]')
  [ -d "$_PTR/.git" ] && _MS_REPO="$_PTR"
fi
# Fall back to well-known locations
if [ -z "$_MS_REPO" ]; then
  for _C in "$HOME/moondrop-skills" "$HOME/projects/moondrop-skills" "$HOME/dev/moondrop-skills" "$HOME/src/moondrop-skills" "$HOME/code/moondrop-skills"; do
    [ -d "$_C/.git" ] && _MS_REPO="$_C" && break
  done
fi

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

## 1. Git Status

Run `git status` and `git log @{upstream}..HEAD --oneline 2>/dev/null` (to check unpushed commits).

Report:
- **Uncommitted changes**: list modified/untracked files, or "clean"
- **Unpushed commits**: list commit subjects, or "in sync with remote"
- **Current branch**: confirm which branch we're on

If there are uncommitted changes, **ask the user** whether to commit and push. Use AskUserQuestion with options: "Commit & push", "Commit only", "Skip — I'll handle it". If the user chooses to commit, use a descriptive message summarizing the session's work.

## 2. Service Deployments

After pushing, check and report on all connected services:

- **Vercel**: Note that push triggers auto-deploy. If Vercel CLI is available, run `vercel ls --limit 1` to confirm latest deployment status.
- **Sanity**: If Sanity schema files were modified this session, remind the user to run `npx sanity@latest schema deploy` (or offer to run it). If no schema changes, report "No Sanity schema changes."
- **Supabase**: If any migration SQL files were created or modified, remind the user to apply them. If no database changes, report "No Supabase migration changes."
- **Other services**: If the project uses additional services (e.g., external APIs with keys added/changed), note any env var or configuration updates needed.

Report the status of each service, or "No updates needed" for services unaffected by the session.

## 3. What I Learned About You

Before saving any memories, **present what you observed about the user this session**. This is the most important part of wrap-up — the user should see themselves reflected back and approve what gets remembered.

**How to surface observations:**

Review the full conversation and identify:
- **Working style shifts**: Did the user do something differently than before? (e.g., ran commands independently for the first time, asked deeper questions, changed their approach)
- **New preferences or corrections**: Did they say "don't do X" or "I prefer Y"? Did they approve a non-obvious approach that should be repeated?
- **Values revealed through decisions**: What did their choices tell you about what they care about? (e.g., chose architecture over speed, prioritized accessibility, asked about the system before running it)
- **Growth or learning**: Did they learn something new this session? Did their questions evolve from "what" to "why" or "how"?

**The distinction that matters:** Actions are evidence, not observations. "You batched 3 related backlog items" is a summary of what happened. "You think in dependency chains, not priority lists" is an insight about who you are. Only surface insights — things that change how I should work with you in future sessions.

**Bad:** "You picked up 3 backlog items and asked me to fix them all" (action replay)
**Good:** "You have zero tolerance for deferred cleanup — when you see tech debt, you want it gone now, not next session" (personality insight that changes my behavior)

Only include observations that would actually change how you work with this person.

**Present via interactive multi-select using AskUserQuestion:**

Use `AskUserQuestion` with `multiSelect: true`. Each observation becomes one option:
- **label**: The insight (concise, 5-10 words)
- **description**: The behavioral impact — explain concretely what changes. Format: "**Without this:** [current behavior]. **With this:** [improved behavior]." The user should be able to read just the description and understand why this memory is worth keeping.

**Example option:**
- label: "Zero tolerance for deferred cleanup"
- description: "**Without this:** I'd suggest adding dead code to the backlog for later. **With this:** I'd proactively delete dead code during the session instead of deferring it."

If the session revealed nothing new about the user, skip the AskUserQuestion entirely and report "Nothing new to report about your working style this session."

**After the user selects**, save only the selected observations to the appropriate memory files. Do NOT save before approval.

## 4. Project & Feedback Memory

After the user observations are handled, review the conversation for project-level memories:

**About the project (save to project memory):**
- **Project context**: Did you learn about ongoing work, goals, deadlines, or decisions not captured in code/git?
- **References**: Did the user mention external resources, dashboards, docs, or tracking systems?
- **Feedback**: Did the user correct your approach in a way that should apply to future sessions?

Memories have two scopes:
- **Project-scoped** (`~/.claude/projects/<project>/memory/`) — project context, references, project-specific patterns
- **Global** (`~/.claude/memory/`) — user preferences, feedback, working style, anything that applies across all projects

**Routing rule:** `user` and `feedback` type memories go to global memory by default. `project` and `reference` type memories go to project memory. If unsure, ask: "does this apply to just this project, or everywhere?"

For each potential memory:
1. Check if it already exists in the appropriate MEMORY.md index (project or global)
2. If new — create the memory file in the correct location and update its MEMORY.md
3. If updating — modify the existing memory file
4. If nothing new — report "No new memories to save"

Project and feedback memories can be saved automatically (they're about the project, not the person). Report what was saved.

## 5. CLAUDE.md Review

Consider whether any project-level CLAUDE.md updates are needed based on the session.

**Only add to CLAUDE.md if ALL of these are true:**
1. It's a behavioral rule, gotcha, or non-obvious convention — not a file/component description
2. Removing it would cause Claude to make mistakes in future sessions
3. It can't be derived by reading the code, configs, or existing docs
4. It's not already documented in DESIGN.md, docs/, or memory files

**Never add:**
- Component inventories or file-path catalogs (Claude can explore the codebase)
- Counts that change (number of documents, queries, types, pages)
- Information already in @-imported files (DESIGN.md, docs/data-architecture.md)
- Standard framework behavior Claude already knows

**Size check:** If project CLAUDE.md exceeds ~120 lines, review for items that can be cut before adding new ones. CLAUDE.md should stay concise — bloat causes Claude to ignore instructions.

Also check the global `~/.claude/CLAUDE.md` if session-wide learnings apply across projects — same filtering rules apply.

Update CLAUDE.md automatically if changes pass the filter above. Report what was updated, or "No CLAUDE.md updates needed."

## 6. Checklist / Plan Progress

If the session involved working through a checklist or plan (e.g., `docs/plans/*-checklist.md`), check:
- Were any items completed that should be marked `[x]`?
- Update the checklist file accordingly

Report what was checked off, or "No active checklists affected."

## 7. Backlog Capture

Review the session for potential backlog items — deferred work, rolled-back features, bugs noticed in passing, ideas discussed but not acted on, TODOs mentioned but not completed. Don't add items that are already in the backlog.

**Present via interactive multi-select using AskUserQuestion:**

Use `AskUserQuestion` with `multiSelect: true`. Each candidate backlog item becomes one option:
- **label**: Short item title (5-10 words)
- **description**: What needs to be done and why

If there are no backlog candidates, skip the AskUserQuestion entirely and report "Nothing to add to the backlog."

**After the user selects**, append only the selected items to `BACKLOG.md` in the project root, grouped under an appropriate category heading. Use `- [ ]` checkbox format.

## 8. Client Decision Log

Review the session for product, design, and technical decisions that the client should know about. If decisions were made, write a client-facing decision log.

**Skip if:** The session was pure implementation with no decisions (just coding what was already decided), or only involved internal cleanup/refactoring with no client-visible impact.

**When to write:**
- Design choices were made (layout, interaction, visual direction)
- Scope was expanded, reduced, or deferred
- Technical tradeoffs were discussed that affect the product
- Features were cut or reprioritized
- Strategy or approach changed from what was previously planned

**How to write:**

1. Create `docs/decisions/YYYY-MM-DD-<topic>.md` (create `docs/decisions/` if it doesn't exist)
2. For each decision, capture:
   - **What was decided** — the choice made
   - **Alternatives considered** — what else was on the table
   - **Why this choice** — connected to business goals, user needs, or stakeholder priorities. Reference stakeholders by name where relevant.
3. Group decisions by category: Strategy, Design, Technical, Deferred
4. Format for client readability: no jargon, no internal tool names, no framework specifics. Write as if the reader is a smart business person who cares about their product but doesn't code.
5. Include a review summary if professional reviews (CEO, eng, design) were run during the session

**Present to the user:** "I found N decisions from this session. Writing the client decision log to `docs/decisions/...`. Want to review before I save?"

## 8.5 Next-Session Handoff Prompt

For multi-session work, the next session needs a self-contained kickoff prompt — read order, open work units, pending decisions, held items, verification gaps, and just-learned workflow lessons — written to a file (not dumped in terminal where it's hard to copy).

**Trigger gate.** Only generate this section if at least one is true:
- `git worktree list` shows >1 worktree (uncommitted parallel work)
- An active plan in `docs/plans/*.md` has unchecked items the session worked on
- The session ended on an explicit pending decision ("stop here, regroup", "checkpoint before next wave")
- Held items / verification gaps were raised but not landed

If none are true, **skip this step silently** — small single-session wraps don't need a handoff doc; the Session Summary covers them.

**How to write.**

1. Create `docs/handoffs/YYYY-MM-DD-<topic>.md` (create `docs/handoffs/` if it doesn't exist). Use the same `<topic>` slug as the related plan or decision doc when one exists.
2. Lead with a `# <Topic> — pick up from YYYY-MM-DD wrap-up` H1 and a 2–3 sentence "## State at hand-off" paragraph. No fluff. What's signed off, what's mid-flight, what's pushed vs uncommitted.
3. Use the section structure below. **Skip any section that has nothing to put in it** — empty sections are noise.

**Template:**

```markdown
# <Topic> — pick up from <YYYY-MM-DD> wrap-up

## State at hand-off

<2–3 sentences: what's signed off, what's mid-flight, what's pushed vs uncommitted.>

## Read order (before any action)

1. `path/to/file` — why this matters
2. `path/to/file` — why this matters
...

## Open work units

| Lane / unit | Branch | Path / location | What's in it | Status |
|---|---|---|---|---|
| ... | ... | ... | ... | uncommitted / pushed / merged |

(Generic for worktrees, in-flight PRs, dirty branches. Drop columns that don't apply.)

## Pending decisions (in order of dependency)

1. **<Decision name>.** <One paragraph: what's at stake, the options, what depends on it.>
2. ...

## Held items (NOT backlog — surfaced for next-wave decision)

- <Item> — <why it's held, not deferred. Estimate to fix if applicable.>

## Verification gaps

- <Gap> — <blocker that prevented verification + the fix to unblock>

## Workflow expectations (lessons from this session)

- <Lesson learned this session that next-session should carry forward. Only the new ones — durable preferences belong in memory, not here.>

## Suggested first action

<One paragraph. Concrete first command, decision, or dispatch — not a phase plan.>
```

**Section rules:**
- **Read order** — paths with one-line "why" each. No multi-paragraph entries. Cap at ~6 entries.
- **Open work units** — table form. Drop columns that don't apply (e.g., no "Path" column if not using worktrees).
- **Pending decisions** — numbered, ordered by dependency (decision 2 depends on decision 1's outcome). This is the highest-value section; spend the most words here.
- **Held items** — explicitly tagged `(NOT backlog — surfaced for next-wave decision)` to short-circuit the "why isn't this in BACKLOG.md" question. Distinct from BACKLOG.md cruft: these are decisions the user chose to hold mid-stream, not deferred work.
- **Verification gaps** — pair the *blocker* with the *fix*. "Untested" alone is not enough; "untested — needs `.env.local` first via `vercel env pull`" is.
- **Workflow expectations** — only the lessons that surfaced **this session**. Skip the section entirely if nothing new. Durable preferences (always-true rules) belong in memory files / CLAUDE.md, not here.
- **Suggested first action** — single paragraph, concrete first command/decision. Not a phase plan. The next session reads this and knows what to do in their first 5 minutes.

**Skip everything that's discoverable.**
- Don't include "current branch is X" — `git status` shows that.
- Don't include file inventories — the Open work units table covers it.
- Don't re-quote CLAUDE.md rules — link the line numbers.
- Don't restate decisions already captured in `docs/decisions/...` — link them.

**Present to the user:** "Multi-session work in flight. Writing handoff prompt to `docs/handoffs/YYYY-MM-DD-<topic>.md` so next session can pick up cleanly. Open it tomorrow, paste the first paragraph or `@`-reference the file."

After writing, report the path. Don't dump the contents in terminal — the whole point is to avoid that.

## 9. Session Summary

Provide a brief summary:
- **What was done**: 2-3 bullet points of the main work completed
- **What's next**: any logical follow-up tasks or items the user mentioned wanting to do next
- **Open items**: anything that was started but not finished, or blocked

## Output Format

Present the report, then the two interactive selectors, then the summary. The flow is:

1. **Report** (text) — Git, Services, CLAUDE.md, Checklists, Project Memory, Client Decision Log, Handoff Prompt (if generated)
2. **Interactive selectors** (AskUserQuestion) — "What I Learned" + "Backlog" as two multi-select questions in a single AskUserQuestion call. Each question uses `multiSelect: true`. Combine both into one call so the user sees them together.
3. **Process selections** — save approved memories, append approved backlog items
4. **Summary** (text) — Done, Next, Open

```
## Wrap-Up Report

### Git
- Branch: ...
- Uncommitted: ... (or clean)
- Pushed: ... (or already in sync)

### Services
- Vercel: [deploying / no push needed]
- Sanity: [schema deployed / no changes]
- Supabase: [migration applied / no changes]

### CLAUDE.md
- [updated/no changes] ...

### Checklists
- [updated/no active checklists] ...

### Project Memory
- [saved/updated/none] ...

### Client Decision Log
- [written to docs/decisions/... / no decisions this session]

### Handoff Prompt
- [written to docs/handoffs/... / not needed — single-session wrap]

[AskUserQuestion with up to 2 multi-select questions:
  Q1 header:"Remember" — "What should I remember about you?" — insights as options
  Q2 header:"Backlog" — "What should go in the backlog?" — deferred items as options
  Skip either question if there are no candidates for it.]

### Summary
- Done: ...
- Next: ...
- Open: ...
```

After presenting the report, ask: "Anything else before closing?" and wait for the user to confirm before considering the session complete.
