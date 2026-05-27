---
title: "engram — QMD + graphify Tool Verification"
project: engram
created: 2026-05-22
agent: research-analyst
purpose: verify @tobilu/qmd and graphifyy can carry engram's retrieval/graph plugin seams
---

# engram Plugin Verification Report: QMD + graphify

**Date:** 2026-05-22
**Purpose:** Verify that @tobilu/qmd and graphifyy can carry engram's retrieval and graph plugin seams, respectively.
**Sources:** GitHub repos (tobi/qmd, safishamsi/graphify), PyPI graphifyy page, local installation at `/home/christoph/.nvm/versions/node/v20.17.0/lib/node_modules/@tobilu/qmd/`, engram SPEC.md.

---

## Part 1: QMD

### 1.1 Identity, Version, and Health

**What it is:** @tobilu/qmd ("Query Markup Documents") is a local, on-device hybrid search engine for Markdown and code. It combines SQLite FTS5 (BM25 keyword), sqlite-vec (dense vector), and three GGUF models for query expansion, embedding, and LLM reranking — all running locally via node-llama-cpp with no outbound API calls.

**Version discrepancy — CRITICAL FLAG:** The locally installed package reports `"version": "0.9.0"`. The upstream GitHub repo and release history show the current version is **v2.5.1** (released 2026-05-20). The gap is ~7 months of development including two major versions (v1.0.0 on 2026-02-15, v2.0.0 on 2026-03-10). v0.9.0 predates the library API stabilization, the unified `QMDStore` interface, and the runtime-aware Node.js/Bun wrapper — all of which landed in v1.0.0–v2.0.0. **The local install is stale and must be upgraded before any plugin integration work begins.**

**Maintenance health:** Actively maintained at high velocity. 8 commits on 2026-05-19 alone; v2.5.0 and v2.5.1 shipped on consecutive days. 32 open issues; no open issues about library API stability. Author is Tobi Lutke (Shopify CEO). Not archived. MIT license.

**Adoption:** npm download stats blocked (HTTP 403). The project appears to be Claude Code's own recommended retrieval tool (it has a `skills/` directory and `.claude-plugin/` in the repo). Cited in the engram SPEC itself.

### 1.2 API Surface

**Library mode (in-process TS module):** Yes, fully supported. The `dist/index.js` ESM export and `dist/index.d.ts` type definitions are the primary interface:

```typescript
import { createStore } from '@tobilu/qmd'
import type { QMDStore, SearchOptions, HybridQueryResult, StoreOptions } from '@tobilu/qmd'
```

**Concrete function signatures (verified from source):**

```typescript
// Initialization
createStore(opts: StoreOptions): Promise<QMDStore>
//   opts.dbPath: string — SQLite file (auto-created if missing)
//   opts.config: { collections: { [name]: { path, pattern?, ignore? } } }
//   opts.configPath: string  (YAML file)

// Indexing
store.update(opts?: { collections?: string[], onProgress?: UpdateProgressCb }): Promise<UpdateResult>
store.embed(opts?: { force?: boolean, chunkStrategy?: 'auto'|'regex', onProgress?: EmbedProgressCb }): Promise<EmbedResult>
store.addCollection(name: string, cfg: CollectionConfig): Promise<void>

// Search
store.search(opts: SearchOptions): Promise<HybridQueryResult[]>
//   SearchOptions.query: string         — unified auto-expanding hybrid search
//   SearchOptions.rerank?: boolean      — default true; false for latency
//   SearchOptions.limit?: number        — default 10
//   SearchOptions.minScore?: number     — 0–1 threshold
//   SearchOptions.collection?: string   — scope to one collection
store.searchLex(query: string, opts?: LexSearchOptions): Promise<SearchResult[]>   // BM25 only
store.searchVector(query: string, opts?: VectorSearchOptions): Promise<SearchResult[]>  // vec only

// Retrieval
store.get(pathOrDocid: string): Promise<DocumentResult | DocumentNotFound>
store.getDocumentBody(path: string, opts?: { fromLine?: number, maxLines?: number }): Promise<string>
store.multiGet(glob: string, opts?: { maxBytes?: number }): Promise<MultiGetResult>

// Status / health
store.getStatus(): Promise<IndexStatus>
store.getIndexHealth(): Promise<IndexHealthInfo>
store.listCollections(): Promise<NamedCollection[]>
store.close(): Promise<void>
```

