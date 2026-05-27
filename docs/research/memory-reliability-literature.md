---
title: "engram — Memory Reliability Literature Review"
project: engram
created: 2026-05-22
agent: research-analyst
purpose: ground and stress-test engram's memory model against the literature
---

# Memory Reliability Literature Review — engram Memory Model Validation

Analyzed: arXiv 2304.03442 (Generative Agents), 2601.18642 (FadeMem), 2603.11768 (SSGM), 2603.25097 (ElephantBroker), 2603.02240 (SuperLocalMemory), 2603.07670 (Memory for Autonomous LLM Agents survey), 2604.04853 (MemMachine), 2602.17913 (TierMem/Provenance), 2603.04814 (Beyond the Context Window), ADM (engrXiv 5919), mem0.ai/research, ACAN (Frontiers in Psychology 2025), RAG reliability playbooks, OCC literature. Benchmarks: LoCoMo, LongMemEval, BEAM.

---

## Area 1 — Generative Agents Retrieval Model: The Actual Formula

**The formula (arXiv 2304.03442 §4.1):**
```
score = α_recency·recency + α_importance·importance + α_relevance·relevance
```
All α = 1 (equal weighting) in the published implementation. All three components min-max normalized to [0,1] **across the current candidate set, per query**.

- **Recency** — exponential decay: `recency = exp(−0.005 · hours_since_last_retrieved)`. Decay factor 0.995/sandbox-hour. Time unit is sandbox hours, not wall-clock.
- **Importance** — set once at creation via LLM prompt ("rate 1–10 poignancy"), divided by 10. Static unless reflection updates it. ("cleaning room" → 2; "asking crush out" → 8.)
- **Relevance** — cosine similarity of memory-text embedding vs query embedding.
- **Reflection tree (§4.2)** — separate higher-order mechanism. Triggered when summed importance of recent events > threshold (150). ~2–3×/day. Generates salient questions → retrieves → produces 5 insights with cited source memories. Stored as non-leaf memory objects, scored identically.

**Assessment of engram §3.6 — faithful in structure, but:**
1. **Recency formula not specified** — engram says "decays over time" (vague). Commit to `exp(−λ·hours)` with a per-type λ.
2. **Importance mechanism unspecified** — the paper uses an LLM 1–10 rating. engram must state: LLM-rated at ingest, re-weighted by dreaming, agent-overridable.
3. **Normalization is query-scoped, not global** — engram stores `importance` as a global value and uses it directly; values from different creation events are incommensurable. ADOPT per-query min-max normalization across the candidate set.
4. **All α=1 is a choice, not a law** — per-type or configurable weighting may suit a coding agent.
5. **Reflection tree = engram's dreaming distillation** — same concept. ADOPT a cumulative-importance accumulator on `staging/`; trigger dreaming when it crosses a threshold, not only on time/size.
6. **Reflections cite sources** — backpointers are what make them non-lossy. engram's dreaming output MUST include `relations:[{to,kind:derived_from}]` backlinks.

## Area 2 — The Four Memory Types: Distinct Lifecycles

The literature unanimously rejects a single decay function for all four types.

| Type | Temporal character | Consolidation target | Retention |
|------|--------------------|--------------------|-----------|
| **Contextual** | session-scoped; irrelevant after session | compress→episodic or discard | TTL = session; hard expiry at session close |
| **Episodic** | timestamped events; immutable ground truth | distill→semantic/procedural | moderate decay, long tail |
| **Semantic** | timeless facts; slow evolution | (is a product) | slow decay unless contradicted/superseded |
| **Procedural** | patterns from repeated episodes | (is a product) | near-permanent; reinforced by usage |

**FadeMem (2601.18642):** dual-layer with `v_i(t) = v_i(0)·exp(−λ_i·(t−τ_i)^β_i)`, `λ_i = λ_base·exp(−μ·I_i(t))` — high-importance memories decay slower. LML half-life ~11d, SML ~5d. Access reinforcement with diminishing returns (Ebbinghaus spacing effect). 45% storage reduction, 82% critical-fact retention.

**Recommendation — CORRECT/ADD:**
- Per-type `decay_rate` defaults: contextual λ≈0.05/hr (half-life ~14h), episodic λ≈0.002/hr (~21d), semantic λ≈0.0002/hr (~200d), procedural λ≈0.0001/hr (effectively permanent).
- Contextual memories auto-transition to `dormant` at SessionEnd unless dreaming promotes them.
- ADOPT access-reinforcement: on recall, boost `importance` with diminishing returns.
- CONSIDER `decay_rate` as a per-memory stored field so high-stakes procedural memories can be made near-permanent.

## Area 3 — Consolidation / Dreaming: Safety Guarantees

