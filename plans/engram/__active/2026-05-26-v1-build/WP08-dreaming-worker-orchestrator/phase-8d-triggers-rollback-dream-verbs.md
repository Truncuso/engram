---
name: phase-8d-triggers-rollback-dream-verbs
title: 8d — Triggers + rollback + dream.* MCP verbs
type: phase
phase_status: pending
wp: wp08-dreaming-worker-orchestrator
goal: Dreaming fires from SessionEnd, cron/nightly, and cumulative-importance threshold; the dreaming-memory config object is loaded from .dreaming/*.md; rollback reverts a merged dream via git revert; the five dream.* MCP verbs replace the WP05 not-implemented stubs with scope enforcement.
verify: "engram dream trigger runs end-to-end against seeded staging; a SessionEnd hook triggers a run; cumulative-importance threshold fires; dream.trigger from an out-of-scope agent → scope-denied; engram dream rollback reverts the merge commit."
---
<!-- Template: WP-folder PHASE v2 (frontmatter-first) -->

# Phase 8d: Triggers + rollback + dream.* verbs

**Goal:** Wire the triggers (§5.6): `SessionEnd` hook, cron/nightly,
cumulative-importance threshold (C10 — sum of `importance` over unconsumed
staging), MCP `dream.trigger`. Load the dreaming-memory config (`.dreaming/*.md`,
daemon-owned S-10; v1 rejects `mode: cross-agent` C11). Implement rollback
(`engram dream rollback <run_id>` = `git revert` of the merge commit, §9.5;
branch retention 7d/90d) and `engram lifecycle revive` (C9). Replace WP05's
`dream.{list,configure,trigger,status,result}` stubs with real implementations +
scope enforcement (`dream.trigger` caller ∈ dreaming-memory scope, else
`scope-denied`, S-13).

**Verify:** `engram dream trigger` runs E2E on seeded staging; SessionEnd hook
triggers; cumulative-importance fires; out-of-scope `dream.trigger` →
`scope-denied` (SC-12); `engram dream rollback` reverts the merge (§9.5).

## Steps

| Step | File | State |
|------|------|-------|
| Dreaming config loader (`.dreaming/*.md`, daemon-owned; reject cross-agent C11) | `src/core/dreaming/config.ts` | TODO |
| Triggers: SessionEnd, cron, cumulative-importance (C10), dream.trigger | `src/core/dreaming/triggers.ts` | TODO |
| Rollback (`git revert` merge commit) + branch retention | `src/core/dreaming/rollback.ts` | TODO |
| `engram lifecycle revive --since` (C9) | `src/cli/commands/lifecycle.ts` | TODO |
| dream.* verbs replace WP05 stubs (+ S-13 scope) | `src/mcp/verbs/dream.ts` | TODO |
| Integration/e2e tests (trigger, scope-denied, rollback) | `tests/integration/dream-triggers.test.ts` | TODO |

## Notes

This is the phase that makes dreaming user-/agent-triggerable end-to-end. The
`dream.*` verbs were registered as not-implemented in WP05 phase-3; here they get
their real bodies. After 8d, SC-5/6/7/11/12/13 are all reachable.
