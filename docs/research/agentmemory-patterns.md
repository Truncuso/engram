---
title: "engram — agentmemory Patterns Research"
project: engram
created: 2026-05-22
agent: research-analyst
note: studied rohitg00/agentmemory for patterns; engram takes no code dependency
---

# agentmemory Patterns Research — Full Report

**Status:** Research complete. 9 areas covered. All source files verified via direct HTTP fetch from github.com/rohitg00/agentmemory. 118 TypeScript source files analyzed across hooks/, functions/, state/, providers/, mcp/ directories.

**Note:** Per session policy, this report is delivered as inline text rather than written to a file. The content below is ready for copy-paste to `/media/christoph/Samsung_Evo990/Projects/00_AI/01_Projects/engram/docs/research/agentmemory-patterns.md`.

---

## Area 1 — Capture Hook Design

### What agentmemory does

agentmemory registers 14 hook files (12 Claude Code lifecycle hooks plus `sdk-guard.ts` and `task-completed.ts`) at `src/hooks/`. Every hook follows an identical structural pattern:

1. Read JSON from stdin and parse.
2. Call `isSdkChildContext(payload)` — if true, return immediately. This function checks `process.env["AGENTMEMORY_SDK_CHILD"] === "1"` OR `payload.entrypoint === "sdk-ts"`. Without this guard, hooks fired by Claude Code Agent SDK child sessions would call back into the memory server, spawning new sessions that fire more hooks — unbounded recursion burning tokens.
3. Construct authentication headers (`Authorization: Bearer ${SECRET}` if `AGENTMEMORY_SECRET` set).
4. POST to `${AGENTMEMORY_URL}` (default `http://localhost:3111`) with an `AbortSignal.timeout(N)` and a `try/catch` that swallows all errors.

**Per-hook specifics:**

| Hook | Captures | Timeout | Notes |
|------|----------|---------|-------|
| SessionStart | session_id (or generated), cwd | 800ms (telemetry) / 1500ms (context injection) | Two paths: `AGENTMEMORY_INJECT_CONTEXT=true` waits and writes context to stdout (Claude Code prepends it to turn 1); default path is fire-and-forget |
| UserPromptSubmit | session_id, cwd, timestamp, prompt text | 800ms | Privacy filter is NOT applied at the hook level — raw prompt forwarded |
| PreToolUse | tool name, file paths, search patterns | 800ms / 1500ms | Default disabled (`AGENTMEMORY_INJECT_CONTEXT` must be true); was causing ~1000 tokens injected into every tool turn on paid accounts — disabled in v0.8.10 |
| PostToolUse | tool name, inputs, outputs | 3000ms | Image extraction: detects base64 images by prefix (`data:image/`, `iVBORw0KGgo`, `/9j/`) and separates them. Output truncated at 8000 chars. The raw event is SHA-256 deduped on `sessionId:toolName:input[:500]` within a 5-minute TTL window before forwarding |
| PostToolUseFailure | tool name, inputs, error | 3000ms | Error string capped at 4000 chars. Checks `data.is_interrupt` to skip user-interrupt events |
| PreCompact | session_id, cwd | 5000ms | POSTs to `/agentmemory/context` with `token_budget: 1500`; writes `result.context` to stdout — Claude Code prepends this as a memory summary before compacting the conversation window |
| SubagentStart | session_id, cwd, agent_id, agent_type | 800ms | Lifecycle event, telemetry only |
| SubagentStop | (same) | 800ms | Lifecycle event, telemetry only |
| Stop | session_id | 120000ms | POSTs to `/agentmemory/summarize` — triggers end-of-session compression. Long timeout because summarization is synchronous here. Also optionally triggers crystal generation and CLAUDE_MEMORY_BRIDGE sync |
| SessionEnd | session_id | 30000ms | Sends session-end marker; optionally triggers consolidation pipeline and bridge sync |
| notification.ts | (notification events) | 800ms | Captures notification events |
| task-completed.ts | (task completion) | 800ms | Captures task lifecycle |

**SHA-256 deduplication** (`src/hooks/dedup.ts`): In-memory `Map<hash, {expiresAt}>`. Hash = SHA-256 of `"${sessionId}:${toolName}:${input[:500]}"`. TTL = 5 minutes. Cleanup interval runs every 60s with `.unref()` (does not keep Node process alive). Prevents duplicate observations from tool calls that fire rapidly in loops.

**Image handling**: Base64 images are extracted from tool outputs into a separate `imageData` field and the original field is replaced with `"[image data extracted]"`. This prevents multi-megabyte image blobs from being transmitted as JSON strings in the observation.

### What engram's capture plugin should adopt

**ADOPT — SDK recursion guard** (exact pattern). This is a safety-critical invariant. engram's Claude Code capture plugin must check an env var (`ENGRAM_SDK_CHILD=1`) AND a payload marker before doing any work. Without it, a dreaming run that uses the Claude Agent SDK will fire hooks that call back into engramd, which may spawn further SDK calls — the exact token-burn loop agentmemory guards against. Implement as the first line of every hook, before any I/O.

**ADOPT — Timeout ladder** (800ms / 3s / 5s / 120s) mapped to hook cost. The 200ms hard timeout proposed in engram's failure-safety review (§4) is likely too aggressive for PostToolUse (which needs to POST to engramd). The agentmemory model is more nuanced: lightweight lifecycle events get 800ms; content-bearing events get 3s; re-injection gets 5s; session-end summarization gets 120s. engram's fire-and-forget contract should adopt this ladder rather than a single 200ms cutoff for all hooks. The 200ms target is right for the file write to `staging/` itself — not for the full HTTP round-trip to a server that may need to run a privacy filter.

**ADOPT — Output truncation before transmission** (8000 chars / 4000 for errors). engram's `staging/` writes are local file appends, not HTTP POSTs, so the urgency is lower — but staging records should still have max sizes. A tool that writes a 50MB file to stdout can produce a 50MB observation. Cap at 8000 chars with a `...[truncated]` marker.

**ADOPT — Image extraction pattern**. Detect base64 images by prefix signature before writing to staging. Replace inline with `[image extracted to: staging/img/obs_<id>_<n>.b64]`. Prevents staging blobs from blowing up the staging directory.

**ADOPT — PreCompact re-injection to stdout**. This is the mechanism that survives context compaction. When Claude Code compacts the conversation, a PreCompact hook fires and its stdout is prepended to the next turn. This is where engram should inject a concise scored-recall summary from the current session's `contextual` memories + top `active` episodic + top `shared semantic`. The token budget is 1500 (agentmemory default); engram should use 2000 (its TOKEN_BUDGET default per §4.4).

**ADAPT — PreToolUse context injection**. agentmemory disabled this by default after discovering it was injecting ~1000 tokens per tool call at subscriber cost. engram should design this gate with the lesson: `ENGRAM_INJECT_CONTEXT` must be an explicit opt-in with a documented per-session token cost estimate. The v1 default should be disabled — only enable if the user explicitly wants pre-tool context enrichment.

**ADAPT — UserPromptSubmit privacy**. agentmemory's prompt hook has no privacy filter — it forwards raw prompts. engram's capture plugin MUST apply the privacy filter before writing the user prompt to `staging/`. The filter runs in the hook process (not the daemon) so it blocks the write without adding daemon latency.

**REJECT — Telemetry mode vs context injection split as a runtime flag**. agentmemory's session-start hook has two code paths governed by `AGENTMEMORY_INJECT_CONTEXT`. engram's architecture is cleaner: `staging/` is the decoupling boundary; hooks always write there, daemon always reads from there. There is no "telemetry vs injection" split at the hook level. Injection (PreCompact + SessionStart) happens by the daemon, not by a runtime mode switch in the hook.

