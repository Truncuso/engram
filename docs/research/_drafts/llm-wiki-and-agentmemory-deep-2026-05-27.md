# llm-wiki + agentmemory — deep dive (input for SPEC v2.2 §15)

> Drafted 2026-05-27. Survey input only — not a re-implementation guide.

## 1. llm-wiki (Pratiyush/llm-wiki) — MIT, active

### KB structure on disk
- `raw/sessions/<project>/*.md` — immutable session transcripts converted from `.jsonl`
- `wiki/sources/` — per-session pages
- `wiki/entities/` — named things (people, tools, projects)
- `wiki/concepts/` — ideas and patterns
- `wiki/syntheses/` — cross-cutting overviews
- `wiki/comparisons/` — side-by-side analyses
- `wiki/questions/` — open questions
- root: `index.md`, `overview.md`, `log.md`
- `site/` — static HTML + `.txt`/`.json` siblings, `llms.txt`, `graph.jsonld`, `sitemap.xml`, RSS

### Write path
1. Session transcripts (`.jsonl`) from Claude Code / Codex CLI / Cursor / Gemini CLI / Obsidian → `raw/sessions/`
2. `/wiki-ingest` agent converts raw → wiki pages with `[[wikilinks]]`
3. LLM-generated frontmatter with `confidence` (4-factor: token count, LLM judge, consistency checks, citation density) and `lifecycle` (5-state)
4. Privacy redaction at write time (usernames, API keys, tokens)

### Read path / retrieval
- MCP server (12 tools): `wiki_confidence(min, max)`, `wiki_lifecycle(state)`, `wiki_dashboard()`
- Full-text via `.txt`/`.json` sibling files
- Obsidian Dataview integration for metadata queries

### Memory taxonomy
- **Lifecycle:** 5-state machine (raw → synthesized → pinned, etc.)
- **Confidence tiers:** 4-factor score
- **Page types:** episodic (sources), semantic (concepts, syntheses); procedural implicit

### Hooks
- Optional `SessionStart` hook installed via `~/.claude/settings.json` to trigger re-ingest

### Skills / slash commands
- `/wiki-ingest` — convert raw transcripts → wiki pages

### Cross-document linking
- `[[wikilinks]]` for inter-page references
- Entity pages produce implicit backlinks
- No explicit `derived_from` provenance chain

### Consolidation / dreaming
- "Auto Dream" runs `MEMORY.md` consolidation
- Lifecycle machine progresses pages through refinement states
- No per-KB daily workers documented

### Session memory
- Adapters extract from `~/.claude/`, `~/.codex/`, Cursor, Obsidian, `~/.gemini/`
- Git-tracked Markdown → idempotent re-runs

### Provenance
- Page → source listing only
- No explicit `derived_from` field

### Installer
```
git clone https://github.com/Pratiyush/llm-wiki.git
./setup.sh
# claude_desktop_config.json: "command": "python3", "args": ["-m", "llmwiki.mcp"]
```

### Multi-KB
- One KB per `config.json`
- Multi-vault ingestion via `vault_paths`
- No explicit KB registry / orchestration

### Maintenance
- Active; pure Python + stdlib + `markdown`

---

## 2. agentmemory (rohitg00/agentmemory) — Apache-2.0, very active

### KB structure
- In-memory vector index (no file tree)
- KV store with 34 scopes across 4 tiers
- Durable via the iii engine backing store

### Write path
1. `PostToolUse` hook → SHA-256 dedup (5-min window) → redaction → raw store
2. LLM-driven fact extraction → structured facts → vector embed
3. Parallel BM25 + vector + knowledge-graph triple stores
4. Working (raw) → auto-compress to Episodic; manual/scheduled consolidation to Semantic/Procedural

### Read path / retrieval
- Hybrid search, RRF-fused, k=60: BM25 + vector cosine + knowledge-graph traversal
- Session-diversified: max 3 results per session
- Default 2000-token budget injected at `SessionStart`

### Memory taxonomy (4-tier + dynamics)
| Tier | Contents |
|---|---|
| Working | Raw tool-use observations |
| Episodic | Compressed session summaries |
| Semantic | Extracted facts, patterns |
| Procedural | Workflows, decision patterns |

Dynamics: Ebbinghaus decay; access-strengthening; auto-eviction; "!! resolution" contradiction handling.