**CLI:** `qmd collection add`, `qmd update`, `qmd embed`, `qmd search`, `qmd vsearch`, `qmd query`, `qmd status`, `qmd doctor` (diagnostics added v2.5.0).

**MCP server:** Yes. Stdio (default) and HTTP (Streamable). Start with `qmd mcp` or `qmd mcp --http`. Four tools: `query`, `get`, `multi_get`, `status`.

### 1.3 QMD Specifics

- **SQLite FTS5 + sqlite-vec confirmed** — tables `documents_fts` (FTS5) and `vectors_vec` (sqlite-vec) verified in source. sqlite-vec loaded as a native extension via optional peer deps.
- **Hybrid + LLM rerank confirmed** — six-stage pipeline (query expansion → parallel BM25+vec → RRF fusion → top-rank bonus → LLM cross-encoder rerank → position-aware blending). All local. Three GGUF models auto-downloaded to `~/.cache/qmd/models/`: EmbeddingGemma-300M (~300MB), Qwen3-Reranker-0.6B (~640MB), QMD Query Expansion 1.7B (~1.1GB). **~2GB cold-start download.**
- **Library API stabilized at v2.0.0** (2026-03-10). The unified `search()` replaced the older split. Current v2.5.x SDK is the stable surface.
- **Node vs Bun — CRITICAL.** The locally-installed v0.9.0 is **Bun-only** (shebang calls `bun`, imports `bun:sqlite`, uses `Bun.Glob`). It will not run on Node.js v20.17.0 at all. Current v2.5.x README states Node>=22 support and a runtime-aware wrapper (added between v0.9.0 and v1.0.0), but the `package.json` `engines` block only declares Bun. **Whether `dist/index.js` (library mode) loads under Node 22 without Bun is an unverified empirical question that must be tested.**
- **Missing/corrupt index behaviour** — missing: `createStore` auto-creates the file + runs `CREATE TABLE IF NOT EXISTS`. sqlite-vec unavailable: graceful degradation, FTS5 continues, vector ops throw. Corrupt DB: SQLite throws "database disk image is malformed" to the caller; no explicit recovery logic.

### 1.4 Risk Assessment — QMD

1. **Bun vs Node runtime ambiguity** — highest-risk item for in-process library use. Test `import { createStore }` under Node 22 immediately.
2. **~2GB cold-start model download** — stalls first run; needs a volume mount in containers; `qmd embed -f` required on embedding-model switch.
3. **Pre-v2.0.0 breaking changes** — upgrading v0.9.0 → v2.5.1 requires full integration rework.
4. **sqlite-vec load failure on macOS** (needs Homebrew SQLite). Linux is supported (`sqlite-vec-linux-x64`).
5. **Abandonment risk: LOW** — active maintainer, institutional backing, commodity SQLite+sqlite-vec core. Unlike Kuzu, the storage format is portable; the plugin seam makes replacement straightforward.

**Fallback if QMD dies:** the `RetrievalPlugin` interface (index/search/rebuild) maps cleanly to sqlite-fts5-only, lancedb, or any MCP search server.

## Part 2: graphify

### 2.1 Identity, Version, and Health

**What it is:** graphifyy (PyPI; CLI `graphify`) transforms directories of code, docs, media, PDFs into a queryable knowledge graph stored as `graph.json`. Code uses local AST extraction (tree-sitter, 31 languages, no API). **Non-code files — including Markdown — require an LLM API call** to extract nodes and edges.

**Version:** v0.8.14 (PyPI, 2026-05-20). NOTE: the repo `pyproject.toml` shows `version = "0.1.14"` — an **unexplained discrepancy** between PyPI release version and repo metadata; must be investigated before a version pin.

**Maintenance health:** Active, high velocity (v0.8.1–v0.8.14 all in May 2026). 106 open issues; 51k stars; 5.5k forks. MIT. Not archived. Python 100%.

### 2.2 API Surface

**Library mode (Python):** Partial. `graphify/__init__.py` exports `extract`, `collect_files`, `build_from_json`, `cluster`, `score_all`, `to_json`, `to_html`, `to_wiki`, etc. Callable programmatically from Python.

**CRITICAL — no headless directory-build CLI.** The `__main__.py` CLI routes only `install`, `vscode`, `claude`, `hook`, `benchmark`. The primary `/graphify .` graph-build invocation runs through the AI assistant's model context (a Claude Code skill), **not a standalone CLI command.** `python -m graphify.serve graph.json` only *serves* an existing graph — it does not build one.