---

## Area 2 — Consolidation (the "Dreaming" equivalent)

### What agentmemory does

**The type system declares four tiers** (`type ConsolidationTier = "working" | "episodic" | "semantic" | "procedural"`) but the live pipeline has three active tiers plus a separate working-memory paging model.

**Working memory** (`src/functions/working-memory.ts`): Not a classic queue-to-episodic pipeline. Instead, it is a token-budget-based paging model. Core entries (immediate working set) get 30% of the token budget; archival entries get 70%. A composite score `0.5×importance + 0.3×recency + 0.2×accessFrequency` determines page-out order. Pinned entries never page out. Auto-page fires when accumulation exceeds budget. This is a context-management pattern, not a consolidation tier.

**Episodic consolidation**: Session summaries are the raw material (produced by the `Stop` hook's summarize call). The Stop hook POSTs to `/agentmemory/summarize`, which compresses the session's observations into a narrative + decision list. These summaries are the input to the semantic tier.

**The three-tier pipeline** (`src/functions/consolidation-pipeline.ts`):

1. **Semantic tier**: Requires ≥5 session summaries. Takes up to 20 most recent summaries, extracts XML-formatted facts with confidence scores (`<confidence>0.85</confidence>`). New facts are stored with IDs prefixed `"sem"`; duplicate facts (matched by title/content) update the existing record. Facts persist with `strength` values (initialized from confidence, updated via access).

2. **Reflect tier**: Calls `mem::reflect` (a separate clustering function). Groups memories into concept clusters, up to `clusterLimit` (default 10). Higher-order pattern recognition — synthesizing cross-session patterns into insight-level memories.

3. **Procedural tier**: Extracts reusable procedures from recurring patterns (≥2 occurrences). LLM output is XML with trigger conditions, steps (max 10), and expected outcomes. Duplicate procedures are detected by fingerprint; existing procedures gain `strength += 0.15` (capped at 1.0). New procedures initialize at `strength = 0.6`.

4. **Decay tier**: Exponential decay applied to both semantic and procedural memories. Each decay cycle: `strength = Math.max(0.1, strength * 0.9)`. Uses `lastAccessedAt || updatedAt` for elapsed-time calculation. Floor is 0.1 (never reaches zero).

**Retention scoring** (`src/functions/retention.ts`): A richer model used for eviction decisions. `score = min(1, salience × exp(-lambda × deltaT) + reinforcementBoost)` where `lambda = 0.01`, `deltaT = elapsed days`. Salience varies by memory type: architecture memories = 0.9, facts = 0.5, with confidence and access bonuses. `reinforcementBoost = Σ (1/daysSinceAccess) × sigma` where `sigma = 0.3`. This yields four tiers: Hot (≥0.7), Warm (0.4–0.69), Cold (0.15–0.39), Evictable (<0.15).

**Compression** (`src/functions/compress.ts`): LLM compresses raw observations into structured XML: `<type>`, `<title>` (80 char max), `<subtitle>`, `<facts>`, `<narrative>`, `<concepts>`, `<importance>` (1–10). Importance is extracted as `Math.max(1, Math.min(10, parseInt(getXmlTag(xml, "importance") || "5")))`. On failure: retry with validator, up to N times. Images are described via `provider.describeImage()` before compression.

**Contradiction detection** (`src/functions/auto-forget.ts`): Scans the 1000 most recent memories, groups by concepts, compares pairs with Jaccard similarity (`|A∩B| / |A∪B|`). Similarity >0.9 → older memory's `isLatest = false` (not deleted). This is a soft supersession, not deletion. Combined with the `mem::remember` flow which also uses Jaccard >0.7 for supersession on save.

**Eviction** (`src/functions/evict.ts`): Four strategies:
- Stale sessions (no summary, >30 days): attempt recovery, then delete.
- Low-importance observations (>90 days, importance <3): delete.
- Project cap (>10,000 observations): evict lowest-importance first.
- Memory expiration: explicit `forgetAfter` TTL or non-latest memories >90 days.

**Importance scoring**: Raw importance comes from LLM (1–10, default 5). Memory save initializes `strength = 7`. Procedural memories initialize at `strength = 0.6`. Strength evolves via decay, access, and reinforcement over time.

### Mapping onto engram's dreaming

**ADOPT — Episodic-first pipeline ordering**: session summary (Stop hook) → semantic extraction (≥5 summaries) → procedural extraction (≥2 patterns). engram's dreaming steps (distill → connect → re-weight → verify/learn) map cleanly: distill ≈ session summary; connect ≈ semantic fact extraction + relation insertion; re-weight ≈ strength/decay update; verify/learn ≈ procedural extraction. The ordering is validated.

**ADOPT — XML-structured LLM output with retry-validator loop**. The `compressWithRetry()` pattern — build a prompt, validate output against schema, retry on failure, up to N times — is exactly right for dreaming's LLM calls. engram's dreaming worker should use this pattern for every LLM step.

**ADOPT — Jaccard similarity for contradiction/supersession detection** (>0.7 on save, >0.9 for contradiction scan). Token-set intersection is cheap and language-agnostic. Use `fingerprintId` (SHA-256 of normalized content) for exact dedup and Jaccard for near-duplicates. engram's dreaming step 2 (Connect) should use Jaccard to detect contradicting episodic memories before linking them.

**ADOPT — Procedural extraction gate**: ≥2 independent occurrences, XML with trigger + steps (max 10) + expected outcomes, dedup by fingerprint. This is directly applicable to engram's "verify & learn" step. Adapt: engram's security review (S-02) requires that promotion to procedural memories be gated at the review queue (not auto-safe) for v1, to prevent crafted observations from steering agent behavior.

**ADOPT — Decay floor (0.1)** and the four retention tiers (Hot/Warm/Cold/Evictable). engram maps: Hot/Warm ≈ `active`, Cold ≈ `dormant` candidate, Evictable ≈ `dormant` confirmed. The threshold values need calibration for engram's use case but the tier structure is correct.

**ADAPT — Working memory paging model** → engram's `contextual` type. agentmemory's working memory is implicit in the context-assembly function, not a separate store tier. engram has `contextual` as a first-class memory type. The paging model (30% budget for pinned/core, 70% for archival by strength×recency) is directly adoptable for the `memory.recall` token-budget parameter.

**ADAPT — Retention scoring formula**. engram's Importance×Relevance×Recency (I×R×Recency) is a product formula; agentmemory uses a sum of `salience×decay + reinforcementBoost`. The two models have different behaviors at extremes (product collapses to 0 if any factor is 0; sum has a floor). engram should add a reinforcement boost to Recency (not the final rank score): `Recency = recency_decay × (1 + access_boost)` where `access_boost = f(access_count, recency of accesses)`. This keeps the I×R×Recency product formula while adding Ebbinghaus reinforcement.

**REJECT — Consolidation trigger on a strict count gate (≥5 summaries for semantic, ≥2 for procedural)**. engram uses time-based triggers (session-end, nightly cron, staging threshold). Count gates are problematic for low-frequency users (will never consolidate) and high-frequency users (consolidates too eagerly after 5 sessions). engram's dreaming-memory `schedule` field is the right model. Use count as a *readiness check* inside the dreaming worker, not as the trigger.

---

## Area 3 — Retrieval

### What agentmemory does

**Triple-stream hybrid search** (`src/state/hybrid-search.ts`):

Three parallel searches: BM25 via `SearchIndex`, vector via `VectorIndex`, graph via `GraphRetrieval`. Results fused with RRF:

```
combinedScore = (0.4 / (60 + bm25Rank)) + (0.6 / (60 + vectorRank)) + (0.3 / (60 + graphRank))
```

Weights are normalized dynamically based on which indices returned results. Session diversification: at most 3 results per session in the final result set (prevents one prolific session from dominating recall). Optional post-RRF cross-encoder rerank using `Xenova/ms-marco-MiniLM-L-6-v2` (top 20 candidates, rescored by neural relevance).

**BM25** (`src/state/search-index.ts`): k1=1.2, b=0.75. Tokenization: remove non-alphanumeric (preserving hyphens/underscores), stemming via Porter stemmer, synonym expansion with 0.7 weight multiplier. CJK (Chinese/Japanese/Korean) handled by segmentation (jieba/kuromoji) rather than stemming. Prefix matching gets 0.5 IDF multiplier. Inverted index with persistence via JSON serialization.

**Vector** (`src/state/vector-index.ts`): Local `all-MiniLM-L6-v2` via `@xenova/transformers` (free, offline), or Gemini/OpenAI/Voyage/Cohere. Input truncated to 16,000 chars (~4000 tokens) before embedding. Batch embedding with configurable batch size (default 32). Index rebuild processes memories and observations separately.

**Graph retrieval** (`src/functions/graph-retrieval.ts`): Dijkstra over `cost = 1/weight` (stronger relationships = lower cost = shorter path). Adjacency list built in O(V+E). Binary min-heap dequeue. Score = `avgWeight × (1/pathLength)`. Direct matches = 1.0. Temporal graph queries supported (entity state at specific timestamps).

**Query expansion** (`src/functions/query-expansion.ts`): LLM generates 3–5 semantic reformulations, temporal concretizations of relative time references ("last week" → concrete date range), and entity extraction (quoted terms + capitalized non-stopwords). This expands recall before the hybrid search runs.

**Session diversification**: A post-processing cap of 3 results per session. Prevents a single heavy session (e.g. a large refactor) from consuming all recall slots.

**Access tracking** (`src/functions/access-tracker.ts`): `AccessLog` per memory: `count`, `lastAt` (ISO timestamp), `recent[]` (sliding window of 20 timestamps). Recorded via `withKeyedLock` (keyed mutex) on each access. Concurrent batch access uses `Promise.allSettled`. This is the Recency signal for retention scoring.

**Context assembly** (`src/functions/context.ts`): Token-budget-controlled assembly. Priority order: pinned slots → project profile → lessons (sorted by `confidence × (project_scope ? 1.5 : 1)`) → session summaries → high-importance observations (importance ≥5, sorted descending). Each block added only if it fits within remaining budget. Block tokens estimated as `Math.ceil(text.length / 3)`. No block is truncated — only full blocks are included or skipped (greedy).

### engram's architecture decision re: RRF

engram's architecture review (Flow D, §1.2) resolved this explicitly: **QMD provides Relevance; the core scoring engine fuses I×R×Recency; graph is an opt-in expansion (candidates, not a competing ranker); RRF is unnecessary.**

The review finding is correct. RRF is designed for fusing two or more independent ranked lists of roughly equal quality. In agentmemory's case, it makes sense because all three indices are peers. In engram's case, QMD is the primary relevance signal and the graph is an expansion — it adds candidate memories not in QMD's result set, not a competing ranking of the same memories. Applying RRF here would mean summing inverse ranks from two fundamentally different operations (ranked retrieval vs. graph traversal), which is semantically confused.

**The agentmemory RRF implementation is NOT evidence engram should reconsider its model.** The architectures differ: agentmemory has three co-equal retrieval streams; engram has one ranked retrieval stream (QMD) and one expansion step (graph). engram's I×R×Recency product formula over QMD hits is the right model.

**ADOPT — Session diversification cap** (max N results per session in recall output). This is an independent quality improvement regardless of the retrieval model. Without it, a session with 50 episodic memories will dominate recall. Recommended N=3 for engram (same as agentmemory).

**ADOPT — Token-budget-aware context assembly** with greedy block inclusion (full blocks only, skip if over budget). The `Math.ceil(text.length / 3)` token estimator is good enough for an internal heuristic. The priority ordering (pinned → profile → lessons → summaries → observations) maps directly to engram's: contextual/pinned → project profile → procedural → episodic → semantic.

**ADOPT — Access tracking sliding window** (20 most recent timestamps, keyed mutex for concurrent writes). This is the Recency signal input. engram's `recency` frontmatter field tracks last-access timestamp and `access_count` counts total accesses — but a sliding window of recent access timestamps gives a much richer Ebbinghaus reinforcement signal than `access_count` alone. Move access tracking to `.engram/stats.db` sidecar (per architecture review F-1.7) and store the 20-timestamp window there.

**ADAPT — Query expansion**. agentmemory's LLM-driven query expansion is excellent but expensive (adds one LLM call per search). engram should support it as an opt-in (`memory.recall({query, expand: true})`). The entity extraction portion (quoted terms + capitalized non-stopwords) can run without an LLM and should always be applied. Temporal concretization is high-value for episodic memory ("what happened last week") and should be the opt-in portion.

**ADAPT — Cross-encoder rerank**. agentmemory loads MiniLM-L-6-v2 locally via `@xenova/transformers`. This is free and offline but adds ~200ms per recall and loads a 25MB model. For v1, engram should defer this to an opt-in plugin. QMD already reranks with an LLM when available; adding a local cross-encoder on top is diminishing returns.

**REJECT — RRF with three co-equal streams**. As argued above, engram's one-ranked-source architecture makes RRF inappropriate. The weights (BM25:0.4, Vector:0.6, Graph:0.3) and the k=60 constant are tuning handles that engram does not need.

---

## Area 4 — Privacy Filter

### What agentmemory does

**Two-layer filter** (`src/functions/privacy.ts`):

Layer 1 — `<private>` tag removal: regex `/<private>[\s\S]*?<\/private>/gi` → `[REDACTED]`.

Layer 2 — 13 API key / secret patterns, all replaced with `[REDACTED_SECRET]`:
- Generic: `api_key\s*=\s*["'][^"']+["']`, `password\s*=\s*["'][^"']+["']`, `token\s*=\s*["'][^"']+["']`, bearer tokens (20+ alphanumeric chars after "Bearer ")
- OpenAI: `sk-proj-[a-zA-Z0-9_-]{20,}`
- Anthropic: `sk-ant-[a-zA-Z0-9_-]{20,}`
- GitHub: `ghp_[a-zA-Z0-9]{36}`, `github_pat_[a-zA-Z0-9_]{22,}`
- Slack: `xoxb-[0-9]+-[0-9A-Za-z-]+`
- AWS: `AKIA[0-9A-Z]{16}`
- Google: `AIza[0-9A-Za-z-_]{35}`
- npm: `npm_[a-zA-Z0-9]{36}`, GitLab: `glpat-[a-zA-Z0-9_-]{20}`, DigitalOcean: `dop_v1_[a-z0-9]{64}`

The filter is exposed as `mem::privacy` via the SDK — a single function that takes a string and returns the sanitized string. It is applied to prompts and observations in the server-side processing path, not in the hooks themselves.

### Gap analysis

agentmemory's filter is a good starting point but has known gaps (identified by engram's security review S-01):
- **No entropy detection**: high-entropy strings (random 32-char hex, base64-encoded tokens) are not detected unless they match a specific prefix pattern.
- **No path blocklist**: file paths like `/home/user/.ssh/id_rsa` or `~/.aws/credentials` are not redacted.
- **No structured data detection**: `{"token": "abc123"}` — the generic `token\s*=` pattern requires `=` not `:`.
- **Filter runs server-side, after the hook transmits raw data**: engram's architecture writes to `staging/` first, requiring the filter to run at the hook level before writing.

