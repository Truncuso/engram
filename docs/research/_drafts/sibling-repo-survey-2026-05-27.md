# Sibling Repository Survey: Agentic Memory Systems
**Date:** 2026-05-27  
**Scope:** 5 repos across episodic/semantic/procedural memory, consolidation, and multi-KB orchestration  
**Context:** Informing engram v2.2+ (WP13–WP17: multi-KB, skills, episodic↔semantic bridges)

---

## 1. Letta (formerly MemGPT)

**One-liner:** Production agent platform with hierarchical memory (core/recall/archival) and persistent self-editing state across sessions.

### Memory Model & Taxonomy
- **Tiers:** Core (in-context, ~2K tokens), Recall (retrieved context), Archival (full history)
- **Semantics:** Treats LLM context window as RAM; external storage as disk. No explicit episodic↔semantic separation; pragmatic context engineering
- **Lifecycle:** Memory blocks persist across sessions; agents self-edit blocks via tool calls; no detached consolidation phase

### Storage & Retrieval
- **Storage:** SQLite via Alembic migrations; configurable memory blocks (`human`, `persona`, custom labels)
- **Retrieval:** Tool-mediated (agent calls functions to insert/retrieve); no autonomous background retrieval
- **Composition:** Single-KB only; no multi-knowledge-base scoping

### Consolidation & Reflection
- **Dreaming:** None. No offline consolidation or reflection phase.
- **Update mechanism:** Real-time self-editing by agent via tool calls; no batch cleanup or merging

### Hooks & Capture Surface
- **CLI:** `letta-code` (npm) with skills and subagent primitives
- **SDKs:** Python (`letta-client`) and TypeScript (`@letta-ai/letta-client`)
- **Tools:** Pre-built `web_search`, `fetch_webpage`; custom tools configurable at agent init

### Skills & Agent Interface
- Mentions "advanced memory and continual learning" but no public skill chaining or composition primitives shown
- Agents are model-agnostic; docs recommend Opus 4.5 / GPT-5.2

### Multi-KB / Multi-Source Support
- Single KB per agent by design
- No cross-agent knowledge sharing or tenant isolation shown

### Provenance & Attribution
- Memory blocks are labeled by role (`human`, `persona`) but no `derived_from` lineage tracking
- No explicit provenance chain

### License & Maintenance
- **License:** Apache-2.0
- **Activity:** 23K+ stars, 2.5K+ forks; active development (v0.16.8 as of search date)
- **Signal:** High. Backed by investment; regular updates.

---

## 2. Cognee

**One-liner:** Open-source memory control plane combining knowledge graphs + embeddings with auto-routing retrieval, session→graph consolidation, and agentic isolation.

### Memory Model & Taxonomy
- **Tiers:** Session memory (fast cache), permanent knowledge graph, embedding index
- **Semantics:** No hard episodic/semantic boundary; entities and relationships are unified in graph
- **Lifecycle:** Session acts as scratchpad; `SessionEnd` bridges into graph; `PreCompact` preserves memory across context resets

### Storage & Retrieval
- **Storage:** Graph (Neo4j/similar) + vector store; multi-dataset scoping (e.g., `forget(dataset="main_dataset")`)
- **Retrieval:** Four core ops: `remember`, `recall`, `forget`, `improve`; auto-routing picks strategy (vector, graph traversal, or fusion)
- **Composition:** Supports "agentic user/tenant isolation" — separate KBs per tenant

### Consolidation & Reflection
- **Dreaming:** Session→graph consolidation is background; OTEL traceability enabled
- **Learning:** Agents learn from feedback; pattern reuse across agents (e.g., distilled SQL queries)
- **Update:** Graph mutations tracked; OTEL audit trails

### Hooks & Capture Surface
- **API:** Four core ops (`remember`, `recall`, `forget`, `improve`)
- **Feedback loop:** Explicit learning from agent corrections
- **Ontology:** Supports structured knowledge via "grounding" with ontologies

