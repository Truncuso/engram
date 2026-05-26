---
name: phase-8b-worker-domain-logic
title: 8b — Worker domain logic (distill / connect / re-weight / verify → manifest)
type: phase
phase_status: pending
wp: wp08-dreaming-worker-orchestrator
goal: The worker reads staging, distills typed memories (derived_from backlinks), connects/detects contradictions (no contradicts edge in v1), re-weights importance within rate limits, runs the counterfactual gate for procedural memories, and writes a schema-valid manifest + git branch. Episodic sources are read-only.
verify: "npm test tests/integration/dream-worker — seeded staging → manifest with derived_from; uncorroborated procedural written confidence 0.3/dormant, a 2nd corroborating episode promotes it; planted contradiction → queue_review (no edge); episodic body never modified."
---
<!-- Template: WP-folder PHASE v2 (frontmatter-first) -->

# Phase 8b: Worker domain logic

**Goal:** Worker steps §5.2: (1) **distill** staging → typed memories,
`derived_from` backlinks, episodic sources read-only (R-4); (2) **connect** —
`related_to` links, emergent entities (≥3 mentions), two-layer contradiction
detection → manifest `queue_review` **without** a `contradicts` edge (C5);
(3) **re-weight** importance (floor 0.05) within per-run rate limits; (4) **verify
& learn** — counterfactual gate: a procedural memory promotes to `active` only if
corroborated by ≥1 independent episode, else `confidence:0.3, dormant`. Output is
**schema-validated structured output** (C6) → manifest + git branch.

**Verify:** `npm test tests/integration/dream-worker` (with a structured-output
model) — seeded staging → manifest w/ `derived_from` (SC-5); uncorroborated
procedural = 0.3/dormant, 2nd episode promotes (SC-5); planted contradiction →
`queue_review`, no edge (SC-6); episodic body unchanged.

## Steps

| Step | File | State |
|------|------|-------|
| Worker entry: read staging (verify MAC S-02), checkpoint stages | `src/worker/dream.ts` | TODO |
| Distill (LlmPlugin structured output vs dream-output schema; derived_from) | `src/worker/distill.ts` | TODO |
| Connect: related_to links + emergent entities + 2-layer contradiction detect | `src/worker/connect.ts` | TODO |
| Re-weight importance (floor 0.05, per-run rate limits) | `src/worker/reweight.ts` | TODO |
| Counterfactual gate for procedural memories | `src/worker/verify-learn.ts` | TODO |
| Manifest writer (§5.3 shape, score_breakdown C4) + git branch | `src/worker/manifest.ts` | TODO |
| Prompt-injection safety: bodies as untrusted, delimiters, structured-only (S-05) | `src/worker/prompts.ts` | TODO |
| Integration tests | `tests/integration/dream-worker.test.ts` | TODO |

## Notes

All LLM work is HERE (worker), never the orchestrator. Worker output is the
artifact the orchestrator (8c) validates + classifies. Episodic immutability is a
hard rule — distillation reads episodics, only writes derivatives.