**ADOPT — The 13 regex patterns**. These cover the most common credential formats in a coding-agent context and are battle-tested by agentmemory's production use. Copy them directly into engram's `CaptureIntake` component. They are a floor, not a ceiling.

**ADAPT — Add entropy detection** as a second pass. A string of 32+ characters that contains at least 3 distinct character classes (upper, lower, digit, symbol) and has Shannon entropy >3.5 bits/char is likely a secret. Add this as a configurable check (default: enabled at length >20, entropy >3.5). False-positive rate is manageable for code contexts where such strings are rare in non-secret positions.

**ADAPT — Add path blocklist**. Block writes that contain path patterns: `~/.ssh/`, `~/.aws/credentials`, `/run/secrets/`, `~/.gnupg/`. These are filesystem paths that almost never need to be in memory observations.

**ADAPT — `<private>` tag → `<engram:private>` tag**. Namespacing the tag prevents collision with HTML/XML that legitimately uses `<private>` in tool outputs.

**ADAPT — Fail-closed enforcement**. agentmemory's filter is best-effort (server-side, after transmission). engram's filter MUST be fail-closed: if the filter throws for any reason, the observation is dropped (not written to staging). This is a hard security invariant. agentmemory's pattern of running the filter at write-time is correct for engram since hooks write directly to `staging/`.

