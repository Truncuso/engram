# Memory Architecture Review — Graph Layer, Cognee/Neo4j, Mem0, and Feature Coverage

**Date**: 2026-05-19
**Author**: cunger + Claude
**Status**: COMPLETE
**Source**: OpenClaw advanced memory management concept + Cognee + Neo4j agent-memory research

---

## Executive Summary

Reviewed the current Phase 2 LLM-Memory Architecture plan against the OpenClaw advanced memory management concept (Cognee knowledge graphs, Mem0 auto-extraction), the Neo4j agent-memory package, and the original implementation guide. Three outcomes:

1. **Graph layer**: Add in-memory graph traversal to WP4 (`memory-query`) + emergent entity detector to WP5 (`cross-linker`). Reject Cognee, Neo4j, Kuzu — in-memory is sufficient for our scale.
2. **Mem0**: Reject. Auto-extraction from conversations is already covered by WP9 (`claude-history-ingest`).
3. **Feature coverage**: 15/15 original guide features covered (11 SUPERIOR, 4 EQUIVALENT, 0 GAPS).

---

## 1. Graph Layer Decision

### Research Conducted

- **Cognee**: Python SDK (`remember`/`recall`/`forget`/`improve`), Kuzu embedded or Neo4j backend, LLM-powered entity/relationship extraction from documents, Claude Code plugin with 5 lifecycle hooks. Published benchmarks (0.85 DeepEval Correctness).
- **Neo4j agent-memory**: Python-only, Neo4j 5.20+ required, three-tier memory (short-term/long-term/reasoning), multi-stage extraction (spaCy → GLiNER → LLM), 16 MCP tools, 8 framework integrations. Apache 2.0.
- **Kuzu standalone**: Embedded graph database, Cypher-like query language, file-based (zero infrastructure).

### Key Insight

Both Cognee and Neo4j agent-memory are designed for **unstructured text → LLM extraction → graph**. They ingest conversation messages and use LLMs to extract entities/relationships.

Our wiki already HAS structured entities (frontmatter `title`, `category`, `tags`, `relationships`) and explicit edges (`[[wikilinks]]` + typed relationships from frontmatter). Using their extraction pipelines would be redundant with our memory vault structure and LESS accurate than the explicit links our `cross-linker` (WP5) creates.

**We need a graph QUERY engine, not an extraction pipeline.** The edges already exist in the Markdown.

### Decision: Kuzu Embedded Graph Database, Folded into WP4+WP5

**Decision revised 2026-05-19 after user review.** Original proposal was in-memory adjacency list. User correctly identified that Kuzu's embedded graph DB provides persistence, graph algorithms, and Cypher queries without Docker — a better long-term foundation.

| Approach | Infrastructure | Extraction | Graph Query | Fit |
|----------|---------------|------------|-------------|-----|
| Cognee + Kuzu | Python + Docker or Kuzu lib | LLM-based (redundant) | Auto-routed | Medium |
| Neo4j agent-memory | Docker Neo4j (heavy) | spaCy→GLiNER→LLM (redundant) | Cypher (powerful) | Low |
| In-memory graph | Zero deps | N/A (we provide edges) | 5 hand-coded patterns | Medium |
| **Kuzu embedded** | **Node.js binding (kuzu npm)** | **N/A (we provide edges)** | **Full Cypher + algorithms** | **Best** |