Five hard safety properties (SSGM 2603.11768): contradiction-blocking (`ΔM ∧ Mcore ⊧ ⊥` → block, don't overwrite); no silent deletion; bounded semantic drift O(N·ε); temporal consistency (`valid_until` on superseded facts); access control preserved under consolidation.

**ADM (engrXiv 5919):** counterfactual verification before commit — candidate semantic rules from episodic failures must be validated against synthetic scenarios before promotion. O(log n) growth, 95% retention after 500 episodes.

**MemMachine (2604.04853):** episodic = immutable ground truth — never rewrite episodes during consolidation; produce new derivatives.

**ElephantBroker (2603.25097):** two-layer contradiction detection — explicit graph edges (supersedes/contradicts) + semantic detection (high cosine sim + large confidence gap). Both needed.

**Recommended consolidation protocol:** write to a branch (engram does this ✓); emit a manifest for every change; classify additive/mutating/destructive deterministically (never LLM-self-classified); run two-layer contradiction detection before any mutation; queue contradictions for human review (don't auto-resolve); preserve `derived_from` backlinks; never delete/overwrite episodic memory; rate-limit lifecycle transitions; prevent visibility leaks (no private/contextual content verbatim in shared/semantic output).

**Recommendation — ADD to §5.2:**
- Step 1: episodic sources are read-only during a dreaming pass; distillation produces new memories with `derived_from` backlinks.
- Step 2: two-layer contradiction detection — semantic (`sim>0.85` AND `|Δimportance|>0.3`) + graph edges. Contradictions → `action: queue_review`.
- Step 4: counterfactual validation gate — before promoting a procedural memory from a failure pattern, corroborate against ≥1 held-out episode; if uncorroborated, write `confidence:0.3`, `lifecycle:dormant`, awaiting human promotion.
- §5.3: safe/gated must be a deterministic predicate over the manifest diff.

## Area 4 — Forgetting: Ebbinghaus, Decay vs Dormancy, Rate Limits

Ebbinghaus: `R = e^(−t/S)`; strength S rises with review. Important memories decay slower — `λ_i = λ_base·exp(−μ·I_i)`. Catastrophic-forgetting prevention: merge similar memories (O(log n) growth).

The failure-safety review's **5% rate limit** is reasonable but bare. Better, layered safeguard:
1. Importance floor (min 0.05) — prevents all-zeros bugs.
2. Percentage rate limit — 5% active→dormant, 2% dormant→archived per run.
3. **Absolute active-pool floor** — `min(100, 20% of total)` must remain `active` at all times, an unconditional constraint on the dreaming merge.
4. Rollback (7-day branch retention).

Decay (continuous score reduction) ≠ dormancy (excluded from default retrieval, recoverable) ≠ archival (cold storage) ≠ deletion (governance-only). Dormancy should trigger only when the score has been below threshold for a **sustained** period (≥2 dreaming runs), not a single noisy sample.

## Area 5 — Retrieval Reliability

Seven RAG failure modes: pure-vector misses rare tokens/IDs/error-codes (silent); low-k fails multi-hop; stale embeddings degrade 10–20pts invisibly; missing provenance; oversized chunks; no change detection; treating retrieval as one-time config.

Production hybrid pipeline: parallel BM25 + vector candidates → RRF fusion (k=60) → cross-encoder/LLM rerank. BM25 rescues exact tokens; vector rescues paraphrases — neither alone suffices.

**ACAN (Frontiers 2025):** the standard I×R×Relevance linear formula leaves ~17% performance on the table vs learned cross-attention weighting (5.94 vs 5.05, p<10⁻¹²). engram's v1 linear formula is a reasonable baseline; note the headroom.

**Beyond the Context Window (2603.04814):** memory-based retrieval becomes cost-competitive with full-context at ~10 turns.

The failure-safety review's fallback chain (vector+BM25 → BM25 → grep → partial) is **validated**. Additions: (a) primary mode is **always hybrid**, pure-vector is degraded-mode-only; (b) reranking is a v1.1/v2 upgrade gate (~10–20pt recall lift on entity-heavy corpora — function names, error codes); (c) BM25 is **not optional** for engram's code-heavy use case.

## Area 6 — OCC for a File-Based Store

OCC is correct when contention is low, retry is cheap, and lock overhead would dominate — exactly engram's case (dreaming writes to a branch; agent writes to main; collisions rare). 

**File-based OCC protocol:** read file + capture `version`; compute changes; write to `.tmp`; acquire a **short-lived advisory lock only during the version-check + rename** (not during compute); if `version` moved, retry; else increment version, fsync, rename, release. This is Git-style — OCC with a tiny commit-time lock window, safe on POSIX.

**Three-way field merge** beats reject-all: when a dream branch conflicts with a main write that occurred during the dream, and the two touched **disjoint fields** (dreaming → importance/lifecycle; agent → body/recency), merge automatically. Only same-field edits need resolution. The synthesis decision to move `recency`/`access_count` to a stats sidecar eliminates the most common contention source entirely.

**Recommendation — CORRECT §6.2:** specify the atomic version-check+rename under a short advisory lockfile; max 3 OCC retries with 50ms jitter then surface as a retriable error; document the three-way field-merge for dream-branch conflicts.

## Area 7 — Provenance and Confidence

**This is engram's most significant v1 gap.** ElephantBroker's four-state evidence model with a scoring multiplier:

| State | Multiplier m_v |
|-------|----------------|
| Unverified | 0.5 |
| Self-Supported (supporting fact attached) | 0.7 |
| Tool-Supported (tool-output evidence) | 0.9 |
| Supervisor-Verified (human sign-off) | 1.0 |

`confidence_score = raw_confidence × m_v` — creates a structural incentive for evidence-gathering. The survey (2603.07670) warns that "reflective memory" risks "self-reinforcing error without external validation" — directly relevant to engram's dreaming. MemMachine: retrieval-stage optimizations contribute far more to accuracy than ingestion-stage ones (+4.2% vs +0.8%) — so adding confidence to *retrieval* scoring has outsized impact.

**Is engram's choice to store-but-not-consume `confidence` in v1 a mistake? YES.** A speculative LLM-generated memory at `confidence:0.3` currently ranks equal to a human-confirmed fact at `confidence:1.0`.

**Recommendation — ADD (HIGH PRIORITY):**
- Confidence is **v1, not v2.** Add it as a multiplicative gate on the final retrieval score: `score = (α_r·recency + α_i·importance + α_rel·relevance) × confidence_multiplier(origin, confidence)`.
- Add `verification_state: {unverified | self-supported | tool-supported | human-verified}` to the schema. Origin-derived defaults: human→1.0, ingested→0.9, agent-session→0.85, self-authored→0.7.
- v1 human-confirmation: `engram memory confirm <id>` sets `human-verified` + bumps confidence.
- Dreaming self-authored memories default to `confidence:0.5`, `verification_state:unverified`.
- The scoring engine reads `confidence` from frontmatter and applies the multiplier — one multiplication, minimal cost, high reliability benefit.

## Synthesis — Cross-Cutting Findings

1. **Confidence is v1, not v2** — one multiplication + one enum field; deferring it lets speculative memories rank equal to verified facts.
2. **Dreaming trigger should add an importance-accumulation threshold**, not time/size only.
3. **One scoring formula across four types is wrong** — per-type decay constants are mandatory (contextual decays in hours, semantic in months).
4. **Episodic memories must be immutable** during dreaming — produce derivatives, never mutate sources.
5. **Two-layer contradiction detection** — semantic + graph, before any mutation.
6. **OCC is correct** — but the commit-time version-check+rename must be explicitly atomic.
7. **Three-way field merge** for dream conflicts beats reject-all.

## 12-Line Summary — Corrections engram's memory model needs

1. **Confidence is v1, not v2** — multiplicative gate `score × m_v(origin)`; self-authored=0.7, human-confirmed=1.0.
2. **Per-type decay rates mandatory** — contextual λ≈0.05/hr (hard expiry at SessionEnd), semantic λ≈0.0002/hr; FadeMem proves 45% storage cut / 82% retention.
3. **Recency formula must be explicit** — `exp(−λ_type·hours)`, with high-importance modulating λ down.
4. **Importance scoring at creation must be specified** — LLM 1–10 rating at ingest, dreaming re-weights, agent may override.
5. **Add a cumulative-importance dreaming trigger** — accumulate importance mass on `staging/`, trigger when worth consolidating.
6. **Episodic memories immutable during dreaming** — derivatives carry `derived_from` backlinks; sources never mutated.
7. **Two-layer contradiction detection** — semantic (sim>0.85 ∧ Δconf>0.3) + graph edges, before any mutation commit.
8. **Counterfactual validation gate** for procedural promotion — corroborate against a second episode, else write dormant at confidence 0.3.
9. **OCC commit must be explicitly atomic** — version-check + rename under a short advisory lock; three-way field merge for dream-vs-main.
10. **Retrieval always hybrid (BM25+vector)** as primary; pure-vector degraded-mode only; min-max normalize per-query.
11. **Absolute active-pool floor** — min(100, 20% of total) always `active`; dreaming merge blocks any violating run.
12. **Four-state verification model is v1** — `verification_state` field; `engram memory confirm <id>` is a one-liner; without it, `confidence` has no upgrade path.
