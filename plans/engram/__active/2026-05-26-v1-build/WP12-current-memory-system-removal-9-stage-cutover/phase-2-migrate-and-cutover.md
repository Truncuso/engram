---
name: phase-2-migrate-and-cutover
title: Stages 3–6 — migrate content, re-point QMD, swap hook, skills cutover
type: phase
phase_status: pending
wp: wp12-current-memory-system-removal-9-stage-cutover
goal: Migrate durable content into engram (with a git-commit backup), re-point QMD collections, swap the SessionStart memory hook to engram's injection path, and cut over the skills — including rewiring grill-with-memory's memory writes to engram while preserving its ADR/glossary value.
verify: "after the hook swap, a new session gets memory context from engram (old hook removed); the 3 project facts + 2 daily logs are recallable via engram; grill-with-memory still produces ADRs/glossary and now writes via engram remember."
---
<!-- Template: WP-folder PHASE v2 (frontmatter-first) -->

# Phase 2 (stages 3–6): migrate & cutover

**Goal:** The reversible heart of the cutover.
- **Stage 3 — migrate (gated on WP05).** `git commit` `dotfiles/claude/.memory/`
  as the backup point, then migrate the 3 project facts + 2 daily logs + useful
  global facts into engram via `memory.remember` / `memory.ingest`. Rollback:
  `git checkout <hash> -- .memory/`.
- **Stage 4 — re-point QMD (gated on WP08 dreaming live).** `qmd collection
  remove ~/.claude/.memory`; engram's RetrievalPlugin owns `~/.engram/memories/`.
  QMD binary + `mcpServers.qmd` STAY.
- **Stage 5 — swap SessionStart hook.** Remove `session-start-memory.cjs` +
  `session-end-memory.cjs` entries from `hooks.json`; engram now injects context.
  **This is the moment the old system stops serving memory** — verify engram does.
- **Stage 6 — skills cutover.** Archive `memory-write/curate/init/onboard`;
  **rewire `grill-with-memory`** memory writes (the 2 `write_memory_file.py` /
  `write_session_outcome.py` calls) to `engram memory remember`; its
  grilling/ADR/glossary logic and `next_adr_number.py` stay untouched.

**Verify:** post-swap new session gets engram context; migrated content
recallable; grill-with-memory still grills + ADRs, writes via engram.

## Steps

| Step | File | State |
|------|------|-------|
| git commit `.memory/` (backup point) | `dotfiles/claude/.memory/` | TODO |
| Migrate 3 facts + 2 daily logs + global facts → engram | engram store | TODO |
| `qmd collection remove ~/.claude/.memory`; add engram memories/ | qmd | TODO |
| Remove SessionStart/End memory-hook entries | `~/.claude/hooks/hooks.json` | TODO |
| Verify engram injects context on new session | — | TODO |
| Archive 4 memory skills; rewire grill-with-memory writes | `~/.claude/skills/**` | TODO |

## Notes

Surgical edits to LIVE config — touch only the targeted entries
(`session-start-memory.cjs` shares the SessionStart array with an unrelated
`lifecycle-launcher`). Stages 3 and 5 are the two reversible checkpoints; each is
independently verifiable before proceeding.