**MCP server tools (verified from serve.py, stdio only):**

| Tool | Input | Returns |
|------|-------|---------|
| `query_graph` | question, mode (bfs/dfs), depth (1–6), token_budget | token-budgeted subgraph text |
| `get_node` | label | node metadata |
| `get_neighbors` | label, relation_filter? | adjacent nodes + edge relation/confidence |
| `get_community` | community_id | nodes in community |
| `god_nodes` | top_n? | top-connected nodes by degree |
| `graph_stats` | — | counts + confidence breakdown |
| `shortest_path` | source, target, max_hops? | path nodes + edge relations |

**graph.json schema:** NetworkX `node_link_data` format (`directed`, `nodes[]`, `links[]` with `relation` + `confidence` EXTRACTED/INFERRED/AMBIGUOUS, `hyperedges[]`).

### 2.3 graphify Specifics

- **Markdown requires an LLM API call** — `detect.py` classifies `.md` as "documents", routed to LLM processing. There is **no local-only Markdown extractor.** For engram (Markdown-heavy store), every ingest triggers an API call unless mitigated.
- **Python-only.** No Node/TS library. TS-host integration is via subprocess only.
- **Open ID-collision bug (#952)** — AST extractor produces ID collisions on identical filenames across folders, merging distinct nodes. engram's `semantic/`, `episodic/` subdirs can share filenames — directly affects engram's ingest.

### 2.4 Risk Assessment — graphify

1. **Markdown-requires-LLM** — the most significant design mismatch with engram's Markdown-first store.
2. **No headless build CLI** — graph construction needs a custom Python-API wrapper script or an upstream contribution.
3. **Python↔TS cross-language subprocess complexity.**
4. **ID collision bug #952** — affects multi-subdir layout until fixed.
5. **Version inconsistency** (PyPI 0.8.14 vs pyproject 0.1.14) — unresolved.
6. **Rapid churn** — graph.json node/edge attributes undocumented and unversioned (treat as opaque, use MCP tools only).
7. **Abandonment risk: MODERATE** — high stars/forks but single-maintainer, large issue backlog. `graph.json` is a standard NetworkX format, so the data is portable if graphify stalls.

## Part 3: engram Plugin Seam Assessment

**QMD → RetrievalPlugin:** `index` → `store.update`; `search` → `store.search`; `rebuild` → `store.update({force}) + store.embed({force})`. All map cleanly to the stable `QMDStore` API. Seam is real, thinner than the Kuzu situation.

**graphify → GraphPlugin:** `ingestEdges` → write files + spawn Python rebuild + read graph.json; `traverse` → `query_graph`/`get_neighbors`/`shortest_path` MCP calls. Works, but `ingestEdges` is **not incremental** — full rebuild per write is expensive. For v1, rebuild on dreaming-run boundaries, not per-write.

## 12-Line Go/No-Go Summary

1. **QMD: GO (conditional).** The `QMDStore` library API is stable (v2.0.0+), covers all of engram's `RetrievalPlugin` contract, actively maintained, MIT.
2. The locally-installed v0.9.0 is stale by 7 months — upgrade to v2.5.1 before integration.
3. Single open question: does `dist/index.js` load under Node 22 without Bun? If yes → in-process library. If no → MCP stdio subprocess. Both supported.
4. Accept the ~2GB GGUF model cold-start download as a documented one-time install cost.
5. QMD's plugin seam is realistic; fallback replacement straightforward (SQLite is commodity).
6. **graphify: GO (conditional), higher risk.** Covers engram's vision; subprocess+MCP integration viable.
7. Biggest blocker: Markdown requires an LLM API call for extraction. Mitigate with a local Ollama model as graphify's backend — low cost, no network dependency.
8. No headless `graphify build <dir>` CLI — needs a Python-API wrapper script or an upstream contribution before v1.
9. ID collision bug #952 affects engram's multi-subdir store — verify fix in v0.8.14, patch node IDs with full relative path if not.
10. Version numbering discrepancy (PyPI 0.8.14 vs pyproject 0.1.14) must be explained before a version pin.
11. graphify plugin seam maps to `ingestEdges` + `traverse`; the NetworkX `graph.json` format ensures portability if graphify is replaced.
12. **Combined verdict:** QMD as in-process retrieval plugin (pending Node-22 library test); graphify as out-of-process graph plugin (pending headless-build entrypoint + Markdown-extraction strategy). Both are correct v1 choices; neither is certain until the listed blockers are resolved.
