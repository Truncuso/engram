---
name: wp04-scoring-engine-recall-degradation-chain
title: Scoring engine + recall (degradation chain)
type: work-package
stage: ready
severity: HIGH
created: 2026-05-26
updated: 2026-05-26
plan: 2026-05-26-v1-build
tags: [scoring, recall, degradation, m_v, importance, recency, relevance, normalization]
relationships:
  - depends_on: wp03-retrieval-plugin-qmd-in-process-stats-sidecar
  - depends_on: wp02-plugin-host-llmplugin-vercel-ai-sdk
  - blocks: wp05-mcp-server-coreservice-facade-16-verbs-bearer
sources: [SPEC-v2.1-§3.6, SPEC-v2.1-§4.D, SPEC-v2.1-§6.3, SPEC-v2.1-§9.1, SPEC-v2.1-§12.3]
---
<!-- Template: WP v2 (frontmatter-first) -->

# WP04: Scoring engine + recall (degradation chain)

## Problem

`memory.recall` is the primary read path and must never fail hard (SPEC §9.1).
It needs a scoring engine that fuses importance, relevance (from QMD), and
recency (from stats sidecar) into `rank = (α_r·recency_norm + α_i·importance_norm
+ α_rel·relevance_norm) × m_v(origin, confidence)` with per-query min-max
normalization (SPEC §3.6). `m_v` must implement C1 (human-verified floors m_v
upward, never down), C2 (importance's double role: additive term + decay
modulator, bounded at 0.7), and C3 (`m_v = base_m_v(origin) ×
(confidence/default_confidence(origin))`, clamped to `[0.1, 1.0]`). The recall
pipeline must call `RetrievalPlugin.search` for relevance scores, fuse them in
the core (scoring never imports the retrieval plugin — SPEC §4.D), and fall through
a four-tier degradation chain (QMD hybrid 300 ms → BM25 150 ms → filesystem grep
2000 ms → partial + `degraded:true`) when tiers fail or time out. On success, the
pipeline touches stats.db for recency (async, after response). This WP closes
Milestone M1: the `remember → recall` path is end-to-end and scored.

> **Dependency note (review 2026-05-26):** the `depends_on: WP02` edge is
> **type-only** — WP04 needs the `PluginLifecycle`/`RetrievalPlugin` interface
> types that originate in WP02. The scoring engine does **not** call `LlmPlugin`
> and does **not** import the retrieval plugin (§4.D invariant): recall
> (`degradation.ts`) calls `RetrievalPlugin.search` for relevance hits; scoring
> (`engine.ts`) only fuses pre-fetched hits. Do not read this edge as a
> scoring↔LLM coupling.

---

## Target Files

- `src/core/scoring/mv.ts` — `computeMv(origin, confidence, verificationState): number`; base_m_v table; C3 formula; C1 human-verified floor (≥0.95, never lowers); clamp to [0.1, 1.0]
- `src/core/scoring/normalize.ts` — `minMaxNormalize(values: number[]): number[]`; per-query normalization across candidate set; handles degenerate case (all equal → 0.5)
- `src/core/scoring/engine.ts` — `scoreHits(hits: RawHit[], opts: ScoringOpts): ScoredHit[]`; assembles `recency_norm`, `importance_norm`, `relevance_norm`; applies α weights (default 1.0 each); multiplies by `m_v`; returns `{id, summary, score:{importance,relevance,recency,m_v,total}, body?}`; filters `lifecycle != active` unless `opts.includeDormant`
- `src/core/recall/degradation.ts` — `DegradationChain`: tries tiers in order with per-tier timeouts (300 ms / 150 ms / 2000 ms); each tier wrapped in `Promise.race` with timeout; returns `{hits, partial, degraded?: {reason, tier_used}}`; stale hits annotated `stale: true` when `file.mtime > last_reindexed`
- `src/core/recall/pipeline.ts` — `recallPipeline(query, opts, ctx)`: enforces access-control scope filter; calls `DegradationChain` to get relevance hits; passes results to `scoreHits`; fires async stats sidecar touch; returns final `{hits, partial, degraded?}` shape matching SPEC §6.3 `memory.recall` return

---

## Verified Evidence

- `SPEC-§3.6:specified` — full rank formula, α defaults, m_v definition (C1/C2/C3), clamp [0.1, 1.0], per-query min-max normalization
- `SPEC-§3.6:example` — `self-authored, confidence:0.3` → `m_v = 0.7 × (0.3/0.5) = 0.42` (C3 test case)
- `SPEC-§3.6:specified` — `human-verified` floors m_v to ≥0.95 and never lowers an origin already above 0.95 (C1)
- `SPEC-§4.D:specified` — scoring never imports retrieval plugin; access-control before search; stats touch async after; return shape `{hits:[{id,summary,score:{importance,relevance,recency,m_v,total},body?}],partial,degraded?}`
- `SPEC-§9.1:specified` — four tiers, per-tier timeouts, `degraded?: {reason, tier_used}`, stale annotation
- `SPEC-§6.3:specified` — `memory.recall` return shape; `score_breakdown` aligned to `{importance,relevance,recency,m_v}` (C4; no `access` key)
- `SPEC-§12.3:SC-2` — "Agent `remember`s a fact; later session `recall`s it, with scored ranking and a confidence-multiplier breakdown"

---

## Implementation Steps

