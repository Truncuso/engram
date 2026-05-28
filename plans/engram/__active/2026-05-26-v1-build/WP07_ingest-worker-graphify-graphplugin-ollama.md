---
name: wp07-ingest-worker-graphify-graphplugin-ollama
title: Ingest worker (graphify GraphPlugin, Ollama)
type: work-package
stage: spec
severity: MEDIUM
created: 2026-05-26
updated: 2026-05-28
plan: 2026-05-26-v1-build
tags: [ingest, graphify, ollama, worker, graph-plugin, path-jail, security, hook-tests]
relationships:
  - depends_on: [[wp02-plugin-host-llmplugin-vercel-ai-sdk]]
  - depends_on: [[wp05-mcp-server-coreservice-facade-16-verbs-bearer]]
  - blocks: [[wp08-dreaming-worker-orchestrator]]
sources: [SRC-01, SRC-SPEC-v2.3-SC-32]
---
<!-- Template: WP v2 (frontmatter-first) -->

# WP07: Ingest worker (graphify GraphPlugin, Ollama)

## Problem

`memory.ingest(rawPath)` is the MCP verb that turns raw source files (PDFs,
Markdown, notes) in `<store>/raw/` into typed memories with `sources:` provenance
and LLM-built tags (SPEC §4.B, §6.2). The verb is already wired in WP05's
CoreService facade; its implementation is a no-op job stub. This WP delivers the
full pipeline: path-jail enforcement, ingest job enqueue, and the worker that
calls `GraphPlugin.extract` + `LlmPlugin.complete` to distill typed memories.

The GraphPlugin must also be implemented. Per SPEC §2.3, Spike 3, and ADR-0004,
graphify is a **rebuildable derived index** (`extract → graph.json → query`), not
a mutable graph DB. `ingestEdges` means batch `graphify update`; `removeNode`
means rebuild/filter after file deletion; `traverse` is the graphify MCP stdio
server's query tools (`query_graph`, `get_neighbors`, `shortest_path`). There is no
live edge write. The graphify Python adapter wraps these three surfaces behind the
`GraphPlugin` contract via JSON-RPC over stdio (SPEC §2.4).

**Ollama is a hard prerequisite** for this phase (SPEC §6.2, ADR-0004): graphify's
`extract --backend ollama` is the only cost-free, network-independent path for
semantic extraction. The worker refuses to start the extract step when Ollama is
unreachable (health check at job start).

Security scope: `rawPath` path-jail (S-06, SPEC §8.4) and subprocess argument
arrays never shell strings (S-16, SPEC §8.4) are both implemented here. These
are the two HIGH mitigations whose home is this WP.

**v2.3 cross-reference (SC-32):** this WP owns the hook test gate. The capture →
staging → dream → recall chain, plus the filter-drop failure-injection case
(§6.1: a blocked observation is logged, never silently dropped), are verified by
`tests/integration/ingest-worker-hooks.test.ts` (W07-HOOK-1/2 below) and rolled
into the v1.3 acceptance gate (WP23, `sc32-hooks.spec.ts`).

---

## Target Files

- `src/plugins/graph/index.ts` — `GraphPlugin` implementation: `PluginLifecycle` +
  all five contract methods; delegates to `graphify-client.ts` (CLI) and the MCP
  stdio subprocess (traverse)
- `src/plugins/graph/graphify-client.ts` — TypeScript wrapper: spawns `graphify`
  CLI and MCP stdio server; `execFile`/`spawn` with argument arrays (S-16); parses
  stdout; surfaces `PluginError` kinds
- `src/plugins/graph/py/adapter.py` — thin Python shim (optional): if the graphify
  MCP stdio server needs a wrapper to adapt its JSON-RPC line format to what the TS
  client expects; kept minimal
- `src/worker/ingest.ts` — ingest worker entry point: reads job from `jobs.db`,
  path-jails `rawPath`, calls `GraphPlugin.extract`, calls `LlmPlugin.complete`
  with structured schema, writes memory files + manifest, updates job state

---

## Verified Evidence

- Spike 3 confirmed: `graphifyy` (PyPI double-y, v0.8.18) CLI `extract --backend
  ollama --max-concurrency 1` produces `graphify-out/graph.json`; MCP stdio server
  (`python -m graphify.serve`) exposes `query_graph`, `get_neighbors`,
  `get_community`, `shortest_path`, `god_nodes`, `graph_stats` — all query-only.
  No live write tools exist. See `docs/research/spikes-2026-05-26.md` §Spike 3.
- ADR-0004 records the GraphPlugin = derived-index decision.

---

## Implementation Steps

