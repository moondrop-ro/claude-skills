# Changelog

## 0.2.0

- `/wrap-up` — added §8.5 next-session handoff prompt. Writes a self-contained kickoff doc to `docs/handoffs/YYYY-MM-DD-<topic>.md` for multi-session work (gated on uncommitted worktrees, active plans, pending decisions, or held items). Replaces ad-hoc terminal-dumped prompts with a markdown file the next session can `@`-reference.

## 0.1.0

- Initial release
- `/wrap-up` — end-of-session checklist with commit, deploy checks, memory review, backlog capture
- `/catch-up` — load context about recent changes across projects with configurable adapters
- `/memory-dream` — memory consolidation: merge duplicates, prune stale entries, cluster feedback
- `/upgrade` — self-update skill with version checking and changelog display
