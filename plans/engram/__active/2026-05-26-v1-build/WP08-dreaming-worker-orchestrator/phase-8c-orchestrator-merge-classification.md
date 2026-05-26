---
name: phase-8c-orchestrator-merge-classification
title: 8c — Orchestrator merge + deterministic classification (zero LLM)
type: phase
phase_status: pending
wp: wp08-dreaming-worker-orchestrator
goal: The orchestrator validates a manifest against the dream-output schema, classifies each hunk deterministically (safe/gated, §5.5), three-way-merges field-disjoint safe hunks, queues gated/overlap hunks for review, and enforces merge validation incl. active-pool floor. Testable with hand-written manifests — zero LLM.
verify: "npm test tests/integration/orchestrator — a manifest with one auto-safe + one gated hunk: safe merges, gated → review queue; planted version regression rejected; active-pool floor blocks mass-archive even with merge_policy:always-auto (SC-14); injected-frontmatter manifest → job FAILED (SC-12)."
---
<!-- Template: WP-folder PHASE v2 (frontmatter-first) -->

# Phase 8c: Orchestrator merge + classification (zero LLM)

**Goal:** Orchestrator consumes the worker's manifest: (0) **schema-validate** the
whole manifest (C6/S-05) — fail → job FAILED; (1) **deterministic safe/gated**
predicate over the diff (§5.5) — LLM cannot self-classify (S-11); (2) **merge
OCC** against manifest `base_versions` (C12, never retries → review queue on
mismatch); (3) **three-way merge** field-disjoint safe hunks, field-overlap →
review; (4) **merge validation** (§9.4): YAML parse, version not regressed,
lifecycle rate limits, importance≥0.05, **active-pool floor** (block mass-archive
regardless of `merge_policy`), visibility invariant (S-12). All testable by
feeding **hand-written manifests** — zero LLM.

**Verify:** `npm test tests/integration/orchestrator` — mixed safe/gated manifest
classifies + routes correctly; version regression rejected; active-pool floor
blocks mass-archive under `always-auto` (SC-14); injected-frontmatter manifest →
FAILED (SC-12).

## Steps

| Step | File | State |
|------|------|-------|
| Manifest schema validation (orchestrator stage, C6) | `src/core/orchestrator/validate.ts` | TODO |
| Deterministic safe/gated predicate (§5.5 table, S-11) | `src/core/orchestrator/classify.ts` | TODO |
| Merge OCC vs base_versions (C12) + three-way field merge | `src/core/orchestrator/merge.ts` | TODO |
| Merge validation (§9.4: 6 checks + active-pool floor R-5) | `src/core/orchestrator/merge-validation.ts` | TODO |
| Visibility invariant check (S-12) | `src/core/orchestrator/visibility.ts` | TODO |
| Review queue for gated/overlap hunks | `src/core/orchestrator/review-queue.ts` | TODO |
| Commit merged hunks to main (CoreService write path) | `src/core/orchestrator/apply.ts` | TODO |
| Integration tests (hand-written manifests) | `tests/integration/orchestrator.test.ts` | TODO |

## Notes

Determinism is a security property (S-11): classification is a pure function of
the manifest diff, never the LLM. This phase needs no worker run — feed crafted
manifests. Forgetting safety rails (§9.5) enforced here at merge time.

Active-pool floor stays `min(100, 20%·total)` (OQ-09, resolved): the low floor
for small/new stores is **intended** — the floor is a mass-archive brake for
*mature* stores, not a young-store freeze (a bootstrap minimum would block early
dreaming). Document this rationale where the floor is implemented.
