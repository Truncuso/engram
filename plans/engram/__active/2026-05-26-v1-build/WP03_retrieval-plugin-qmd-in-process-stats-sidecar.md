---
name: wp03-retrieval-plugin-qmd-in-process-stats-sidecar
title: Retrieval plugin (QMD in-process) + stats sidecar
type: work-package
stage: ready
severity: HIGH
created: 2026-05-26
updated: 2026-05-26
plan: 2026-05-26-v1-build
tags: [retrieval, qmd, in-process, stats-sidecar, bm25, vector, degradation]
relationships:
  - depends_on: wp01-core-scaffold-store-schemas-applog-jobs-occ-coreservice-doctor-init
  - depends_on: wp02-plugin-host-llmplugin-vercel-ai-sdk
  - blocks: wp04-scoring-engine-recall-degradation-chain
sources: [SPEC-v2.1-§2.3, SPEC-v2.1-§4.D, SPEC-v2.1-§9.1, SPEC-v2.1-§11.1, ADR-0004, Spike-1]
---
<!-- Template: WP v2 (frontmatter-first) -->

# WP03: Retrieval plugin (QMD in-process) + stats sidecar

## Problem

The recall path (SPEC §4.D, §9.1) requires a `RetrievalPlugin` that provides
hybrid search (BM25 + vector), BM25-only fallback, and a full rebuild path, all
behind the `PluginLifecycle` contract established in WP02. QMD
(`@tobilu/qmd@2.5.2`) runs in-process as a TS ESM library — Spike 1 confirmed
all required method shapes exist and the BM25/vector split directly supports the
four-tier degradation chain. The retrieval plugin must also wire the stats sidecar
(`.engram/stats.db`) to record async recency/access touches after scoring — these
touches must never block the recall response and must not go to git or AppLog
(SPEC §4.D, §7.2). Note: first indexing triggers a one-time ~1.28 GB GGUF
embedding-model download to `~/.cache/qmd/models`; this must be documented in
setup instructions but is not a code failure.

---

## Target Files

- `src/plugins/retrieval/index.ts` — `QmdRetrievalPlugin` class implementing `RetrievalPlugin` (from WP02 `PluginLifecycle` + `RetrievalPlugin` interface); `init`, `health`, `shutdown`; wraps QMD `createStore()`
- `src/plugins/retrieval/qmd-adapter.ts` — Method mapping: `index(ref)` → `store.update(...)`, `update(ref)` → `store.update(...)`, `deindex(id)` → `store.deactivate(id)` (soft-delete), `search(query, opts)` → `store.search(...)` (hybrid), `searchLex(query, opts)` → `store.searchLex(...)` (BM25 only), `rebuild(storePath)` → `Maintenance.reindex()`; collections for scope separation via `addCollection`/`listCollections`; `getIndexHealth()` → `health()` response

---

## Verified Evidence

- `Spike-1:confirmed` — `@tobilu/qmd@2.5.2` ESM library mode live: `createStore({dbPath})` works; `search` (hybrid), `searchLex` (BM25), `searchVector`, `update`, `embed`, `deactivate` (soft-delete), `Maintenance` (reindex), `getIndexHealth`, `addCollection`/`removeCollection`/`listCollections` all confirmed
- `Spike-1:confirmed` — `searchLex` requires no embedding model; safe for the grep-adjacent degradation tier
- `Spike-1:confirmed` — BM25/vector split maps directly to SPEC §9.1 degradation tiers 1 and 2
- `Spike-1:confirmed` — ~1.28 GB GGUF embedding model downloads to `~/.cache/qmd/models` on first index; BM25 tier has no such dependency
- `ADR-0004:accepted` — QMD runs in-process; no subprocess fallback; "files are truth, index is rebuildable derived" semantics
- `SPEC-§2.3:specified` — `index`/`update`/`deindex` failures are non-fatal; core logs and enqueues retry; `rebuild()` always recovers correctness
- `SPEC-§4.D:specified` — recency/access touch goes to `stats.db` async after scoring; not git, not AppLog
- `SPEC-§9.1:specified` — four degradation tiers: (1) hybrid 300 ms, (2) BM25 150 ms, (3) filesystem grep 2000 ms, (4) partial + `degraded:true`; response carries `degraded?: {reason, tier_used}`

---

## Implementation Steps