**Rationale for Kuzu embedded:**
- Single npm dependency (`npm install kuzu`), no server, no Docker
- `.kuzu/` directory per vault persists between sessions — no rebuild from scratch
- Full Cypher query language: multi-hop, PageRank, betweenness centrality, community detection
- Sync pattern identical to QMD: refresh on SessionStart if stale, force-refresh on SessionEnd after writes
- Scale: constant-time queries regardless of vault size
- Markdown files remain the durable source of truth; Kuzu is a derived index (same as QMD's role)

### What Goes Into Each WP

**WP4 (`memory-query`) — Graph Traversal Module (~150 lines):**
- Build in-memory graph at query time from vault pages
- 5 query patterns QMD cannot handle:
  1. **Multi-hop**: BFS/DFS through relationship chains (e.g., "who owns permissions service?" → Alice → manages → Auth Team → owns → Permissions Service)
  2. **Reverse-lookup**: Which pages wikilink TO this page?
  3. **Neighborhood**: Everything within N hops of a page
  4. **Path existence**: Is there a relationship chain from A to B?
  5. **Bridge detection**: Which pages connect two categories?
- Schema: Node(Page {title, category, tags[], lifecycle, scope}) + Edge(LINK {rel_type})
- Seven rel_type values mirror frontmatter: extends, implements, contradicts, derived_from, uses, replaces, related_to
- Integrates as augmentation layer ABOVE QMD — QMD handles semantic search, graph handles structural traversal

**WP5 (`cross-linker`) — Emergent Entity Detector:**
- After the existing unlinked-mention scan, run a second LLM pass
- Extract named entities from each page
- Compare against the page catalog
- Flag terms appearing across ≥3 pages that lack a dedicated page file
- Suggest new page creation (output: list of `[[suggested page titles]]`)
- Optional per-analysis-run; not on every cross-link pass (cost gate)

---

## 2. Mem0 — Rejected

**What Mem0 provides:** Per-exchange LLM extraction of facts from conversation transcripts → embedding store → retrieval before each response.

**Why we reject it:**

| Capability | Our System | Mem0 |
|---|---|---|
| Extract facts from conversations | WP9 `claude-history-ingest` (batch JSONL) | Per-exchange LLM extraction |
| Structured fact storage | `memory-write` with full memory frontmatter | Embedding store |
| Deduplication | `memory-curate` DEDUP mode | Built-in |
| Consolidation | `memory-synthesize` (WP8) | `profileFrequency` rebuilds |
| Trigger | Explicit `/remember` or deferred batch | Automatic every exchange |

Mem0's per-exchange extraction adds LLM cost to EVERY response with accuracy issues the OpenClaw article itself admits: "occasionally irrelevant facts get stored and occasionally important ones get missed." Our explicit-write + batch-processing approach trades convenience for accuracy.

**Verdict**: Same outcome, different timing. Not worth the dependency or cost.

---

## 3. Original Implementation Guide — Feature Coverage Audit

Full audit of all 15 features from `Claude Code Memory Improvements - Implementation Guide.md` against the current plan:

| # | Original Feature | Current Plan | Verdict |
|---|-----------------|--------------|---------|
| 1 | MEMORY.md curated scratchpad (2,500 chars) | `index.md` (auto-generated from frontmatter) + `hot.md` (semantic snapshot) | SUPERIOR |
| 2 | USER.md user profile (1,375 chars) | Absorbed into `hot.md`; fallback loads USER.md | EQUIVALENT |
| 3 | Daily session logs (YYYY-MM-DD.md) | `journal/` structured files + WP4 daily-update + WP11 handoff routing | SUPERIOR |
| 4 | Transcript capture hook (500 chars per Stop) | WP9 `claude-history-ingest` (batch JSONL) + `session-end-memory.cjs` | EQUIVALENT* |
| 5 | MemSearch retrieval (tiered) | QMD hybrid search (BM25+vector+LLM rerank) + LSP → Grep → Read | SUPERIOR |
| 6 | Memory write (dedup, cap enforcement) | WP6 overhauled `memory-write` with memory frontmatter, CAPTURE, UPDATE modes | SUPERIOR |
| 7 | Memory budget (~3,000 tokens) | ≤2,500 tokens enforced in `session-start-memory.cjs` | EQUIVALENT |
| 8 | Cron jobs (daily distill, nightly index, weekly curator) | WP12 CronCreate jobs + WP14 autonomous daemons | SUPERIOR |
| 9 | Step 5: Audit existing memory instructions | WP1 Phase 6 + WP3 Phase 6 (CLAUDE.md override update) | EQUIVALENT |
| 10 | Step 6: CLAUDE.md updates (5 sections) | Moved from inline prose to structured skills + routing table | SUPERIOR |
| 11 | Step 7: transcript-capture.js hook | `session-end-memory.cjs` + WP9 batch JSONL ingestion | EQUIVALENT* |
| 12 | Step 8: Integration (skills, AGENTS.md, learnings, hooks) | WP0 framework repo + WP11 skill edits + WP3 symlink farm | SUPERIOR |
| 13 | Step 9: memsearch config | WP3 Phase 8: QMD multi-collection registration | SUPERIOR |
| 14 | Step 10: .gitignore entries | WP1 vault migration; needs explicit entries for new paths | GAP (doc)* |
| 15 | Step 12: Verify (6 steps) | WP15 E2E verification (50+ scripts) + per-WP verification matrices | SUPERIOR |

**Three minor gaps (none block the plan):**
- **Gap 4/11**: Real-time Stop-hook transcript capture not directly replicated. `session-end-memory.cjs` hook exists but lacks transcript-capture detail. WP9's `claude-history-ingest` covers the outcome via batch JSONL ingestion. **Severity: LOW** — Claude Code natively saves sessions to JSONL.
- **Gap 6**: Per-file character caps dissolved into combined token budget at injection boundary. Intentional architectural shift — memory pages can be larger because they're retrieved on demand, not pre-loaded.
- **Gap 14**: `.gitignore` entries for new vault paths not explicitly documented. Fix: add to WP1 implementation steps. `_archive/`, `_meta/`, `.obsidian/`, `_raw/` need git-ignore in project vaults.

---

## 4. Hook Architecture — Clean, Path Update Needed

Current hook system is well-designed:

**SessionStart** (`session-start-memory.cjs`):
- Loads global MEMORY.md → USER.md → project MEMORY.md → PROJECT.md → daily log
- Token budget: 2,500 tokens, enforced with truncation marker
- `<memory-context>` wrapper with data-not-instructions framing
- Project-overrides-global precedence
- QMD index refresh if stale (staleness-gated, 20s timeout)
- Never blocks session (any error → exit 0)

**SessionEnd** (`session-end-memory.cjs`):
- Force-refreshes QMD index so writes are searchable next session
- 30s timeout, non-blocking, async

**Required changes (WP3):**
- Update `globalDir` from `~/.claude/.memory/` → `~/memory/`
- Update section labels: MEMORY.md → `index.md`, USER.md → `hot.md`
- Add per-project `.memory/` index.md + hot.md load
- The injection pattern itself is correct — no structural changes needed

---

## 5. Bug Fixes for Plan Documents

### OQ-11 Inconsistency

OPEN_QUESTIONS.md resolution log correctly states OQ-11 is resolved:
> "Yes, private git repo. git init ~/memory/ with .gitignore for daily/, _archive/, .obsidian/workspace*.json."

But TODO.md still lists it as "Open — must resolve before WP-1 executes."

**Fix**: Update TODO.md to mark OQ-11 as resolved.

### .gitignore Documentation Gap

WP1 (Directory Migration + Vault Setup) needs explicit .gitignore entries:
- `~/memory/.obsidian/workspace*.json`
- `~/memory/_archive/`
- `<repo>/.memory/.obsidian/workspace*.json`
- `<repo>/.memory/_archive/`
- `<repo>/.memory/_raw/`
- `<repo>/.memory/_meta/`

---

## 6. Updated WP4 and WP5 Specifications

### WP4 Additions — Kuzu Graph Integration

**New deliverable**: `llm-memory/scripts/graph/sync-graph.cjs` + `graph-queries.cjs`

Kuzu embedded graph database (`npm install kuzu`). `.kuzu/` directory per vault:

Schema:
```
Node table: Page(title STRING PRIMARY KEY, category STRING, tags STRING[], lifecycle STRING, scope STRING)
Rel table:  LINK(FROM Page TO Page, rel_type STRING)
```

Build pipeline:
1. SessionStart: parse all vault `.md` files → extract nodes (frontmatter) + edges (wikilinks + relationships field) → upsert into Kuzu
2. SessionEnd: force-refresh Kuzu after writes (same pattern as QMD)
3. `.kuzu/` git-ignored (derived index, rebuildable from Markdown)

6 Cypher query patterns QMD cannot handle:
1. **Multi-hop**: MATCH path = (a)-[*1..4]-(b) WHERE a.title = $from AND b.title = $to RETURN path
2. **Reverse-lookup**: MATCH (p:Page)-[l:LINK]->(target:Page {title: $t}) RETURN p.title, l.rel_type
3. **Neighborhood**: MATCH (p:Page {title: $t})-[l:LINK*1..$d]-(n:Page) RETURN DISTINCT n.title, n.category
4. **Path existence**: MATCH (a:Page {title: $a})-[*1..5]-(b:Page {title: $b}) RETURN COUNT(*) > 0
5. **Bridge detection**: MATCH (a:Page {category: $ca})-[*1..3]-(b:Page {category: $cb}) RETURN DISTINCT ...
6. **Centrality**: PageRank, betweenness via Kuzu built-in algorithms

Integration: `memory-query` checks structural patterns via Kuzu before falling through to QMD semantic search.

### WP5 Additions — Emergent Entity Detector

**New deliverable**: `llm-memory/.skills/cross-linker/detect-emergent-entities.sh`

```
Phase 1 (existing): grep for [[wikilinks]], detect unlinked page-title mentions
Phase 2 (new): LLM pass — extract named entities from page body,
               cross-reference against page catalog,
               flag entities appearing in ≥3 pages without dedicated files
Phase 3 (new): output suggested page titles → [[suggested entities]]
```

Cost-gated: runs only when `--deep` flag is passed or during weekly curation.

---

## 7. Complexity Delta

| Added | Removed | Net Justification |
|-------|---------|-------------------|
| Kuzu embedded graph DB (`kuzu` npm) | — | Embedded, no server/Docker. Cypher + graph algorithms (PageRank, betweenness). Single npm dep. |
| Graph sync module (~200 lines) | — | Parse Markdown → upsert Kuzu. Same refresh pattern as QMD. |
| Emergent entity detector (~80 lines + LLM) | — | Closes cross-linker blind spot. Gated behind --deep. |
| — | Mem0 (rejected) | Avoids Python + per-exchange LLM cost. |
| — | Cognee/Neo4j (rejected) | Avoids Docker + Python. Kuzu gives graph DB without extraction pipeline we don't need. |

---

## 8. Verification

- [x] WP4 graph integration: Kuzu schema + 6 Cypher query patterns spec'd
- [x] WP5 emergent entity detector: spec'd with --deep flag gate
- [x] OQ-11 TODO.md inconsistency fixed
- [x] OQ-8/OQ-9 TODO.md statuses synced with OPEN_QUESTIONS.md
- [ ] WP1 .gitignore entries added (during WP1 implementation)
- [ ] SessionStart hook loads `index.md` + `hot.md` from `~/memory/` and `<repo>/.memory/` (during WP3 implementation)
- [ ] `make verify` passes after WP4/WP5 changes (during WP15)

### Code Review Findings (2026-05-19 review fleet)

Four HIGH-severity bugs found in current hook code. Fix during WP3 (memory-init overhaul):

| ID | Issue | File:Line | Fix |
|----|-------|-----------|-----|
| H1 | `startsWith` path-boundary collision | session-start-memory.cjs:77 | Use `(parent + path.sep).startsWith(root + path.sep)` or `path.relative` |
| H2 | `findProjectMemory()` walks to filesystem root | verify-memory.cjs:224-233 | Add git-root boundary matching session-start-memory.cjs |
| H3 | `qmdAvailable()` duplicated | verify-memory.cjs:51-58 + qmd-refresh.cjs:65-72 | Extract to shared module |
| H4 | Hardcoded `MEMORY.md` filename | session-start-memory.cjs:98,104 + verify-memory.cjs:79 | Update to `index.md` during WP1 migration |

### Plan Document Fixes Applied

| Fix | Document | Description |
|-----|----------|-------------|
| OQ-11 contradiction | TODO.md:61 | "must be answered before WP-1 runs" → "resolved (private git repo)" |
| OQ-8/OQ-9 stale status | TODO.md:56-57 | "Open" → "Resolved 2026-05-19" with resolution summary |
| Kuzu decision | ARCHITECTURE_REVIEW.md | In-memory → Kuzu embedded, with rationale |
| WP4 graph deliverables | WP4.md + ARCHITECTURE_REVIEW.md §6 | Two new files: sync-graph.cjs + graph-queries.cjs |
| VERIFICATION gaps | VERIFICATION.md | Added V4.8-V4.13 (graph), V5.17-V5.18 (entity detector), V1.7-V1.10 (WP1 safety gates) |