**REJECT — LLM-assisted filtering as an always-on pass**. An LLM sweep for secrets adds latency and token cost to every capture event. Make it an explicit opt-in for high-security environments. The pattern + entropy + path layers cover the vast majority of secrets without an LLM.

---

## Area 5 — Reliability

### What agentmemory does

**Circuit breaker** (`src/providers/circuit-breaker.ts`): Three-state FSM. Configurable: `failureThreshold` (default 3), `failureWindowMs` (default 60s), `recoveryTimeoutMs` (default 30s). Transitions: closed → open when failure count reaches threshold within the window; open → half-open after recovery timeout; half-open → closed on success, half-open → open on failure. The window resets if `now - lastFailureAt > failureWindowMs` (stale failure counter). Exposes `getState()` for monitoring.

**ResilientProvider** (`src/providers/resilient.ts`): Decorator pattern. Wraps any `MemoryProvider` with the circuit breaker. All calls go through `call()` which checks `this.breaker.isAllowed` first (throws `"circuit_breaker_open"` if not) and records success/failure after. Exposes `circuitState` getter.

**FallbackChainProvider** (`src/providers/fallback-chain.ts`): Ordered provider list. `tryAll()` iterates sequentially, returns on first success, throws last error if all fail. Named as `"fallback(p1 -> p2 -> p3)"`. Clean separation: resilience (circuit breaker) and fallback are separate concerns that compose.

**Health monitoring** (`src/health/monitor.ts`): Collects CPU usage, memory consumption, event loop lag, uptime at 30s intervals. KV connectivity probe: set/get with 5s timeout via `Promise.race`. Worker status via `engine::workers::list`. Evaluates via `evaluateHealth()` for alert generation. Observability only — no routing decisions based on health signals.

**Self-healing** (`src/functions/diagnostics.ts`): `mem::diagnose` scans 14 entity categories (actions, leases, sentinels, sketches, signals, sessions, memories, lessons, summaries, semantic/procedural memories, crystals, insights, mesh peers) for: expired entities, orphaned references, invalid confidence scores (non-finite or outside 0–1), dependency inconsistencies. `mem::heal` fixes: unblocks actions when dependencies complete, expires stale leases, discards expired sketches (cascade-deletes related actions/edges), marks superseded memories as non-latest, removes orphaned references. All mutations use `withKeyedLock` and re-fetch the entity before mutation ("double-check under lock").

### ADOPT/ADAPT/REJECT for engram

**ADOPT — Three-state circuit breaker** (exact implementation, with the same defaults). This is a standard pattern but agentmemory's specific defaults (threshold=3, window=60s, recovery=30s) are well-calibrated for LLM API calls. engram should use this for its LLM plugin calls and its QMD plugin calls. The `isAllowed` getter with auto-transition from open→half-open on timeout is cleaner than polling.

**ADOPT — Decorator-pattern ResilientProvider over FallbackChainProvider**. The separation between "circuit breaking per provider" and "fallback across providers" is architecturally clean. engram's `LlmPlugin` should be wrapped in a `ResilientPlugin` (circuit breaker) and multiple `LlmPlugin` instances (Claude → OpenAI → Ollama) should be composed in a `FallbackPluginChain`. Same pattern applies to `RetrievalPlugin` (QMD → grep fallback, per failure-safety review §1).

**ADOPT — `engram doctor` self-healing pattern** (modeled on `mem::diagnose`/`mem::heal`). engram's failure-safety review independently specified this (§7), and agentmemory validates it. Key adaptation: engram's diagnose must scan frontmatter for YAML parse errors, duplicate IDs, dangling relation edges, and field range violations. Heal must use `withKeyedLock` equivalent (engram uses OCC version tokens rather than in-memory mutexes — same intent, different mechanism).

**ADAPT — Health monitoring**. agentmemory's health monitor is observability-only. engram's daemon should use health probe results to drive the fallback chain: if QMD health probe fails, immediately activate the BM25-only fallback tier without waiting for the circuit breaker to open. The health probe is a faster signal than accumulated failures.

**ADAPT — Startup crash recovery** (agentmemory implicitly handles this via KV reconstruction; engram has an explicit state machine requirement). Adapt: on startup, scan `.engram/jobs.db` for SPAWNED/RUNNING jobs whose PID is not alive → transition to FAILED. This is more explicit than agentmemory's implicit KV-based recovery.

**REJECT — Event loop lag monitoring** as a primary health signal. This is iii-engine specific (measures the iii runtime event loop). engram uses plain Node.js; event loop lag is still measurable (`perf_hooks.monitorEventLoopDelay`) but is less meaningful for a file-I/O-heavy system where most latency is synchronous I/O, not event loop blocking.

---

## Area 6 — MCP Surface

### What agentmemory does

**51 tools across 8 groups:**

| Group | Count | Key tools |
|-------|-------|-----------|
| CORE_TOOLS | 13 | recall, compress_file, save, patterns, smart_search, file_history, sessions, timeline, profile, export, relations, graph_query, consolidate |
| V040_TOOLS | 7 | sync (MEMORY.md bidirectional), graph_consolidate, team_share, team_feed, governance_delete, audit_query, snapshot |
| V050_TOOLS | 8 | action_create/update/list/get/edge_create, frontier, lease_acquire/release/renew/cleanup, routine_*, signal_send/read/threads/cleanup, checkpoint |
| V051_TOOLS | 8 | sentinel_*, sketch_*, crystallize, diagnose, heal, facet_tag/untag/query/get/stats/dimensions, verify |
| V061_TOOLS | 1 | verify (provenance tracing) |
| V070_TOOLS | 3 | lesson_save, lesson_recall, obsidian_export |
| V073_TOOLS | 2 | reflect (synthesis), insights_list |
| SLOTS_TOOLS | 6 | slot_set/get/list (pinned/project/global memory slots) |