### Hooks (Claude Code, 12)
`SessionStart`, `UserPromptSubmit`, `PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `PreCompact`, `SubagentStart`, `SubagentStop`, `Stop`, `SessionEnd`. (OpenCode: 22 hooks.)

### Skills (8 slash commands)
`/recall`, `/remember`, `/session-history`, `/forget`, `/recap`, `/handoff`, `/commit-context`, `/commit-history`.

### Cross-document linking
- Knowledge-graph traversal (entity-based)
- Contradiction edges between conflicting facts
- No wikilink syntax

### Consolidation / dreaming
- 4-tier consolidation: manual via `memory_consolidate` MCP tool, or auto with `AGENTMEMORY_REFLECT=true` (Stop hook)
- Slot reflection scans recent observations post-session
- Configurable decay (Ebbinghaus)

### Session memory
- Hook-driven (12 / 22 capture points)
- Durable `iii-queue` for failed embedding/compression jobs
- Privacy filter strips secrets at write time
- Manual: `/remember`

### Provenance
- Citation provenance — trace memory back to source observations
- Multi-source: Claude Code, Cursor, Codex, Gemini, OpenCode, Hermes, OpenClaw, pi
- No explicit `derived_from` field

### Installer
```
npm install -g @agentmemory/agentmemory
agentmemory                # start server :3111
agentmemory connect claude-code
# or: /plugin marketplace add rohitg00/agentmemory && /plugin install
```
MCP: `@agentmemory/mcp` shim (7 tools local, 53 via `AGENTMEMORY_URL`).

### Multi-KB
- Team namespacing (implicit multi-KB via project profile)
- P2P sync: `memory_mesh_sync`
- No explicit KB registry; one server, memories shared

### Maintenance
- 18.5k stars, 1.5k forks, 78 issues; ~21.8k LOC; 950+ tests
- Built on Anthropic-internal iii engine microkernel

---

## 3. What engram should adopt / adapt / reject

### From llm-wiki

| Feature | Verdict | Engram landing |
|---|---|---|
| `[[wikilinks]]` + entity resolution | Adopt | KbPlugin (WP13) `llm-wiki` type; bridges (WP15) include wikilink-aware edge inference |
| 5-state page lifecycle | Adapt for derived docs, not raw memories | SPEC §15 KB sub-type config — derived KB pages have a `wiki_state`; engram memories keep their existing `active/dormant/archived` |
| 4-factor confidence scoring | Adopt as opt-in per KB | Folds into existing `confidence` field — extend SPEC §3.6 with computation recipe |
| Session-to-wiki adapter pattern | Adopt | KbPlugin types in WP13 (one type per adapter) |
| Write-time redaction | Already in SPEC §6 (CaptureIntake privacy filter) | Confirm in §15 that per-KB write paths reuse it |
| Obsidian Dataview as fallback retrieval | Adopt for obsidian-vault KB type | KbPlugin (WP13) can delegate retrieval to a per-KB hook |
| No per-KB consolidation workers | Reject | WP14 fills the gap |
| No explicit `derived_from` chain | Reject | WP17 hardens episodic↔semantic linkage |

### From agentmemory

| Feature | Verdict | Engram landing |
|---|---|---|
| 4-tier model (Working / Episodic / Semantic / Procedural) | engram already has 4 types (Semantic / Episodic / Procedural / Contextual). Adopt the "Working = raw, ephemeral" framing for Contextual + add SessionEnd hard transition (already in SPEC R-2) | §15 KB sub-types map cleanly: `agent-self` KB owns Working+Episodic, project KB owns Semantic+Procedural |
| Ebbinghaus decay + access-strengthening | Already in SPEC §3.6 (per-type decay) + R-1 confidence gate | Confirm only |
| RRF-fused hybrid retrieval (BM25 + vector + graph) | **Conflict with SPEC scoring policy** — SPEC explicitly says no RRF, scoring engine owns the formula | Reject the RRF, but adopt the *idea* of opt-in graph expansion (already in SPEC); §15 makes graph expansion configurable per KB |
| Knowledge-graph contradiction edges (`!! resolution`) | Adopt | Bridges (WP15) emits `contradicts` edges; surfaces them in `engram memory why`; counterfactual gate (R-4) handles promotion |
| 12 lifecycle hooks | engram already specs 8 capture hooks + `PreCompact`. Adopt `SubagentStart/Stop` if missing. | Verify SPEC §6.1 hook list; extend if a gap exists |
| 8 slash skills | Adopt as the seed set of the WP16 skill subsystem | `engram-recall`, `engram-remember`, `engram-forget`, `engram-recap`, `engram-handoff`, `engram-session-history`, `engram-commit-context`, plus new `engram-connect-kb`, `engram-wiki-ingest`, `engram-session-rollup`, `engram-grill-with-memory` |
| Durable iii-queue for failed jobs | engram already has SQLite `jobs` table with retry/state machine | Confirm only |
| Multi-source citation provenance | Adopt | WP17 — every dreaming-product memory carries provenance |
| Team namespacing + `memory_mesh_sync` | Reject for v1.2 | Cross-instance federation is explicitly out of scope (D-3 deferral stands) |
| Per-tier consolidation rules | Adopt | WP14 — daily jobs differ per KB type (Contextual: archive at SessionEnd; Semantic: weekly review; Procedural: counterfactual gate before promotion) |
| Skill chaining | Adopt | WP16 — chain.yaml orchestrator, modular children |

### Cross-cutting amendments for SPEC v2.2 §15

| Sub-section | Content |
|---|---|
| §15.1 KB types + KbPlugin contract | manifest, write policy, frontmatter dialect, default lifecycle jobs |
| §15.2 KB registry (`kb.list/register/connect/disconnect/route`) | global vs project store; auto-discovery; per-KB QMD index + graphify graph |
| §15.3 Per-KB lifecycle workers | reuse dream-job state machine; new job kinds `kb.daily.ingest`, `kb.recall.rollup`, `kb.connect.bridge` |
| §15.4 Cross-KB bridges (derived) | bridges.json built from per-KB graphs; recall optionally expands via bridges |
| §15.5 Skill subsystem + installer | `engram` orchestrator + chain.yaml; `engram agent install` |
| §15.6 Auto-discovery | `engram doctor --discover` scans `.obsidian/`, `MEMORY.md`, `.engram/` |
| §15.7 Episodic↔semantic linkage | `derived_from` mandatory; `engram memory why <id>` walks chain |

## Sources

- https://github.com/Pratiyush/llm-wiki — README, hook/skill defs (WebFetch)
- https://github.com/rohitg00/agentmemory — README, hook list, skill list (WebFetch)
