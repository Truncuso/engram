# Open Questions — Memory System Overhaul

**Date**: 2026-05-17
**Updated**: 2026-05-17

---

## Active Questions

| ID | Question | Context | Blocks | Owner | Status |
|----|----------|---------|--------|-------|--------|
| OQ-01 | Should the SessionStart memory-load hook prompt to run `memory-init` when `.memory/` is absent in a repo? | hook ↔ memory-init interaction | Phase 5 | cunger | DEFERRED — recommendation: hook prompts user, does not auto-create |
| OQ-02 | What rate threshold counts as "skipped" `memory-curate` before wiring to a hook schedule? | memory-curate v2 | follow-up | cunger | DEFERRED until ≥ 4 weeks of usage data |
| OQ-03 | Validate `memory-write` global vs project boundary against 2-3 real-world feedback examples | memory-write testing | Phase 3 test cases | cunger | DEFERRED — user to provide examples during memory-write verification |

---

## Resolution Log

| ID | Question | Resolution | Resolved By | Date |
|----|----------|------------|-------------|------|
| OQ-R01 | Single capped MEMORY.md vs typed files? | Typed files — better search, lower friction, no race | user via AskUserQuestion | 2026-05-17 |
| OQ-R02 | memsearch + QMD vs QMD-only vs grep? | QMD-only — no new deps, fits existing tiering | user via AskUserQuestion | 2026-05-17 |
| OQ-R03 | Raw transcript capture vs defer? | Defer — security HIGH risk; agent-authored summaries instead | user via AskUserQuestion | 2026-05-17 |
| OQ-R04 | Git policy for `.memory/`? | Mixed — daily/ gitignored, rest committed | user via AskUserQuestion | 2026-05-17 |
| OQ-R05 | Skill scope: memory-init only, full set, or partial? | Full set — memory-init + memory-write + memory-curate | user via AskUserQuestion | 2026-05-17 |
| OQ-R06 | Skill authoring approach? | `/skill-creator` interactive | user via AskUserQuestion | 2026-05-17 |
| OQ-R07 | Daily-log trigger v1 scope? | Only on `/handoff` (not every Stop) | user via AskUserQuestion | 2026-05-17 |
| OQ-R08 | plan-manager first, or straight to WP-A? | plan-manager first | user via AskUserQuestion | 2026-05-17 |
| OQ-R09 | WP execution order: sequential vs parallel? | Sequential A→B→C→D→E | user via AskUserQuestion | 2026-05-17 |

---

## Escalation Protocol

If any active question becomes blocking:

1. Halt the affected WP.
2. Surface the question with:
   - WP impacted
   - Proposed resolution (Option 1 / Option 2) with one-sentence rationale per option
   - Cost of remaining ambiguous (e.g. "blocks WP-E verification step T08")
3. Resolve before proceeding past the WP boundary.