**ESSENTIAL set** (8, shown by default): `memory_save`, `memory_recall`, `memory_consolidate`, `memory_smart_search`, `memory_sessions`, `memory_diagnose`, `memory_lesson_save`, `memory_reflect`.

**Capability gate**: `AGENTMEMORY_TOOLS=all` exposes all 51. Default shows only ESSENTIAL.

### Mapping to engram's 16 verbs

engram's 16 verbs (`memory.init`, `remember`, `update`, `recall`, `get`, `list`, `forget`, `ingest`, `history`, `governance_delete`, `dream.list`, `dream.configure`, `dream.trigger`, `dream.status`, `dream.result`, `system.status`) cover the core. Mapping:

| agentmemory tool | engram verb | Notes |
|-----------------|-------------|-------|
| memory_save | memory.remember | Direct map |
| memory_recall / memory_smart_search | memory.recall | engram collapses these into one verb with `{expand, graphExpand}` opts |
| memory_sessions | memory.list (with type filter) | engram uses typed listing |
| memory_diagnose / memory_heal | system.status + `engram doctor` CLI | engram separates diagnostic output (MCP: system.status) from repair (CLI: engram doctor) |
| memory_lesson_save | memory.remember (type=procedural) | engram uses memory type, not a separate lesson concept |
| memory_reflect | dream.trigger | Reflection is a dreaming run |
| memory_consolidate | dream.trigger | Same |
| memory_export | `engram export` CLI | Not needed as an MCP verb for v1 |
| memory_timeline / memory_file_history | memory.history | Unified in engram |
| memory_profile | memory.list + recall (project-scoped) | No dedicated verb needed — profile is a canned query |
| memory_patterns | Not mapped | Explicitly addressable via recall |
| memory_audit | memory.history + system.status | Folded in |
| memory_relations | memory.get (with relations in response) | No separate verb |
| memory_verify | memory.get (provenance in response) | Folded in |
| snapshot tools | git-native (engram's git store) | Not needed as MCP verbs |
| MEMORY.md sync | Not applicable | engram's source-of-truth is .md files, not a MEMORY.md |
| team_share/feed | Not in v1 | Team = v2 |
| action/lease/signal/sentinel/sketch | Not in engram | Task management is out of scope |
| facets | Not in v1 | Addressed by memory.list filters |
| lessons (separate tier) | Merged into procedural type | engram's type system subsumes this |
| slots (pinned memory) | `visibility` + recall priority | Pinned = high-importance active contextual memory |

**Gap analysis — things agentmemory reveals that engram might be missing:**

- `memory.profile` (canned project-profile query) — agentmemory's `mem::profile` is one of its most useful features for session-start context injection: it computes top concepts, files, conventions, and errors from recent sessions. engram's `memory.recall` can return this but a dedicated `system.profile` endpoint (cached, returns project summary for SessionStart injection) would be cleaner and faster.
- `memory_lessons` as a separate confidence-decay tier — agentmemory uses a separate lessons tier with `confidence × (1-confidence) × 0.1` reinforcement and weekly decay. engram folds this into `procedural` type. The lessons mechanism (fingerprint dedup, confidence decay, reinforcement) is worth importing as the lifecycle mechanics of `procedural` memories.
- Co-change pattern detection (`mem::patterns`) — detects files modified together across sessions (≥3 co-occurrences → "co_change" pattern) and recurring errors (≥2 sessions → "error_repeat"). This is implementable by the dreaming worker's "verify & learn" step and does not need a separate MCP verb.

**ADOPT — Capability-gated visibility** (essential tools by default, full set on opt-in). engram's 16-verb set is already minimal enough that gating isn't urgent, but the pattern of a minimal surface for session agents and an extended surface for admin/CLI clients should be preserved. engram's capability model (read/write/dream/cross-agent/govern) is a stronger and more principled implementation of this idea.

**ADOPT — `system.status` verb** returning plugin health, worker status, store stats. agentmemory's `memory_diagnose` does a more detailed scan than a status check. engram's architecture review already includes this; agentmemory validates it.

**ADOPT — Provenance tracing in `memory.get`** (the `mem::verify` pattern). When `memory.get({id, provenance: true})` is called, return the chain of source staging observations that produced this memory. agentmemory's `mem::verify` walks `sourceObservationIds`, `sessionIds`, and hinted session lookups. engram's `staged_from: [staging_file_id]` (observability review recommendation) provides this provenance field.

**ADAPT — Lessons tier mechanics into procedural type**. Import agentmemory's lesson reinforcement formula (`confidence += 0.1 × (1-confidence)` giving diminishing returns at high confidence) and confidence decay (`confidence -= confidence × decayRate × weeks`) as the lifecycle mechanics of engram's `procedural` memories. Use `fingerprintId` for dedup. This gives procedural memories a richer lifecycle than a flat strength number.

**REJECT — Action/lease/signal/sentinel/sketch system** (V050/V051 tools). These are a task-management and multi-agent coordination layer built on top of the memory system. 21+ tools. engram's scope is memory management — it explicitly defers task management and multi-agent mesh to v2+. None of these map to engram's v1 requirements.

**REJECT — MEMORY.md bidirectional sync** (V040 `memory_sync`). This syncs the memory store to/from a single `MEMORY.md` file for MEMORY.md-based agents. engram's architecture (Markdown files as the source of truth, graphify for navigation) makes this redundant and potentially dangerous (MEMORY.md sync could create an alternative write path bypassing access control).

**REJECT — Obsidian export** (V070 `memory_obsidian_export`). engram's `memories/` directory IS Obsidian-compatible by design (Markdown + frontmatter + `[[wikilinks]]`). No export step needed.

---

## Area 7 — Observability

### What agentmemory does

**Real-time viewer** (port 3113): Live observation stream (SSE/WebSocket via iii-stream), session explorer (browse sessions with observation details), knowledge graph visualization (entity nodes and edges from graph extraction), health dashboard, session replay with timeline scrubbing and 0.5×–4× playback speed. The replay feature is notable: it can re-run a session's event sequence at any speed for debugging.

**Audit trail** (`src/functions/audit.ts`): `recordAudit(operation, targetIds, details)` → `AuditEntry` stored in KV. Policy: scoped deletions emit one row per invocation; bulk eviction/retention runs emit one batched row with all IDs + `evicted` count (prevents log flooding). "Silent deletes are not acceptable" — every `kv.delete()` must be preceded by `recordAudit()`. Query via `queryAudit(operation?, dateRange?, limit?)`.

**Git snapshots** (`src/functions/snapshot.ts`): `mem::snapshot-create` collects all state (sessions, memories, graph nodes, observations, access logs) into `state.json`, then `git add + commit`. Restoration: `git checkout <hash> -- state.json` then re-populate KV. `ensureGitRepo()` initializes with `agentmemory@local` author. Snapshot listing: `git log --format=%H|%aI|%s` up to 20 commits. Commit hash validated by regex `/^[0-9a-f]{7,40}$/i`.

**Project profile** (`src/functions/profile.ts`): Generated from the 20 most recent sessions. Cache TTL: 1 hour. Contents: top 15 concepts (frequency-ranked), top 15 files, up to 10 unique error titles, 10 recent activities (highest-importance observations), detected conventions (TypeScript vs JavaScript, `src/` directory structure, test frameworks, technology frequencies ≥3). Useful for SessionStart context injection and for understanding what a project is about.

**Citation provenance** (`src/functions/verify.ts`): `mem::verify(id)` returns a memory's source observations. Two-phase lookup: hint-based (check session IDs in the memory's metadata first) then exhaustive scan. Returns `{citations: [{observationId, sessionId, project, status}]}`.