### Skills & Agent Interface
- Marketing emphasizes "Persistent and Learning Agents"
- Pattern reuse / distillation is built-in (SQL expert → junior analyst transfer)
- No explicit skill chaining documented

### Multi-KB / Multi-Source Support
- **Multi-tenant:** Full isolation; datasets scoped and independently forgettable
- **Deployment:** Local, Cognee Cloud, Modal, Railway, Fly.io, Render, Daytona

### Provenance & Attribution
- OTEL traceability but no explicit `derived_from` lineage in memory objects
- Audit traits mentioned but not detailed

### License & Maintenance
- **License:** Apache-2.0
- **Activity:** Raised $7.5M seed (recent); actively maintained
- **Signal:** Very high. Well-funded; growing ops surface.

---

## 3. Mem0 (Memo.ai)

**One-liner:** Scalable memory-centric architecture with dynamic extraction, consolidation, and retrieval for multi-session dialogues; graph variant available.

### Memory Model & Taxonomy
- **Tiers:** Short-term (FIFO of conversational context), Long-term (extracted facts)
- **Semantics:** Episodic (conversation fragments) → semantic (consolidated facts via consolidation pass)
- **Lifecycle:** Periodic consolidation (summarization, merging similar facts) keeps memory manageable; salient details retained

### Storage & Retrieval
- **Storage:** Hybrid (vector + optional graph for relationships between conversational elements)
- **Retrieval:** Merged short-term + long-term at each get; multi-hop question support
- **Composition:** Graph variant models relationships; single-KB evaluation (LOCOMO benchmark)

### Consolidation & Reflection
- **Dreaming:** Periodic "consolidation" reviews and summarizes older memories, merges similar info; intentional offline phase
- **Reflection:** Implicit via consolidation logic (extracting salient narrative, not every word)
- **Metrics:** 26% relative improvement over OpenAI on LLM-as-Judge; 91% lower p95 latency; >90% token savings

### Hooks & Capture Surface
- **Interface:** Not explicitly detailed in abstract; paper-level view only
- **Integration:** Integrates with LlamaIndex (documented)
- **Extraction:** Dynamic (ongoing during conversation)

### Skills & Agent Interface
- No explicit skill system shown in abstract/integration docs
- LlamaIndex integration: `Mem0Memory` instance with user/agent/run-id context

### Multi-KB / Multi-Source Support
- Not addressed in published materials; single-KB tested
- No tenant isolation or dataset scoping documented

### Provenance & Attribution
- No explicit `derived_from` or lineage tracking mentioned
- Consolidation loses granular source attribution (by design)

### License & Maintenance
- **License:** Implied proprietary (arXiv paper; no repo link in abstract)
- **Activity:** Research paper (2504.19413, published April 2025); production product (mem0.ai) active
- **Signal:** High research + commercial signal; recent publication.

---

## 4. ModelScope MemoryScope

**One-liner:** Framework adding long-term memory tiers (short-term, medium-term, long-term) to LLM chatbots; integrates with AutoGen and AgentScope.

### Memory Model & Taxonomy
- **Tiers:** Short-term (working memory), medium-term (recent events), long-term (consolidated history)
- **Semantics:** Three-tier hierarchy; no explicit episodic/semantic labels but temporal stratification
- **Lifecycle:** Decay/retention policies per tier; older facts compressed into summaries

### Storage & Retrieval
- **Storage:** Tiered (details in dedicated "Memory Storage System" section; not fully exposed in overview)
- **Retrieval:** Tier-aware queries (prefer recent/relevant tier first)
- **Composition:** Designed for multi-component/multi-agent frameworks (AutoGen, AgentScope)

### Consolidation & Reflection
- **Dreaming:** Decay policies and summary compression; offline consolidation implied
- **Update:** Tier movement as information ages
- **Specifics:** Limited detail available without consulting full docs

### Hooks & Capture Surface
- **Integration:** CLI and API access; compatible with AutoGen and AgentScope
- **Extensibility:** Not detailed in available docs

