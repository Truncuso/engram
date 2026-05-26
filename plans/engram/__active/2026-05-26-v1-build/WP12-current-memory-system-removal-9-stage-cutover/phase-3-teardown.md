---
name: phase-3-teardown
title: Stages 7–9 — remove data, scripts, CLAUDE.md block
type: phase
phase_status: pending
wp: wp12-current-memory-system-removal-9-stage-cutover
goal: "Immediately after Stage-5 cutover is verified (engram serves memory) and a verified-restorable backup exists, remove the old memory data, the 3 .cjs hook scripts (only once all skill callers are gone), and the CLAUDE.md override block — leaving the system clean and fully on engram. No 14-day soak (OQ-10)."
verify: "the Stage-3 backup was proven restorable (dry-run/checksum) before any deletion; grep confirms no remaining caller of qmd-refresh.cjs or the old collection; CLAUDE.md no longer contains the override block; engram fully serves memory on a fresh session."
---
<!-- Template: WP-folder PHASE v2 (frontmatter-first) -->

# Phase 3 (stages 7–9): teardown

**Goal:** Immediate, clean removal once Stage-5 cutover is verified — **no 14-day
soak** (OQ-10, resolved). The safety net is the **verified-restorable backup**
(Stage 3) plus an **exhaustive static-caller grep**, not a usage timer. engram's
AppLog never logged old-system reads, so it was never valid evidence; the Stage-5
hook swap is the guarantee that nothing invokes the old store.
- **Stage 7 — data removal.** Precondition: Stage-3 backup proven restorable
  (dry-run/checksum) and engram demonstrably serving on a fresh session. Export
  legacy `~/.claude/projects/*/memory/` (NOT git-tracked) and **verify the export
  is restorable** before deleting. Remove the data dir; git-commit in dotfiles.
- **Stage 8 — scripts.** Remove `session-start-memory.cjs`,
  `session-end-memory.cjs`, `qmd-refresh.cjs` — **only after** an exhaustive grep
  shows no remaining caller (`grep -r qmd-refresh ~/.claude/skills` clean;
  `capture-learning` / `handoff` rewired or gone). git-commit.
- **Stage 9 — CLAUDE.md.** Remove the `<!-- memory-init:override:start -->…end`
  block; add a short engram-referencing note. (Leave the hand-written global
  "Memory System Override" policy until engram has equivalent documented scope.)

**Verify:** Stage-3 backup proven restorable before any delete; `grep` shows no
`qmd-refresh`/old-collection caller; CLAUDE.md block gone; engram serves memory on
a fresh session.

## Steps

| Step | File | State |
|------|------|-------|
| Precondition gate: Stage-3 backup proven restorable (dry-run/checksum) + engram serves on fresh session | — | TODO |
| Export legacy projects/*/memory, **verify restorable**, then delete | `~/.claude/projects/*/memory/` | TODO |
| Remove data dir; git-commit (dotfiles) | `dotfiles/claude/.memory/` | TODO |
| Exhaustive grep-gate: no qmd-refresh / old-collection callers remain | `~/.claude/skills/**` | TODO |
| Remove 3 .cjs scripts; git-commit | `~/.claude/scripts/hooks/*.cjs` | TODO |
| Strip CLAUDE.md override block; add engram note | `~/.claude/CLAUDE.md` | TODO |

## Notes

Risk flags: with the 14-day soak dropped (OQ-10), the **verified-restorable
backup is the entire safety net** — the export/backup MUST be proven restorable
before any deletion, and the static-caller grep MUST be exhaustive (a missed
caller breaks at removal with no soak to catch it). Legacy `projects/*/memory/`
has no git rollback — export + verify first. Don't strip the global hand-written
memory policy until engram's docs cover the same scope.