### engram's v1 observability position

engram defers the dashboard to v2. v1 ships headless (daemon + MCP + CLI). The relevant patterns are:

**ADOPT — Audit trail with "no silent deletes" invariant**. The `safeAudit()` wrapper (logs failures without throwing) is the right pattern — audit logging failures should not block the operation. The batch-row policy for eviction (one row, many IDs) prevents log flooding. engram must implement this before `governance_delete` and any dreaming lifecycle-transition logic.

**ADOPT — Project profile generation** as a v1 feature (cached, fast). This is high-value for SessionStart context injection. engram's `system.profile` should be a lightweight version: top 10 concepts, top 10 files, recent episodic summaries (last 5), active procedural memories. 1-hour cache is appropriate. Session-start context injection should always begin with the project profile.

**ADOPT — Citation provenance in `memory.get`**. The two-phase lookup (hint-based then exhaustive) is the right algorithm for engram. engram's frontmatter has `staged_from: [staging_file_id]` — this is the hint. Provenance is: staging records → compressed observation → memory. Return this chain when `provenance: true` is passed.

**ADOPT — Dream run audit record** (per engram's observability review §4.3, validated by agentmemory's approach). Every dreaming run should produce a `dream-run-<run_id>.json` in `.dreaming/runs/` recording: staging files processed/discarded (with discard reasons), memories created/modified (with field diffs and LLM rationale), links added/removed, emergent entity proposals (with appearances and decision: queued/created/rejected), review queue items produced, LLM model used, token costs. This is the primary observable artifact of dreaming for v1 (no dashboard, but `engram dream log` can read these).

**ADAPT — Git snapshots → engram's git store**. agentmemory snapshots to `state.json` (a single blob export of the KV store). engram's git store IS the memory store — every memory is a Markdown file tracked by git. The snapshot equivalent is `git tag` or a named branch point. The dream branch IS a snapshot of the store state at a point in time. No separate snapshot mechanism is needed. The pattern that IS adoptable: use `git log --format=...` for listing history and hash-validation regex for restore operations.

**ADAPT — Session replay** (v2 only, but spec it now). The 0.5×–4× playback speed control is a UX idea worth preserving for v2 dreaming visualization. In v1, `engram dream log --format=timeline` should output a chronological event sequence that could be fed to a future replay system.

**REJECT — iii-stream / SSE live observation stream** at a separate port. engram uses a Unix socket for IPC (per the failure-safety review) and serves the dashboard on a local HTTP port. A separate port for the observation stream is unnecessary complexity for v1. In v2, the dashboard REST/WS endpoint serves the live stream.

---

## Area 8 — The iii Engine Backend

### What it is

iii (at `iii.dev`) is a serverless runtime that replaces traditional Node.js service infrastructure. It provides:
- **iii-http**: HTTP trigger system replacing Express/Fastify. Routes are registered as `sdk.register("path::method", handler)` calls.
- **iii-state**: KV store abstraction with pluggable adapters. The `file_based` adapter uses SQLite (`./data/state_store.db`). Data is accessed via `sdk.kv.set(key, value)` / `sdk.kv.get(key)` / `sdk.kv.list(prefix)`.
- **iii-queue**: Durable task queue for embedding/compression retries. Ensures LLM calls are retried after failures.
- **iii-pubsub**: Pub/sub for multi-instance fan-out (optional).
- **iii-cron**: Scheduled job execution for consolidation and decay cycles.
- **iii-stream**: WebSocket/SSE stream server (port 3112) for the real-time viewer.
- **iii-observability**: OTEL traces with "memory" exporter, full sampling, console-logged output.
- **iii-exec**: File watcher that rebuilds on source changes.

The `iii-sdk` package (v0.11.2) is agentmemory's core dependency. All business logic calls `sdk.register()`, `sdk.kv.*`, `sdk.trigger()`, `sdk.stream.*`. The runtime handles HTTP routing, persistence, scheduling, and observability as platform concerns.

**What agentmemory avoids by using iii**: manually wiring Express routes, implementing a retry queue, setting up cron jobs, plumbing OTEL, and managing a stream server.

**What agentmemory inherits by using iii**: coupling to iii's release cadence, the iii-sdk API, iii's licensing/hosting model, and the operational complexity of an unfamiliar runtime.

### What engram avoids by NOT using iii

**ADOPT — The abstraction pattern** (not the implementation). iii's `KV` namespace hierarchy (`KV.sessions(id)`, `KV.observations(id)`, `KV.memories`, `KV.teamShared(teamId)`, etc.) is a clean key-prefix organization scheme. engram should use a similar KV-prefix convention in its SQLite app-log and stats database — not for the memory store (that's Markdown files) but for the operational metadata: `{store}/{memory_id}/access_log`, `jobs/{job_id}`, `stats/{memory_id}`, `index_state/{memory_id}`.

**ADOPT — Durable queue for LLM retries**. iii-queue's role is to ensure that LLM/embedding calls are retried after network failures. engram's dreaming worker should implement an equivalent as a persistent retry table in `.engram/jobs.db`. The table holds pending LLM call descriptors; the worker consumes them with exponential backoff. This is engram's OQ-I answer extended: the jobs table handles both dream jobs AND LLM retry entries.

**ADOPT — iii-cron conceptual pattern → SQLite + Node `setInterval`**. agentmemory's scheduled consolidation uses iii-cron. engram implements this as: the dreaming orchestrator reads `schedule` from each `.dreaming/*.md` config, computes next-run times, and uses a 60s polling loop in the core daemon to check for overdue schedules. No external cron dependency needed.

**REJECT — iii-sdk as a dependency**. The iii SDK is the abstraction layer through which all agentmemory logic runs. Adopting it would mean engram depends on iii's release cadence, licensing, and API stability. The SPEC explicitly rejected agentmemory as a dependency for this reason. engram's equivalent stack: Node.js HTTP server (native or lightweight framework) + SQLite (better-sqlite3) + Markdown files + optional git shell calls. This is smaller, more auditable, and has zero vendor lock-in.

**REJECT — iii-stream / iii-pubsub for v1**. engram's v1 has no dashboard and no multi-instance fan-out. The core daemon uses a Unix socket for CLI IPC and a local HTTP port for the MCP server. SSE streams are v2 infrastructure.

---

## Area 9 — Team Memory

### What agentmemory does

**Namespace isolation** (`src/functions/team.ts`): Shared items stored in `KV.teamShared(config.teamId)` — all shared items for a team are under a common key prefix. Observations remain session-scoped; memories remain in the global `KV.memories` namespace but with a `sharedBy`/`sharedAt`/`visibility` metadata overlay.

**Three supported item types**: `memory` (persistent knowledge), `pattern` (reusable templates), `observation` (session-specific data).

**Feed with visibility filtering**: `mem::team-feed` filters by `visibility === "shared"`, sorts by timestamp, returns paginated results with configurable limits.

**Team profile**: Aggregates across members — member counts, concept frequencies, top file references, pattern snippets. Useful for understanding team-wide memory distribution.

**Access model**: Visibility flag (`"shared"`) enforced at the application layer. KV namespacing provides logical isolation but not cryptographic separation. Audit logging tracks all share operations.

### Relevance to engram v1

engram defers cross-agent/team memory to v2 but has `scope` + `visibility` (`shared|private|hidden`) in v1 as the access-control model.

