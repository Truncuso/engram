---
name: phase-4-init-doctor-grep-recall
title: engram init + doctor + grep-recall (Milestone M0)
type: phase
phase_status: pending
wp: wp01-core-scaffold-store-schemas-applog-jobs-occ-coreservice-doctor-init
goal: "engram init scaffolds an idempotent store; engram doctor runs abbreviated checks + quarantine; recall works via filesystem grep with importance+recency scoring — the M0 remember→recall loop, no plugins."
verify: "engram init twice = idempotent; a planted broken-frontmatter file is quarantined without breaking startup; remember a fact then recall it by grep returns it with an importance+recency score."
---
<!-- Template: WP-folder PHASE v2 (frontmatter-first) -->

# Phase 4: init + doctor + grep-recall — **Milestone M0**

**Goal:** `engram init {--global|--project}` scaffolds the store idempotently
(§13.1); `engram doctor` runs the abbreviated startup checks (§9.8) and
quarantines broken frontmatter without crashing; **`recall` works via filesystem
grep** (degradation-chain tier 3, §9.1) with importance (frontmatter) + recency
(stats sidecar) scoring — relevance is absent until WP03/04. This is the **M0
"it's alive"** milestone: a complete remember→recall loop before any plugin.

**Verify:** `engram init` run twice = no-op (SC-1); planted broken-frontmatter
file → `.engram/quarantine/`, startup still succeeds (SC-15); remember → grep
recall returns the fact with an `(importance, recency)` score (M0).

## Steps

| Step | File | State |
|------|------|-------|
| `engram init` (scaffold §3.3, schema-version, idempotent) | `src/cli/commands/init.ts` | TODO |
| `engram doctor` abbreviated + quarantine | `src/core/doctor.ts`, `src/cli/commands/doctor.ts` | TODO |
| grep recall + partial importance×recency scoring | `src/core/recall/grep.ts` | TODO |
| Wire `recall` into CoreService (degraded path) | `src/core/coreservice.ts` | TODO |
| e2e M0 + init idempotency + doctor quarantine | `tests/e2e/m0-remember-recall.test.ts` | TODO |

## Notes

Full scored recall (vector relevance, per-query normalization, m_v gate) lands in
WP04; this phase deliberately ships only the grep tier so the loop works early.
SC = SPEC §12.3 success criterion.
