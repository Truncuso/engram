---
name: phase-3-teardown
title: Stages 7–9 — remove data, scripts, CLAUDE.md block
type: phase
phase_status: pending
wp: wp12-current-memory-system-removal-9-stage-cutover
goal: After a 14-day no-access confirmation, remove the old memory data, the 3 .cjs hook scripts (only once all skill callers are gone), and the CLAUDE.md override block — leaving the system clean and fully on engram.
verify: "no session in the prior 14 days touched ~/.claude/.memory (AppLog/logs); grep confirms no remaining caller of qmd-refresh.cjs; CLAUDE.md no longer contains the override block; engram fully serves memory."
---
<!-- Template: WP-folder PHASE v2 (frontmatter-first) -->

# Phase 3 (stages 7–9): teardown

**Goal:** Final, irreversible removal — only after confidence the old system is
unused.
- **Stage 7 — data removal.** Confirm no session in the past 14 days relied on
  `~/.claude/.memory/` (check engram AppLog + daemon logs). Remove the data dir;
  git-commit the removal in dotfiles. Export legacy `~/.claude/projects/*/memory/`
  (NOT git-tracked) before deleting.
- **Stage 8 — scripts.** Remove `session-start-memory.cjs`,
  `session-end-memory.cjs`, `qmd-refresh.cjs` — **only after** confirming no
  remaining skill callers (`grep -r qmd-refresh ~/.claude/skills` clean;
  `capture-learning` / `handoff` rewired or gone). git-commit.
- **Stage 9 — CLAUDE.md.** Remove the `<!-- memory-init:override:start -->…end`
  block; add a short engram-referencing note. (Leave the hand-written global
  "Memory System Override" policy until engram has equivalent documented scope.)

**Verify:** 14-day no-access confirmed; `grep` shows no `qmd-refresh` caller;
CLAUDE.md block gone; engram fully serves memory.

## Steps

| Step | File | State |
|------|------|-------|
| 14-day no-access check (AppLog + logs) | engram AppLog | TODO |
| Export legacy projects/*/memory then delete | `~/.claude/projects/*/memory/` | TODO |
| Remove data dir; git-commit (dotfiles) | `dotfiles/claude/.memory/` | TODO |
| grep-gate: no qmd-refresh callers remain | `~/.claude/skills/**` | TODO |
| Remove 3 .cjs scripts; git-commit | `~/.claude/scripts/hooks/*.cjs` | TODO |
| Strip CLAUDE.md override block; add engram note | `~/.claude/CLAUDE.md` | TODO |

## Notes

Risk flags: legacy `projects/*/memory/` has no git rollback — export first. Don't
strip the global hand-written memory policy until engram's docs cover the same
scope. This phase is gated on real evidence of non-use, not a timer alone.