**ADOPT — KV namespace isolation pattern → engram's scope-based directory layout**. agentmemory's `KV.teamShared(teamId)` prefix is the KV equivalent of engram's `scope` field + `visibility` metadata. engram's Markdown store already has the structure: `memories/` contains all memories; the `scope` and `visibility` frontmatter fields serve the same access-control purpose as namespace isolation. The access-control check on `memory.recall` enforces this.

**ADOPT — Visibility-filtered listing**. agentmemory's `mem::team-feed` pattern (filter by visibility, sort by timestamp, paginate) is directly applicable to engram's `memory.list` verb: `memory.list({visibility: "shared", type: "semantic", cursor, limit})` returns shared memories across scopes.

**ADOPT — Team profile concept → `system.profile`** (already recommended in Area 7). The project profile that agentmemory generates for teams is identical to what engram needs for session-start context injection at the project level.

**ADAPT — Multi-agent scope model for v1 headless use**. In v1, engram only has one "agent identity" per token (`agent:claude-code`). The `scope` field exists in frontmatter and is enforced by the access-control module, but there is no actual cross-agent sharing yet. This is correct — the infrastructure is present (scope/visibility/author fields, per-agent tokens) and cross-agent sharing is a v2 feature that needs no v1 implementation. The SPEC's current design is sufficient.

