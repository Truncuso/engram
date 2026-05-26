---
name: phase-1-build-and-coexist
title: Stages 1–2 — build to WP05, install capture hooks, coexist
type: phase
phase_status: pending
wp: wp12-current-memory-system-removal-9-stage-cutover
goal: engram is built through WP05 (agent-accessible) and WP06 capture hooks are installed; engram and the old memory system run side-by-side with no conflict and nothing removed.
verify: "old session-start-memory.cjs still loads context AND engram daemon answers system.status; engram capture hooks (new events) coexist with old SessionStart/End memory hooks without collision."
---
<!-- Template: WP-folder PHASE v2 (frontmatter-first) -->

# Phase 1 (stages 1–2): build & coexist

**Goal:** Reach engram WP05 (M2, agent-accessible) and install WP06 capture
hooks. engram's store is separate (`~/.engram/`); the old system at
`~/.claude/.memory/` is untouched and still loading context. New capture hooks
use NEW event types — no collision with the old SessionStart/SessionEnd memory
hooks. **Nothing removed yet.**

**Verify:** old `session-start-memory.cjs` still loads context; engram daemon
answers `system.status`; engram capture hooks coexist.

## Steps

| Step | File | State |
|------|------|-------|
| Gate: engram WP05 verified (M2) | — | TODO |
| Gate: engram WP06 verified (capture → staging) | — | TODO |
| Confirm engram store path distinct from old `.memory/` | `~/.engram/` | TODO |
| Confirm no hook event collision | `~/.claude/hooks/hooks.json` | TODO |

## Notes

Pure coexistence checkpoint. The only risk is hook event collision — verify
engram's capture events (PostToolUse etc.) don't duplicate the old memory hooks'
behavior in a conflicting way.
