---
title: "Agentic-Memory Survey — input for engram SPEC v2.2 §15"
date: 2026-05-27
status: research
authors: [engram-extension-planning]
inputs:
  - docs/research/_drafts/llm-wiki-and-agentmemory-deep-2026-05-27.md
  - docs/research/_drafts/sibling-repo-survey-2026-05-27.md
scope: |
  Light pass on 7 agentic-memory / persistent-memory / wiki-from-sessions repos.
  Output: a focused "adopt / adapt / reject" verdict tied to engram WPs
  WP13–WP17 and SPEC §15 sub-sections. No re-implementation guide; survey only.
tags: [engram, memory, research, sibling-repos, v2.2]
---

# Agentic-Memory Survey (input for SPEC v2.2 §15)

This survey informs the SPEC v2.2 amendment adding multi-knowledge-base orchestration, a 5th plugin seam (`KbPlugin`), per-KB lifecycle workers, cross-KB bridges, a skill subsystem with chaining, and episodic↔semantic provenance to engram.

Full per-repo notes live in the two draft files in `_drafts/`. This document is the consolidated verdict. Acknowledgements list every project studied — full attribution lives in `README.md`.

## Repos studied

| Repo | License | Why we looked |
|---|---|---|
| [Pratiyush/llm-wiki](https://github.com/Pratiyush/llm-wiki) | MIT | Session → static knowledge base; Markdown + wikilinks; closest analog to engram's `llm-wiki` KB type. |
| [nashsu/llm_wiki](https://github.com/nashsu/llm_wiki) | (see repo) | Same LLM-wiki family as Pratiyush/llm-wiki — "compile knowledge once" pattern: incremental page merge, LanceDB hybrid BM25+vector search, sigma.js link-graph viz, format-agnostic ingest (PDF/web/YouTube), Chrome-extension capture. Direct influence on engram's read-only dashboard graph view (ADR-0010) and multi-format ingest (ADR-0011/0012). User-cited; cross-referenced here for attribution. |
| [rohitg00/agentmemory](https://github.com/rohitg00/agentmemory) | Apache-2.0 | 4-tier memory + 12 hooks + 8 slash skills; closest analog to engram's hook+skill surface. |
| [letta-ai/letta](https://github.com/letta-ai/letta) (formerly MemGPT) | Apache-2.0 | Production memory hierarchy (core/recall/archival); tool-mediated self-edit. |
| [mem0ai/mem0](https://github.com/mem0ai/mem0) | Apache-2.0 | Two-phase extract-then-update memory ops; LongMemEval/LOCOMO benchmarks. |
| [topoteretes/cognee](https://github.com/topoteretes/cognee) | Apache-2.0 | Graph + vector control plane; multi-dataset scoping (closest to multi-KB). |
| [agno-agi/agno](https://github.com/agno-agi/agno) | MPL-2.0 / commercial | Multi-agent framework with shared/team memory + reasoning steps as memory. |
| [kingjulio8238/Memary](https://github.com/kingjulio8238/Memary) | MIT | Lightweight Neo4j + RAG agent memory; entity- and relation-grounded recall. |

Also retained as prior art (already cited in SPEC v2.1 inputs): MemMachine, ElephantBroker, TierMem, agentmemory's iii-engine backing.

## Memory-model patterns across the field

| Pattern | Where seen | engram position |
|---|---|---|
| 3-tier (working / episodic / semantic) | letta, agentmemory, mem0 | engram has 4 types — adopt the "working" framing for `contextual` + hard-transition at SessionEnd (already in SPEC R-2) |
| Tool-mediated self-edit of memory | letta | **Reject for engram core** — engram keeps the immutability-of-episodic invariant (R-4). Self-edit lives only on Procedural via counterfactual gate. |
| Two-phase extract-then-update | mem0 | **Adopt for dreaming worker output** — already aligned with SPEC §5 dreaming pipeline; surface as a worker-stage contract in §15.3 |
| Graph + vector unified store | cognee, Memary | engram keeps them separate by design (ADR-0004: graph is derived index, not authoritative). **Do not unify.** |
| Multi-dataset scoping with isolation | cognee | **Adopt** — informs §15.2 KB registry + §15.4 cross-KB bridges (opt-in expansion, not unified rank) |
| Shared / team memory in multi-agent | agno | **Defer to v2** — cross-agent dreaming is the existing D-3 deferral |
| Entity-grounded recall via KG | Memary, agentmemory | **Adopt** — bridges (WP15) include entity-match edges |
| Wikilinks + entity backlinks | llm-wiki | **Adopt** for `llm-wiki` and `obsidian-vault` KB types |
| RRF fusion of BM25 + vector + graph | agentmemory | **Reject** — SPEC explicitly forbids RRF (scoring engine owns the formula). Adopt only the *idea* of opt-in graph expansion. |
| 12-hook capture surface | agentmemory | engram has 8 capture hooks + `PreCompact`. **Verify gap**, add `SubagentStart/Stop` if missing. |
| Slash-skill set (~8 verbs) | agentmemory, llm-wiki | **Adopt as seed** for skill subsystem (WP16) |
| Skill chaining / composition | (mentioned in letta, not formalised) | **Adopt — be first to formalise** via chain.yaml in WP16 |
| Two-phase consolidation | mem0 | **Adopt** as dreaming worker contract |
| Detached consolidation worker | mem0 (implied), engram | engram is already the cleanest example; keep |
| Counterfactual gate for procedural promotion | engram (R-4) | Confirm via tests in WP17 |
| Multi-source citation provenance | agentmemory (partial) | **Strengthen** — engram's `derived_from` becomes mandatory in WP17 |

## Engram landing — by WP

| Source pattern | engram WP | SPEC §15 sub-section |
|---|---|---|
| `KbPlugin` type per source flavour (markdown-store / llm-wiki / obsidian-vault / raw-sources / agent-self) | WP13 | §15.1 KB types + KbPlugin contract |
| Registry + auto-discovery + per-KB QMD index + per-KB graphify graph | WP13 | §15.2 KB registry |
| Daily ingest / recall rollup / cross-bridge build, reusing dream-job state machine | WP14 | §15.3 Per-KB lifecycle workers |
| Bridges.json as derived layer over per-KB graphs (no live edge mutation) | WP15 | §15.4 Cross-KB bridges |
| `engram` orchestrator skill + `chain.yaml` + modular children + installer | WP16 | §15.5 Skill subsystem + installer |
| `engram doctor --discover` scans for `.obsidian/`, `MEMORY.md`, `.engram/`; fs watcher promotes new KB roots | WP13 | §15.6 Auto-discovery |
| Mandatory `derived_from` for dreaming products; `engram memory why <id>` walks chain; counterfactual gate regression suite | WP17 | §15.7 Episodic↔semantic linkage |

## Explicit rejections (with reason)

| Rejected pattern | Source | Reason |
|---|---|---|
| Unified graph + vector store as authoritative | cognee, Memary | violates ADR-0004 (graph is rebuildable derived index) |
| Tool-mediated mutation of episodic memory | letta | violates R-4 (episodic = ground truth, immutable during dreaming) |
| RRF fusion in retrieval scoring | agentmemory | violates SPEC §3.6 (scoring engine owns formula; QMD = relevance only) |
| Cross-agent / federated dreaming | agno | D-3 deferral stands |
| Single big in-memory KV index | agentmemory | conflicts with engram's "files are truth" invariant |

## Out of scope for this survey

- Vendor product comparisons (Letta cloud, mem0 cloud, agno cloud, etc.).
- Benchmark numbers — none of these systems share engram's success-criteria definition, so head-to-head benchmarking is not informative for v2.2 design.
- Implementation details below the contract surface (vector index choice, embedding model, etc.) — those are engram-side decisions, already locked by ADRs.

## Sources

- https://github.com/Pratiyush/llm-wiki
- https://github.com/rohitg00/agentmemory
- https://github.com/letta-ai/letta
- https://github.com/mem0ai/mem0
- https://github.com/topoteretes/cognee
- https://github.com/agno-agi/agno
- https://github.com/kingjulio8238/Memary
- `docs/research/_drafts/llm-wiki-and-agentmemory-deep-2026-05-27.md` (full notes)
- `docs/research/_drafts/sibling-repo-survey-2026-05-27.md` (full notes)