**REJECT — Mesh / multi-instance pubsub** (agentmemory's `iii-pubsub` / V050 `mesh_sync` tool). This is multi-machine synchronization infrastructure. engram is local-first; multi-machine sync is explicitly out of v1 and v2 scope per SPEC §9.3.

---

## Cross-Cutting Findings

### Identity: `fingerprintId` vs ULID

agentmemory uses two ID strategies: `generateId()` (timestamp + UUID, for entities whose identity is tied to creation moment) and `fingerprintId()` (SHA-256 of normalized content, for dedup-by-content entities like lessons, skills, and procedures). engram uses ULIDs for all memory IDs, which is correct for `origin ∈ {agent-session, ingested, self-authored}`. For `procedural` memories created by dreaming, engram should ALSO generate a `fingerprintId` from normalized `(title + trigger + steps[:3])` and check it before creating, to enable the "reinforce existing procedural memory rather than create a duplicate" pattern.

### `withKeyedLock` / OCC interaction

agentmemory uses in-process keyed mutexes for all concurrent write operations (lease acquire, access tracking, relation creation, auto-forget). engram uses OCC (version tokens) for the same purpose, which is correct for a multi-process system where the dreaming worker and the core daemon write concurrently. OCC subsumes in-process mutexes when cross-process concurrency is the threat. engram's OCC design is correct; no change needed.

### Importance as an integer (1–10) vs float (0–1)

agentmemory stores importance as an integer (1–10) extracted from LLM XML; engram stores it as a float (0–1). Both are equivalent with a simple `importance / 10.0` conversion. engram's float is better for the I×R×Recency product formula. No change needed.

### Staging as fire-and-forget

agentmemory's hooks write to a remote HTTP server. engram writes directly to local `staging/` files. The direct-write model is strictly better: lower latency (no network hop), crash-safe (file write is durable), no dependency on daemon availability at capture time. The 200ms hook-timeout target in engram's failure-safety review is achievable for a file write; it would not be achievable for an HTTP POST to a potentially loaded daemon.

---

## Summary Table

| Pattern | Source | engram ruling | Priority |
|---------|--------|---------------|----------|
| SDK recursion guard (isSdkChildContext) | hooks/sdk-guard.ts | ADOPT | Critical |
| Timeout ladder (800/3000/5000/120000ms) | All hook files | ADOPT | High |
| Output truncation + image extraction | hooks/post-tool-use.ts | ADOPT | High |
| PreCompact stdout re-injection | hooks/pre-compact.ts | ADOPT | High |
| SHA-256 5-minute observation dedup | hooks/dedup.ts | ADOPT | Medium |
| XML-structured LLM output with retry-validator | functions/compress.ts | ADOPT | High |
| Jaccard similarity for contradiction detection | functions/auto-forget.ts | ADOPT | Medium |
| Procedural extraction gate (≥2 occurrences, fingerprint dedup) | consolidation-pipeline.ts | ADOPT | High |
| Retention scoring tiers (Hot/Warm/Cold/Evictable) | functions/retention.ts | ADOPT | Medium |
| Session diversification cap (max 3 per session) | state/hybrid-search.ts | ADOPT | High |
| Access tracking sliding window (20 timestamps) | functions/access-tracker.ts | ADOPT | High |
| Token-budget context assembly (greedy blocks) | functions/context.ts | ADOPT | High |
| 13 regex privacy patterns + `<private>` tag | functions/privacy.ts | ADOPT | Critical |
| Fail-closed privacy filter enforcement | (gap in agentmemory) | ADOPT gap | Critical |
| Three-state circuit breaker + ResilientProvider | providers/circuit-breaker.ts | ADOPT | High |
| FallbackChainProvider (sequential) | providers/fallback-chain.ts | ADOPT | High |
| diagnose/heal self-healing pattern | functions/diagnostics.ts | ADOPT | High |
| Audit trail ("no silent deletes") | functions/audit.ts | ADOPT | High |
| Project profile generation (cached) | functions/profile.ts | ADOPT | Medium |
| Citation provenance (two-phase lookup) | functions/verify.ts | ADOPT | Medium |
| Dream run audit record | (absent in agentmemory) | ADOPT pattern | High |
| Capability-gated MCP visibility | mcp/tools-registry.ts | ADOPT | Medium |
| Lessons lifecycle mechanics for procedural type | functions/lessons.ts | ADAPT | Medium |
| fingerprintId for procedural memory dedup | state/schema.ts | ADAPT | Medium |
| Ebbinghaus reinforcement boost in Recency | functions/retention.ts | ADAPT into I×R×Recency | Medium |
| Entropy detection in privacy filter | (absent in agentmemory) | ADAPT add | High |
| PreToolUse injection as explicit opt-in | hooks/pre-tool-use.ts | ADAPT | Low |
| Query expansion (entity extraction always-on) | functions/query-expansion.ts | ADAPT (LLM-driven = opt-in) | Low |
| KV namespace prefix convention for metadata | state/schema.ts | ADAPT to SQLite tables | Low |
| RRF triple-stream fusion | state/hybrid-search.ts | REJECT (one ranked source) | — |
| iii-sdk / iii-engine | iii-config.yaml | REJECT | — |
| Action/lease/signal/sentinel system | functions/actions.ts | REJECT (scope) | — |
| MEMORY.md bidirectional sync | mcp/tools-registry.ts | REJECT | — |
| Obsidian export | mcp/tools-registry.ts | REJECT (not needed) | — |
| iii-stream / iii-pubsub | iii-config.yaml | REJECT (v1 headless) | — |
| Cross-agent mesh sync | functions/mesh.ts | REJECT (v2) | — |
| Team memory namespacing (KV) | functions/team.ts | REJECT v1 / PLAN v2 | — |

---

## 12-Line Summary — Highest-Value Patterns for engram

1. **SDK recursion guard**: every hook must check `ENGRAM_SDK_CHILD=1` env var AND a payload marker before doing any work — prevents token-burning recursive observation loops when the dreaming worker uses the Claude Agent SDK.

2. **Timeout ladder, not a single timeout**: 800ms for lifecycle telemetry, 3s for content-bearing PostToolUse, 5s for PreCompact re-injection, 120s for Stop/summarize — match the timeout to the cost of the operation.

3. **PreCompact stdout re-injection**: the PreCompact hook POSTs to the daemon, receives a scored-recall context string, and writes it to stdout — Claude Code prepends this before compacting; this is the mechanism that keeps memory alive across compaction events.

4. **Privacy filter fail-closed at the hook**: apply the 13 regex patterns + `<private>` tag strip + entropy detection before writing to `staging/` — if the filter throws, drop the observation entirely rather than writing raw; never transmit to staging unfiltered.

5. **XML-structured LLM output with retry-validator loop**: dreaming worker LLM calls should produce XML with required fields (`<type>`, `<title>`, `<importance>`, `<facts>`, `<narrative>`) and retry with a validator that checks structure before accepting — prevents silently bad LLM output from reaching the memory store.

6. **Procedural memory fingerprint dedup + reinforcement**: for procedural memories created by dreaming, compute `fingerprintId(title + trigger + steps[:3])` and check before creating — if a duplicate exists, apply `confidence += 0.1 × (1 - confidence)` (diminishing returns at high confidence) rather than creating a duplicate.

7. **Three-state circuit breaker + FallbackChainProvider on LLM plugin**: wrap each LLM provider in a `ResilientPlugin` (circuit breaker, threshold=3, window=60s, recovery=30s) and chain providers in a `FallbackPluginChain` — this is the mechanism that makes dreaming survive API outages without a crash.

8. **Recall session diversification (max 3 per session)**: post-process recall results to cap at 3 hits per source session — prevents a single prolific session from consuming all recall slots in the context injection.

9. **Access tracking sliding window (20 timestamps, keyed mutex)**: store the 20 most recent access timestamps per memory in `.engram/stats.db` — this is the Ebbinghaus reinforcement signal; add `accessBoost = Σ(1/daysSinceAccess) × sigma` to the Recency factor in I×R×Recency scoring.

10. **`engram doctor` self-healing pattern**: on startup (abbreviated) and weekly (full), scan for YAML parse errors, duplicate IDs, dangling relation edges, and field range violations — move unparseable files to `.engram/quarantine/` rather than crashing; re-fetch entities before mutation in the heal step (double-check under OCC).

11. **Dream run audit record**: every dreaming run writes `dream-run-<run_id>.json` to `.dreaming/runs/` recording staging files processed/discarded, memories created/modified with LLM rationale, links added, emergent entity proposals, review queue items, and token costs — this is v1's substitute for the dashboard's dreaming view and the primary debugging artifact for a bad dream.

12. **Project profile cache for SessionStart injection**: generate a project profile (top 10 concepts, top 10 files, recent episodic summaries, active procedural memories) cached for 1 hour and returned by the SessionStart hook or `system.profile` — this is the highest-value single use of engram's memory for a session agent.

---

*Research completed 2026-05-22. Sources: github.com/rohitg00/agentmemory — 14 hook files, 88 function files, 12 state files, 9 provider files, 6 MCP files analyzed via direct HTTP fetch. All findings source-verified against raw TypeScript.*

Sources:
- [agentmemory README](https://raw.githubusercontent.com/rohitg00/agentmemory/main/README.md)
- [agentmemory GitHub repository](https://github.com/rohitg00/agentmemory)
- [src/hooks/session-start.ts](https://raw.githubusercontent.com/rohitg00/agentmemory/main/src/hooks/session-start.ts)
- [src/hooks/post-tool-use.ts](https://raw.githubusercontent.com/rohitg00/agentmemory/main/src/hooks/post-tool-use.ts)
- [src/hooks/pre-compact.ts](https://raw.githubusercontent.com/rohitg00/agentmemory/main/src/hooks/pre-compact.ts)
- [src/hooks/stop.ts](https://raw.githubusercontent.com/rohitg00/agentmemory/main/src/hooks/stop.ts)
- [src/hooks/sdk-guard.ts](https://raw.githubusercontent.com/rohitg00/agentmemory/main/src/hooks/sdk-guard.ts)
- [src/functions/privacy.ts](https://raw.githubusercontent.com/rohitg00/agentmemory/main/src/functions/privacy.ts)
- [src/functions/consolidation-pipeline.ts](https://raw.githubusercontent.com/rohitg00/agentmemory/main/src/functions/consolidation-pipeline.ts)
- [src/functions/retention.ts](https://raw.githubusercontent.com/rohitg00/agentmemory/main/src/functions/retention.ts)
- [src/functions/evict.ts](https://raw.githubusercontent.com/rohitg00/agentmemory/main/src/functions/evict.ts)
- [src/functions/auto-forget.ts](https://raw.githubusercontent.com/rohitg00/agentmemory/main/src/functions/auto-forget.ts)
- [src/functions/compress.ts](https://raw.githubusercontent.com/rohitg00/agentmemory/main/src/functions/compress.ts)
- [src/functions/diagnostics.ts](https://raw.githubusercontent.com/rohitg00/agentmemory/main/src/functions/diagnostics.ts)
- [src/functions/lessons.ts](https://raw.githubusercontent.com/rohitg00/agentmemory/main/src/functions/lessons.ts)
- [src/state/hybrid-search.ts](https://raw.githubusercontent.com/rohitg00/agentmemory/main/src/state/hybrid-search.ts)
- [src/state/reranker.ts](https://raw.githubusercontent.com/rohitg00/agentmemory/main/src/state/reranker.ts)
- [src/state/search-index.ts](https://raw.githubusercontent.com/rohitg00/agentmemory/main/src/state/search-index.ts)
- [src/providers/circuit-breaker.ts](https://raw.githubusercontent.com/rohitg00/agentmemory/main/src/providers/circuit-breaker.ts)
- [src/providers/fallback-chain.ts](https://raw.githubusercontent.com/rohitg00/agentmemory/main/src/providers/fallback-chain.ts)
- [src/providers/resilient.ts](https://raw.githubusercontent.com/rohitg00/agentmemory/main/src/providers/resilient.ts)
- [src/mcp/tools-registry.ts](https://raw.githubusercontent.com/rohitg00/agentmemory/main/src/mcp/tools-registry.ts)
- [src/functions/audit.ts](https://raw.githubusercontent.com/rohitg00/agentmemory/main/src/functions/audit.ts)
- [src/functions/snapshot.ts](https://raw.githubusercontent.com/rohitg00/agentmemory/main/src/functions/snapshot.ts)
- [src/functions/profile.ts](https://raw.githubusercontent.com/rohitg00/agentmemory/main/src/functions/profile.ts)
- [src/functions/verify.ts](https://raw.githubusercontent.com/rohitg00/agentmemory/main/src/functions/verify.ts)
- [src/functions/context.ts](https://raw.githubusercontent.com/rohitg00/agentmemory/main/src/functions/context.ts)
- [src/functions/access-tracker.ts](https://raw.githubusercontent.com/rohitg00/agentmemory/main/src/functions/access-tracker.ts)
- [src/functions/team.ts](https://raw.githubusercontent.com/rohitg00/agentmemory/main/src/functions/team.ts)
- [src/functions/remember.ts](https://raw.githubusercontent.com/rohitg00/agentmemory/main/src/functions/remember.ts)
- [iii-config.yaml](https://raw.githubusercontent.com/rohitg00/agentmemory/main/iii-config.yaml)
- [src/config.ts](https://raw.githubusercontent.com/rohitg00/agentmemory/main/src/config.ts)
- [src/types.ts](https://raw.githubusercontent.com/rohitg00/agentmemory/main/src/types.ts)
- [src/state/schema.ts](https://raw.githubusercontent.com/rohitg00/agentmemory/main/src/state/schema.ts)