### Skills & Agent Interface
- Multi-agent ready by design
- No explicit skill chaining documented

### Multi-KB / Multi-Source Support
- Multi-agent/multi-component compatibility; scoping/isolation not detailed
- Dataset-level multi-source not explicitly addressed

### Provenance & Attribution
- No mention in overview; likely not a focus

### License & Maintenance
- **License:** Implied open-source (ModelScope ecosystem)
- **Activity:** Active in ModelScope community; maintenance signal unclear from available materials
- **Signal:** Moderate. Community-backed but less commercial momentum than Letta/Cognee/Mem0.

---

## 5. MemGPT (cpacker/MemGPT, now Letta)

**One-liner:** Foundational 2023 research + prototype introducing OS-like memory hierarchy (core/recall/archival) with recursive search/paging and context summarization.

### Memory Model & Taxonomy
- **Tiers:** Core (in-context, ~2K tokens), Recall (indexed retrieval), Archival (full history)
- **Semantics:** Pragmatic context engineering; core = RAM, archival = disk
- **Lifecycle:** Summarization + paging strategies; agents can search nested structures multi-hop

### Storage & Retrieval
- **Storage:** Flat or hierarchical document structure; recursive key-value retrieval
- **Retrieval:** Paging + recursive search; significant gains on multi-hop (context-window approaches fail)
- **Composition:** Single-agent focus; no multi-KB

### Consolidation & Reflection
- **Dreaming:** None. Context summarization is reactive (agent-driven).
- **Update:** Passive (agent reads, framework pages in/out)

### Hooks & Capture Surface
- **Interface:** Early toolkit; now subsumed by Letta CLI/SDK

### Skills & Agent Interface
- Minimal. Foundational research artifact.

### Multi-KB / Multi-Source Support
- Not addressed; research focused on single agent + KB

### Provenance & Attribution
- No provenance tracking

### License & Maintenance
- **License:** Not independently maintained; merged into Letta (Apache-2.0)
- **Activity:** Research (arXiv:2310.08560, Oct 2023); engineering merged into Letta (v0.16+)
- **Signal:** Archive. Live on in Letta.

---

## Summary: Adoption Table for engram v2.2+

| **Repo** | **Should engram adopt / adapt / reject?** | **Concrete engagement (WP/SPEC §)** |
|----------|-------------------------------------------|-----------------------------------|
| **Letta** | **Adopt hierarchical multi-tier retrieval; reject single-KB model.** Letta's core/recall/archival taxonomy is proven; engram should adapt the tier semantics for knowledge graphs (core = derived, recall = indexed entity, archival = raw fact). Reject the tool-mediated self-edit pattern; engram's detached worker model is superior for batch consolidation. | WP14 (retrieval composition), WP15 (episodic lifecycle); SPEC §5 (scoring + retrieval fusion) |
| **Cognee** | **Adopt multi-KB scoping + session→graph consolidation; adapt entity/rel model.** Cognee's tenant isolation and session-cache pattern directly inform multi-KB orchestration (WP13). Auto-routing is not unique (engram has QMD fusion already). Adapt graph mutation tracking for provenance; engram should extend with `derived_from` lineage. | WP13 (multi-KB, KbPlugin seam), WP16 (episodic↔semantic bridges); SPEC §15.2 (KB lifecycle) |
| **Mem0** | **Adopt consolidation semantics; reject research-only status.** Mem0's periodic summarization + merging strategy for long-term management is architecturally sound and fits engram's dreaming worker (WP17). Graph variant hints at relational consolidation. However, no public implementation or multi-KB support; learning from paper only. Evaluate reference code if released. | WP17 (dreaming worker, consolidation), WP15 (episodic→semantic transition); SPEC §8.3 (consolidation rules) |
| **MemoryScope** | **Reject as distinct system; Letta supersedes.** Three-tier model is less structured than Letta's (no agent-editing, no async boundary). Integration with AutoGen/AgentScope is useful reference but orthogonal to engram's MCP model. Tier-aware decay is implementable without MemoryScope's adoption. | Nil (no unique contribution). Reference: SPEC §3.6 (decay policies) |
| **MemGPT (original)** | **Reject as live system; archive noted.** Letta is the maintained vehicle. Core/recall/archival taxonomy is inherited by Letta and already captured above. Keep as historical reference for retrieval paging strategies (WP14). | Reference: SPEC §5 (retrieval), arXiv:2310.08560 |

