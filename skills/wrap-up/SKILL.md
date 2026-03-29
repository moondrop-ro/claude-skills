# Session Wrap-Up

End-of-session checklist. Run every check below, execute the actions, and present a summary to the user before they close the session.

## 1. Git Status

Run `git status` and `git log @{upstream}..HEAD --oneline 2>/dev/null` (to check unpushed commits).

Report:
- **Uncommitted changes**: list modified/untracked files, or "clean"
- **Unpushed commits**: list commit subjects, or "in sync with remote"
- **Current branch**: confirm which branch we're on

If there are uncommitted changes, commit them automatically with a descriptive message summarizing the session's work.

After committing (or if there were already unpushed commits), **push to remote** automatically. This triggers Vercel auto-deploy. Report the push result and note that Vercel deployment is in progress.

## 2. Service Deployments

After pushing, check and report on all connected services:

- **Vercel**: Note that push triggers auto-deploy. If Vercel CLI is available, run `vercel ls --limit 1` to confirm latest deployment status.
- **Sanity**: If Sanity schema files were modified this session, remind the user to run `npx sanity@latest schema deploy` (or offer to run it). If no schema changes, report "No Sanity schema changes."
- **Supabase**: If any migration SQL files were created or modified, remind the user to apply them. If no database changes, report "No Supabase migration changes."
- **Other services**: If the project uses additional services (e.g., external APIs with keys added/changed), note any env var or configuration updates needed.

Report the status of each service, or "No updates needed" for services unaffected by the session.

## 3. Memory Review

Review the conversation for anything worth persisting to memory. Start with the user — what did you learn about *them* this session?

**About the user (save to global memory):**
- **Working style**: Did you observe how the user prefers to approach problems? (e.g., research-first, prefers incremental over big-bang, wants control at checkpoints)
- **Design philosophy**: Did the user reveal values or principles that guide their decisions? (e.g., tool-agnostic, publishable by default, AI-first)
- **Preferences**: Did the user express how they like to work, communicate, or receive help?
- **Feedback**: Did the user correct your approach in a way that should apply to future sessions? (e.g., "don't do X", "always do Y"). Also capture *confirmed* approaches — when a non-obvious choice worked and the user validated it.

**About the project (save to project memory):**
- **Project context**: Did you learn about ongoing work, goals, deadlines, or decisions not captured in code/git?
- **References**: Did the user mention external resources, dashboards, docs, or tracking systems?

Memories have two scopes:
- **Project-scoped** (`~/.claude/projects/<project>/memory/`) — project context, references, project-specific patterns
- **Global** (`~/.claude/memory/`) — user preferences, feedback, working style, anything that applies across all projects

**Routing rule:** `user` and `feedback` type memories go to global memory by default. `project` and `reference` type memories go to project memory. If unsure, ask: "does this apply to just this project, or everywhere?"

For each potential memory:
1. Check if it already exists in the appropriate MEMORY.md index (project or global)
2. If new — create the memory file in the correct location and update its MEMORY.md
3. If updating — modify the existing memory file
4. If nothing new — report "No new memories to save"

Save or update memories automatically — do not ask for confirmation.

## 4. CLAUDE.md Review

Consider whether any project-level CLAUDE.md updates are needed based on the session:
- New architectural decisions or patterns established
- New gotchas discovered
- Workflow changes
- New files/sections added to the project

Also check the global `~/.claude/CLAUDE.md` if session-wide learnings apply across projects.

Update CLAUDE.md automatically if changes are needed. Report what was updated, or "No CLAUDE.md updates needed."

## 5. Checklist / Plan Progress

If the session involved working through a checklist or plan (e.g., `docs/plans/*-checklist.md`), check:
- Were any items completed that should be marked `[x]`?
- Update the checklist file accordingly

Report what was checked off, or "No active checklists affected."

## 6. Backlog Capture

Review the session for potential backlog items — deferred work, rolled-back features, bugs noticed in passing, ideas discussed but not acted on, TODOs mentioned but not completed.

1. **Present a numbered list** of candidate items with a short description each
2. **Let the user pick** which to add (e.g., "1, 3, 5" or "all" or "none")
3. **Append approved items** to `BACKLOG.md` in the project root, grouped under an appropriate category heading. Use `- [ ]` checkbox format.
4. Don't add items that are already in the backlog.

## 7. Session Summary

Provide a brief summary:
- **What was done**: 2-3 bullet points of the main work completed
- **What's next**: any logical follow-up tasks or items the user mentioned wanting to do next
- **Open items**: anything that was started but not finished, or blocked

## Output Format

Present all findings in a single, scannable report. Use this structure:

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

### Memory
- [saved/updated/none] ...

### CLAUDE.md
- [updated/no changes] ...

### Checklists
- [updated/no active checklists] ...

### Backlog
- [proposed items or "nothing to add"]

### Summary
- Done: ...
- Next: ...
- Open: ...
```

After presenting the report, ask: "Anything else before closing?" and wait for the user to confirm before considering the session complete.