| Step | File | State |
|------|------|-------|
| 1. Implement `computeMv`: define base_m_v table (human=1.0, ingested=0.9, agent-session=0.85, self-authored=0.7) and default_confidence table; apply C3 formula; apply C1 floor for human-verified (≥0.95, min not max); clamp result to [0.1, 1.0] | `src/core/scoring/mv.ts` | TODO |
| 2. Implement `minMaxNormalize`: per-query normalization; guard against zero-range (all equal → map to 0.5) | `src/core/scoring/normalize.ts` | TODO |
| 3. Implement `scoreHits`: read recency from stats sidecar per hit id; compute `recency_norm = exp(−λ_eff · hours_since_last_used)` where `λ_eff = λ_base · (1 − importance · 0.7)`; assemble importance_norm (from frontmatter `importance`), relevance_norm (from QMD hit score); normalize all three across candidate set; multiply by m_v; build `score:{importance,relevance,recency,m_v,total}` breakdown | `src/core/scoring/engine.ts` | TODO |
| 4. Implement `DegradationChain`: tier 1 = `RetrievalPlugin.search` (hybrid, 300 ms); tier 2 = `RetrievalPlugin.searchLex` (BM25, 150 ms) on timeout/error; tier 3 = filesystem grep over `memories/**/*.md` (2000 ms); tier 4 = return what was found + `{partial:true, degraded:{reason,tier_used}}`; annotate stale hits | `src/core/recall/degradation.ts` | TODO |
| 5. Implement `recallPipeline`: apply access-control scope filter (readable scope set from WP01 CoreService); call DegradationChain; call scoreHits; filter by lifecycle; fire async stats touch; return final shape | `src/core/recall/pipeline.ts` | TODO |
| 6. Register `recallPipeline` on CoreService as the `memory.recall` handler | `src/core/recall/pipeline.ts` | TODO |

---

## Verification

### Build Gates

| Gate | Command | Expected |
|------|---------|----------|
| TypeScript compiles | `tsc --noEmit` | 0 errors |
| Unit tests pass | `vitest run src/core/scoring/ src/core/recall/` | All green |
| Lint | `eslint src/core/scoring/ src/core/recall/` | 0 errors |

### Specific Tests

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| T-WP04-01 | SC-2: `remember` a memory with `confidence:0.8, origin:agent-session`; `recall` with matching query | Hit returned; `score.m_v` = `0.85 × (0.8/0.7) ≈ 0.971` (≤ 1.0, **no clamp applied**); `score.importance`, `score.relevance`, `score.recency`, `score.m_v`, `score.total` all present | Integration test (SPEC §12.3 SC-2) |
| T-WP04-02 | C3: compute `m_v` for `origin:self-authored, confidence:0.3` | Result = `0.7 × (0.3/0.5)` = 0.42 | Unit test (`mv.ts`) |
| T-WP04-03 | C1: `computeMv` for `human-verified` memory with low confidence | Result ≥ 0.95 (floor applied); result for `origin:human, confidence:0.90` = 1.0 (not lowered) | Unit test (`mv.ts`) |
| T-WP04-04 | C1: `computeMv` for `origin:human, verification_state:human-verified` | Result = 1.0 (base; floor does not lower) | Unit test (`mv.ts`) |
| T-WP04-05 | Clamp: `computeMv` with very high confidence on `self-authored` | Result clamped to 1.0 | Unit test (`mv.ts`) |
| T-WP04-06 | Clamp: `computeMv` with `confidence:0.01` on any origin | Result ≥ 0.1 (floor) | Unit test (`mv.ts`) |
| T-WP04-07 | `minMaxNormalize([0.2, 0.5, 0.8])` | Returns `[0.0, 0.5, 1.0]` | Unit test |
| T-WP04-08 | `minMaxNormalize([0.5, 0.5, 0.5])` | Returns `[0.5, 0.5, 0.5]` (degenerate case) | Unit test |
| T-WP04-09 | Degradation tier 1 times out (mock QMD hybrid to exceed 300 ms) | Falls to BM25 tier; response carries `degraded:{reason:"hybrid-timeout", tier_used:2}` | Unit test with mock |
| T-WP04-10 | Degradation tier 1+2 both unavailable (mock QMD to throw) | Falls to filesystem grep; response carries `degraded:{reason:..., tier_used:3}` | Unit test with mock |
| T-WP04-11 | Degradation all tiers exhausted (mock grep to time out) | Returns `{hits:[], partial:true, degraded:{reason:..., tier_used:4}}` — never throws | Unit test with mock |
| T-WP04-12 | Stats sidecar touch fires after recall returns | `statsDb.touch` called with hit ids; caller not blocked (fire-and-forget) | Unit test with spy |
| T-WP04-13 | `score_breakdown` keys: verify no `access` key in returned score object (C4) | `score` object has exactly `{importance, relevance, recency, m_v, total}` | Unit test |

---

## Recommended Agents

| Phase | Agent | Rationale |
|-------|-------|-----------|
| Implementation | typescript-pro | Numeric scoring, per-query normalization, Promise.race timeout patterns |
| Review | typescript-reviewer | Verify scoring never imports retrieval plugin (import graph check); score shape matches SPEC §6.3 and C4 (no `access` key) |
| Review | code-reviewer | Degradation chain timeout correctness; async stats touch does not leak; lifecycle filter |

---

## Review

| Reviewer | Verdict | Notes |
|-----------|---------|-------|
| — | — | — |

---

## Follow-up Issues

| Issue | Severity | Status |
|-------|----------|--------|
| — | — | — |