| Step | File | State |
|------|------|-------|
| 1. Implement `QmdRetrievalPlugin` class: `init` calls `createStore({dbPath: storeRoot + "/.engram/qmd.db"})`; `health()` calls `getIndexHealth()`; `shutdown()` closes store | `src/plugins/retrieval/index.ts` | TODO |
| 2. Map `index(ref)` → `store.update({id, content: ref.frontmatter.summary + body, metadata: ref.frontmatter})`; map `update(ref)` the same way (QMD `update` is idempotent) | `src/plugins/retrieval/qmd-adapter.ts` | TODO |
| 3. Map `deindex(id)` → `store.deactivate(id)` (soft-delete; record survives for rebuild) | `src/plugins/retrieval/qmd-adapter.ts` | TODO |
| 4. Map `search(query, opts)` → `store.search(query, {collection: opts.scopeFilter, limit: opts.limit})` returning `ScoredHit[]` with relevance score from QMD | `src/plugins/retrieval/qmd-adapter.ts` | TODO |
| 5. Expose `searchLex(query, opts)` → `store.searchLex(...)` for degradation tier 2; expose `searchVector(query, opts)` for tier 1 explicit call | `src/plugins/retrieval/qmd-adapter.ts` | TODO |
| 6. Map `rebuild(storePath)` → `Maintenance.reindex()` over `memories/`; non-blocking (returns jobId or fire-and-forget) | `src/plugins/retrieval/qmd-adapter.ts` | TODO |
| 7. Wire collection creation for scope: on `init`, ensure collection per scope exists via `addCollection`; pass collection filter in `search`/`searchLex` opts | `src/plugins/retrieval/qmd-adapter.ts` | TODO |
| 8. Wire stats sidecar recency touch: after `search` returns hits, fire async `statsDb.touch(hitIds, Date.now())` with no await in the caller (SPEC §4.D); touch writes to `.engram/stats.db` only | `src/plugins/retrieval/index.ts` | TODO |
| 9. Document one-time ~1.28 GB GGUF model download in setup instructions; ensure `health()` returns `{ok: false, detail: "embedding model not yet downloaded"}` until ready (use `getIndexHealth` status field) | `src/plugins/retrieval/index.ts` | TODO |

---

## Verification

### Build Gates

| Gate | Command | Expected |
|------|---------|----------|
| TypeScript compiles | `tsc --noEmit` | 0 errors |
| Unit tests pass | `vitest run src/plugins/retrieval/` | All green |
| Lint | `eslint src/plugins/retrieval/` | 0 errors |

### Specific Tests

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| T-WP03-01 | `index()` a memory ref; `search()` with matching query | Returns at least one hit with non-zero relevance score | Integration test (QMD in-proc, temp dbPath) |
| T-WP03-02 | `searchLex()` with matching keyword query against indexed memory | Returns hit(s) without requiring embedding model (BM25 path) | Integration test |
| T-WP03-03 | `deindex(id)` for an indexed memory; then `search()` with same query | Deindexed memory not present in results | Integration test |
| T-WP03-04 | `rebuild()` after store is corrupted / wiped; then `search()` | Search returns previously indexed results after rebuild | Integration test |
| T-WP03-05 | `health()` before `init()` / when QMD store fails to open | Returns `{ok: false, detail: <reason>}` | Unit test |
| T-WP03-06 | `index()` failure (mock `store.update` to throw) | Returns without throwing; caller receives non-fatal result; error logged | Unit test |
| T-WP03-07 | Stats sidecar touch after `search()` | `stats.db` receives async recency write; `search()` caller is not blocked | Unit test with spy on statsDb.touch |
| T-WP03-08 | SC-2 precursor: index a memory with `confidence: 0.8`; recall via `search()` | Hit returned with relevance score (fused scoring in WP04; this validates the QMD output shape feeds WP04) | Integration test |

---

## Recommended Agents

| Phase | Agent | Rationale |
|-------|-------|-----------|
| Implementation | typescript-pro | QMD ESM types, async iterator handling, collection API |
| Review | typescript-reviewer | Verify `ScoredHit[]` return shape matches WP04 input contract; no `any` across adapter boundary |
| Review | code-reviewer | Non-fatal `index`/`update`/`deindex` paths; stats sidecar fire-and-forget (no leaked promise) |

---

## Review

| Reviewer | Verdict | Notes |
|-----------|---------|-------|
| — | — | — |

---

## Follow-up Issues

| Issue | Severity | Status |
|-------|----------|--------|
| ~1.28 GB GGUF download may fail in offline/CI environments; BM25-only mode should remain functional without it | MEDIUM | Open |
