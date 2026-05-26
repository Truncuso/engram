# ADR-0004: Retrieval and graph plugins are rebuildable derived indexes

- **Status:** Accepted
- **Date:** 2026-05-26
- **Related:** SPEC §2.3, §6.2, §11.1, Spike 1 (QMD), Spike 3 (graphify)

## Context

The SPEC made QMD's retrieval transport conditional ("in-process if Node-22
library mode confirmed, else MCP subprocess") and assumed the GraphPlugin could
do live, incremental edge mutation (`ingestEdges`/`removeNode`). Both assumptions
were load-bearing and unverified. Two spikes resolved them.

- **Spike 1 (QMD):** `@tobilu/qmd@2.5.2` is a real in-process ESM library —
  `createStore().search/searchLex/searchVector/update/embed/...` maps directly to
  the `RetrievalPlugin` contract, including the BM25/vector split needed for the
  recall degradation chain. No subprocess fallback is needed.
- **Spike 3 (graphify):** `graphifyy` is `extract → graph.json → query`. Its MCP
  stdio server exposes **query-only** tools; there is **no live `add_node` /
  `remove_node` / edge-write** operation. Graph construction is CLI `extract`
  (LLM, with Ollama backend) and `update` (AST, no-LLM).

## Decision

- **QMD RetrievalPlugin runs in-process** (TS library), not as a subprocess.
- **Both QMD and graphify are rebuildable derived indexes, not authoritative
  stores.** This generalizes the existing "files are truth" principle to the
  graph:
  - `GraphPlugin.ingestEdges(refs)` → batch `update`/re-extract, not live writes.
  - `GraphPlugin.removeNode(id)` → rebuild/filter after the memory file is
    deleted (governance cascade), not a live delete.
  - `traverse` → graphify MCP stdio query tools.
- **Ollama is a hard Phase-7 prerequisite** for ingest (graphify's semantic
  extraction backend), not merely a cost mitigation.

## Consequences

- Index/graph failures stay non-fatal: log, enqueue retry, the file write still
  succeeds; `rebuild()` always recovers correctness. Consistent across both
  plugins.
- Recall degradation chain (hybrid → BM25 → grep) is directly supported by QMD's
  `searchVector`/`searchLex` + filesystem fallback.
- The graph is refreshed as a batch (on dream-merge or schedule), never mutated
  per memory write — simpler and matches graphify's actual shape.

## Alternatives considered

- **QMD as MCP subprocess** — the SPEC's fallback; unnecessary given in-process
  works. Would have added a second managed subprocess and IPC latency to the
  300 ms recall budget.
- **A mutable graph DB (Neo4j/Kuzu)** — rejected earlier (Kuzu archived; extra
  infra); engram already has explicit frontmatter edges, so a derived graph index
  suffices.