---

## Key Gaps Relative to engram's Vision (WP13–WP17)

1. **Provenance & `derived_from` lineage:** None of the surveyed systems explicit track why a fact is in memory (derived from original, inferred, consolidated, corrected). engram's §4 provenance model is unique.

2. **Multi-KB orchestration with cross-KB bridges:** Cognee has tenant isolation; no repo shows cross-KB querying or knowledge transfer. WP13 + SPEC §15.2 are novel.

3. **Skill subsystem with chaining:** Letta hints at skills; no system shows composition or parameter binding. WP16 is a gap.

4. **Episodic ↔ semantic linkage:** Mem0 and Letta separate layers but don't formalize the transition. engram's `derived_from` + timestamp indexing (SPEC §4, §15.3) is a design advance.

5. **Detached dreaming worker:** Only Mem0 hints at offline consolidation. engram's worker model (WP17, SPEC §11.2) is architecturally cleaner than agent-driven self-edit.

---

## Concrete Recommendations

- **WP13 (multi-KB):** Adopt Cognee's multi-tenant scoping pattern; add cross-KB discovery via derived graphs.
- **WP14–WP15 (retrieval + episodic):** Adapt Letta's tier semantics for QMD + graphify queries; use Mem0's consolidation logic for long-term management.
- **WP16 (skills):** Design from first principles; no viable sibling reference.
- **WP17 (dreaming worker):** Validate Mem0's periodic consolidation + summarization against engram's design; implement as pluggable MemoryBlock in §11.2.
- **SPEC §15 (episodic↔semantic):** Formalize `derived_from` as edge kind; Cognee's OTEL traceability is a good secondary model.

---

## Sources

- [Letta GitHub](https://github.com/letta-ai/letta)
- [Letta Blog: Agent Memory](https://www.letta.com/blog/agent-memory)
- [Letta: Building Stateful LLM Agents with Memory](https://medium.com/@vishnudhat/letta-building-stateful-llm-agents-with-memory-and-reasoning-0f3e05078b97)
- [Cognee GitHub](https://github.com/topoteretes/cognee)
- [Cognee: Memory for AI Agents](https://www.cognee.ai/blog/fundamentals/how-cognee-builds-ai-memory)
- [Cognee: Self-Improving AI Memory with Graphs](https://memgraph.com/blog/from-rag-to-graphs-cognee-ai-memory)
- [Mem0 arXiv:2504.19413](https://arxiv.org/abs/2504.19413)
- [Mem0 Blog: Memory in Agents](https://mem0.ai/blog/memory-in-agents-what-why-and-how)
- [LlamaIndex Memory Modules](https://developers.llamaindex.ai/python/examples/memory/memory/)
- [ModelScope MemoryScope Overview](https://deepwiki.com/modelscope/MemoryScope)
- [MemGPT arXiv:2310.08560](https://arxiv.org/pdf/2310.08560)
- [MemGPT: Engineering Semantic Memory](https://informationmatters.org/2025/10/memgpt-engineering-semantic-memory-through-adaptive-retention-and-context-summarization/)
- [Memory for Autonomous LLM Agents (Survey)](https://arxiv.org/html/2603.07670v1)
- [MARS: Memory-Enhanced Agents with Reflection](https://arxiv.org/pdf/2503.19271)
- [CraniMem: Cranial-Inspired Gated Memory](https://arxiv.org/pdf/2603.15642)
- [EverMemOS: Self-Organizing Memory OS](https://arxiv.org/pdf/2601.02163)