| Step | File | State |
|------|------|-------|
| 1. Implement `GraphPlugin` class with `PluginLifecycle`; `init` spawns graphify MCP stdio subprocess; `health` pings it | `src/plugins/graph/index.ts` | TODO |
| 2. Implement `extract(rawPath)`: validate rawPath is under `<storeRoot>/raw/` (S-06); spawn `graphify extract <rawPath> --backend ollama --max-concurrency 1` as `execFile` with args array (S-16); parse `graphify-out/graph.json`; return `ExtractedStructure` | `src/plugins/graph/graphify-client.ts` | TODO |
| 3. Implement `traverse(from, opts)`: send JSON-RPC request to running graphify MCP stdio subprocess for `query_graph`/`get_neighbors`/`shortest_path` depending on `opts.mode`; parse response into `GraphResult` | `src/plugins/graph/graphify-client.ts` | TODO |
| 4. Implement `ingestEdges(refs)`: batch call `graphify update <storeRoot>/memories/` as `execFile` (AST-only, no-LLM, fast); non-fatal on failure — log + schedule retry | `src/plugins/graph/graphify-client.ts` | TODO |
| 5. Implement `removeNode(id)`: rebuild/filter `graph.json` post-delete; called from governance cascade (WP10); non-fatal | `src/plugins/graph/graphify-client.ts` | TODO |
| 6. Implement `rebuild(store)`: run `graphify extract --backend ollama` over `memories/`; full re-derive; used for recovery | `src/plugins/graph/graphify-client.ts` | TODO |
| 7. Write ingest worker entry: read job row (state SPAWNED); verify `rawPath` under `<storeRoot>/raw/` with `path.resolve()` + prefix assert; abort with `path-denied` AppLog if jail violated | `src/worker/ingest.ts` | TODO |
| 8. Worker: call `GraphPlugin.extract(rawPath)` → `ExtractedStructure`; on Ollama unavailable → job `FAILED` with `plugin-unavailable` | `src/worker/ingest.ts` | TODO |
| 9. Worker: call `LlmPlugin.complete` with structured schema (Zod) to distill `ExtractedStructure` into typed memories; set `origin: ingested`, `sources: [rawPath]`, `ingest_run_id`, LLM-rated `importance`, `tags`; validate output against `src/schemas/ingest-output.schema.json` | `src/worker/ingest.ts` | TODO |
| 10. Worker: write memory files (additive — all ingest output is auto-safe per §4.B); record manifest; update job → `MERGING`; orchestrator auto-merges; write `AppLog ingest_run` event | `src/worker/ingest.ts` | TODO |
| 11. Wire `GraphPlugin` into plugin host (WP02); register in startup sequence (order: LLM → Retrieval → Graph per §9.9) | `src/plugins/graph/index.ts` | TODO |
| 12. Write Python adapter shim if graphify MCP stdio JSON-RPC line framing needs normalisation for the TS JSON-RPC client | `src/plugins/graph/py/adapter.py` | TODO |

---

## Verification

### Build Gates

| Gate | Command | Expected |
|------|---------|----------|
| TypeScript compiles | `npm run build` | exit 0, no type errors |
| Unit tests pass | `npm test -- --testPathPattern=plugins/graph` | all green |
| Ingest worker tests pass | `npm test -- --testPathPattern=worker/ingest` | all green |
| Path-jail unit | `npm test -- --testPathPattern=ingest.*path` | traversal attempts rejected |

### Specific Tests

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| SC-3 | Drop a PDF into `raw/`, call `memory.ingest({rawPath})`, await job MERGED | Job state = MERGED; `memories/` contains ≥1 typed Markdown file with `origin: ingested`, `sources: ["raw/<file>"]`, `ingest_run_id` set, and LLM-built `tags` array | Integration test in `tests/integration/ingest.test.ts` |
| S-06 path jail | Call ingest worker with `rawPath = "<storeRoot>/../../etc/passwd"` | Worker aborts with `path-denied`; no file created; AppLog contains `path-denied` event | Unit test |
| S-06 absolute outside store | Call with absolute path outside `<storeRoot>/raw/` | Same as above | Unit test |
| S-16 argument array | Inspect all `execFile`/`spawn` calls in `graphify-client.ts` | No shell: true, no string interpolation in args; all paths passed as array elements | Code inspection + test that spawns with a space in path |
| graphify extract | Stub Ollama to succeed; call `GraphPlugin.extract(rawPath)` | Returns `ExtractedStructure` with nodes/edges; `graph.json` written under `graphify-out/` | Integration test with real graphify subprocess |
| traverse neighbors | Call `GraphPlugin.traverse([memId], {mode: "neighbors"})` after extract | Returns `GraphResult` with neighbor node IDs | Integration test with live graphify MCP stdio subprocess |
| Ollama unavailable | Start worker with Ollama not running | Job transitions to `FAILED` with error `plugin-unavailable`; no partial memory written | Integration test (mock Ollama port closed) |
| ingestEdges non-fatal | `graphify update` exits non-zero | Plugin logs WARN, job does not fail, memory write already succeeded | Unit test with stubbed subprocess |
| W07-HOOK-1 hook trace happy path | Capture hook fires on a simulated session → observation reaches `staging/` → dream worker promotes to typed memory → recall returns it | All 4 stages logged in AppLog in order; recall returns the planted memory | Integration test in `tests/integration/ingest-worker-hooks.test.ts` |
| W07-HOOK-2 filter-drop failure injection | Privacy filter blocks an observation (§6.1) | AppLog records `capture.filter.drop` with reason; observation NOT silently lost; no memory created | Integration test |

---

## Recommended Agents

| Phase | Agent | Rationale |
|-------|-------|-----------|
| Implementation | code-reviewer | Review path-jail logic and all subprocess invocations for S-06/S-16 compliance before merge |
| Implementation | tdd-guide | Worker distillation pipeline has well-defined I/O: staged input → typed memory files; strong TDD candidate |
| Security gate | security-reviewer | All subprocess argument construction must be reviewed; path traversal is a HIGH mitigation |

---

## Review

| Reviewer | Verdict | Notes |
|----------|---------|-------|
| — | — | — |

---

## Follow-up Issues

| Issue | Severity | Status |
|-------|----------|--------|
| — | — | — |
