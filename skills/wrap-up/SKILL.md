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

## 3. What I Learned About You

Before saving any memories, **present what you observed about the user this session**. This is the most important part of wrap-up — the user should see themselves reflected back and approve what gets remembered.

**How to surface observations:**

Review the full conversation and identify:
- **Working style shifts**: Did the user do something differently than before? (e.g., ran commands independently for the first time, asked deeper questions, changed their approach)
- **New preferences or corrections**: Did they say "don't do X" or "I prefer Y"? Did they approve a non-obvious approach that should be repeated?
- **Values revealed through decisions**: What did their choices tell you about what they care about? (e.g., chose architecture over speed, prioritized accessibility, asked about the system before running it)
- **Growth or learning**: Did they learn something new this session? Did their questions evolve from "what" to "why" or "how"?

**Present observations to the user as a numbered list:**

```
### What I Learned About You This Session

1. [Observation] — [what it means for future sessions]
2. [Observation] — [what it means for future sessions]
3. [Observation] — [what it means for future sessions]

Which of these should I remember? (e.g., "all", "1, 3", "none")
```

Be specific and concrete. "You're getting better at X" is vague. "You ran the Vercel env var commands in your own terminal instead of asking me to do it, and debugged the cd error yourself" is concrete.

**After the user approves**, save approved observations to the appropriate memory files. Do NOT save before approval.

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

Consider whether any project-level CLAUDE.md updates are needed based on the session:
- New architectural decisions or patterns established
- New gotchas discovered
- Workflow changes
- New files/sections added to the project

Also check the global `~/.claude/CLAUDE.md` if session-wide learnings apply across projects.

Update CLAUDE.md automatically if changes are needed. Report what was updated, or "No CLAUDE.md updates needed."

## 6. Checklist / Plan Progress

If the session involved working through a checklist or plan (e.g., `docs/plans/*-checklist.md`), check:
- Were any items completed that should be marked `[x]`?
- Update the checklist file accordingly

Report what was checked off, or "No active checklists affected."

## 7. Backlog Capture

Review the session for potential backlog items — deferred work, rolled-back features, bugs noticed in passing, ideas discussed but not acted on, TODOs mentioned but not completed.

1. **Present a numbered list** of candidate items with a short description each
2. **Let the user pick** which to add (e.g., "1, 3, 5" or "all" or "none")
3. **Append approved items** to `BACKLOG.md` in the project root, grouped under an appropriate category heading. Use `- [ ]` checkbox format.
4. Don't add items that are already in the backlog.

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

## 9. Session Summary

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

### What I Learned About You
- [numbered observations, wait for approval before saving]

### Project Memory
- [saved/updated/none] ...

### CLAUDE.md
- [updated/no changes] ...

### Checklists
- [updated/no active checklists] ...

### Backlog
- [proposed items or "nothing to add"]

### Client Decision Log
- [written to docs/decisions/... / no decisions this session]

### Summary
- Done: ...
- Next: ...
- Open: ...
```

After presenting the report, ask: "Anything else before closing?" and wait for the user to confirm before considering the session complete.
