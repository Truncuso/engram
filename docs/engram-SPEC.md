---
title: "engram — Agentic Memory System — SPEC v2.1"
project: engram
repo: github.com/Truncuso/engram
version: v2.1 (2026-05-26)
status: APPROVED — implementable; folds Round-3 consistency review + 4 de-risking spikes
supersedes: SPEC v2 (2026-05-22), SPEC v1 (2026-05-22 draft) — see git history
inputs:
  - docs/_history/round-1-2/security-review.md
  - docs/_history/round-1-2/architecture-review.md
  - docs/_history/round-1-2/failure-safety-review.md
  - docs/_history/round-1-2/observability-review.md
  - docs/_history/round-1-2/SYNTHESIS.md  (D-1..D-7, R-1..R-5)
  - docs/research/qmd-graphify-verification.md
  - docs/research/agentmemory-patterns.md
  - docs/research/mcp-agentsdk-contract.md
  - docs/research/memory-reliability-literature.md
tags: [AGI_AgenticAI_Memory, AGI_AgenticAI_Memory_CodingAgent, spec, v2]
---

# engram — Agentic Memory System — SPEC v2

A standalone, self-contained, installable application that gives AI coding
agents a **persistent, principled, self-organizing memory** — and a *dreaming*
process that consolidates and learns from it.

This is **SPEC v2.1**: SPEC v2 (the rewrite that folded two parallel review
rounds — security, architecture, failure-safety, observability — and four
research reports) after a **Round-3 internal-consistency review** and **four
de-risking spikes** run before implementation. v2.1 fixes 14 contradictions the
merge introduced, closes 2 design gaps, folds in spike findings, and closes the
last 3 open questions. It is intended to be implementable end-to-end.

### Changelog v2 → v2.1 (2026-05-26)

Round-3 review found **no architectural flaws**; the fixes below are mechanical
consistency repairs + decisions confirmed by spikes.

**Scoring & memory model:**
- **C1** `human-verified` no longer lowers `m_v` below an origin's base — it
  *floors* m_v upward, never down (§3.6).
- **C2** importance's double role (additive term + decay modulator) is now
  stated as intentional and bounded (§3.6).
- **C3** `m_v(origin, confidence)` computation defined explicitly — confidence is
  a direct multiplicative gate (§3.6).
- **C4** manifest `score_breakdown` aligned to `{importance, relevance, recency,
  m_v}`; spurious `access` key removed (§5.3).
- **C5** `contradicts` stays a v2 relation *edge kind*, but v1 contradiction
  *detection* records `queue_review` in the manifest without writing the edge
  (§3.4, §5.2, §12.1).

**Dreaming / worker / jobs:**
- **C6** S-05 prompt-injection enforcement path now specified end-to-end:
  versioned dream-output JSON schema, orchestrator validates the manifest
  pre-merge, failure → job FAILED (§5.2, §5.5, §8.3). *(Spike 1b gave live
  evidence: the LLM substrate already rejects schema-noncompliant output.)*
- **C7** job-claim transition fixed: atomic `QUEUED→SPAWNED` + pid; `→RUNNING` on
  first heartbeat (§5.4).
- **C10** `trigger.cumulative_importance` unit defined: sum of `importance`
  (0–1) over unconsumed staging for the dreaming-memory's scope (§5.1).
- **C11** v1 `dream.configure` rejects `mode: cross-agent` → `invalid-config`
  (§6.3).
- **C12** dual-OCC interaction clarified: dream-merge OCC uses manifest
  `base_versions`, never retries → review queue; the advisory lock is
  real-time-write-only (§5.4, §7.2).

**Plugins / startup / observability:**
- **C13** LLM init demoted from FATAL to degraded at boot — daemon serves
  get/list/history; dreaming unavailable; dream jobs → `plugin-unavailable`
  (§2.3, §9.9).
- **C14** AppLog gains `mcp_denied` event type so `engram log --denied` has a
  source (§7.3, §10.2).
- **C8 / OQ-L** AppLog is NOT git-tracked — it is its own SQLite store inside the
  git-ignored `.engram/` (§7.3, §12.2).
- **C9** undefined-but-referenced CLI verbs (`engram lifecycle revive`,
  `engram governance scrub`, `engram repair`) defined in §10.2.

**Design gaps closed:**
- **G1** `PreCompact` is NOT fire-and-forget — memory re-injection uses a
  request/response hook that returns context; capture remains fire-and-forget
  for the other hooks (§6.1).
- **G2** per-agent MAC secret provisioning specified: `0600` file at
  `~/.engram/agent-secrets/<agent-id>.mac`, daemon-written, read by the hook at
  invocation, never in env (§6.1, §8.3).

**Spike findings folded (see `SPIKES.md`):**
- QMD confirmed as in-process TS library (`@tobilu/qmd@2.5.2`, ESM) — no
  subprocess fallback (§11.1; OQ on QMD transport closed).
- MCP TS SDK (`@modelcontextprotocol/sdk@1.29`) confirmed: multi-session
  Streamable HTTP + built-in bearer middleware + resource subscriptions (§6.3).
- Vercel AI SDK substrate confirmed; structured-output-capable model required
  per provider; worker validates against schema (§11.1, C6).
- **GraphPlugin redefined** to "rebuildable derived index" semantics:
  graphify is extract→`graph.json`→query with no live edge mutation;
  `ingestEdges`/`removeNode` become batch rebuild/update, not incremental ops.
  Ollama is a hard Phase-7 prerequisite (§2.3, §6.2).

**Open questions closed:** OQ-D (`~/.engram/` global, `<repo>/.engram/`
project), OQ-G (raw/ git-ignored default, LFS opt-in), OQ-L (AppLog not
git-tracked).

---

## 0. Why this exists & v1 audit trail

The prior effort (`plans/memory-overhaul/`) was **scrapped** — it was a
config-dir transplant, not a standalone system, and it locked an archived
dependency (Kuzu). engram is the redesign.

**SPEC v1** (2026-05-22 draft, replaced by this document) established the
architectural shape. It was then audited by four parallel specialist reviews
plus four research reports. The reviews **affirmed the architecture** and
supplied four missing contracts; the research **validated the tool choices**
and surfaced five corrections to v1's memory model. SPEC v2 is the merged
output. See `docs/_history/round-1-2/SYNTHESIS.md` for the decision trail
(D-1…D-7, R-1…R-5) and how each finding lands in this document.

---

## 1. Vision & Scope

**engram** is a background application — `engramd` daemon + MCP server +
CLI + (v2) dashboard — that stores agent memory as Markdown files, retrieves
it with hybrid scored search, and runs a decoupled *dreaming* process to
consolidate, connect, and learn from it.

### 1.1 What it is NOT
- **Not a wiki.** A hand-authored knowledge base is just *one origin* of memory.
- **Not Obsidian-centric.** Memories are Markdown files; Obsidian *can* open
  the folder, but engram does not require it.
- **Not a fork of agentmemory.** `rohitg00/agentmemory` is studied as a
  reference for *patterns* (hooks, 4-tier model, RRF, privacy filter, viewer)
  but engram shares no code with it and takes no dependency on its iii engine.
- **Not an agent config.** It is a standalone product the user installs.

### 1.2 v1 scope vs v2

| Capability | v1 (headless) | v2 |
|------------|:-------------:|:--:|
| Core kernel (store, scoring, versioning, plugin host) | ✅ | |
| Markdown memory store + 4 memory types | ✅ | |
| Retrieval plugin (QMD) | ✅ | |
| Graph plugin (graphify, subprocess) | ✅ | |
| MCP server — Streamable HTTP on `127.0.0.1`, bearer-token gated | ✅ | |
| CLI | ✅ | |
| Capture plugin (Claude Code hooks → staging) | ✅ | |
| Dreaming worker + dreaming-memory objects | ✅ | |
| **Confidence-weighted retrieval** (R-1) | ✅ | |
| **Per-type decay rates** (R-2) | ✅ | |
| **Episodic immutability + counterfactual gate** (R-4) | ✅ | |
| **Absolute active-pool floor** (R-5) | ✅ | |
| **Threat model + 6 CRITICAL mitigations** (D-6) | ✅ | |
| **§9 Failure & Safety; §10 Observability** | ✅ | |
| Cross-agent dreaming | spec'd (D-3) | ✅ |
| Capture plugins for Codex / Copilot / generic | | ✅ |
| Dashboard (graph view, dreaming visualization, review queue UI) | spec'd | ✅ |
| App-log hash-chained tamper-evidence | | ✅ |
| Team / multi-machine sync | | ✅ |

---

## 2. Architecture — Microkernel + Detached Dreaming Worker

### 2.1 Topology

```
┌──────────────────────────────────────────────────────────────┐
│  CORE KERNEL  —  always-up daemon process (engramd)           │
│                                                               │
│  FIXED CORE (not swappable — engram's identity):              │
│   • Markdown Store          files = truth, typed memories     │
│   • Scoring Engine          (I × Relevance × Recency) × m_v   │
│   • GitVersioning           store-wide history (opt-in)       │
│   • AppLog (SQLite)         per-field provenance (always-on)  │
│   • AccessControl           agent identity, bearer-token gate │
│   • Dreaming Orchestrator   queue + schedule + merge          │
│   • CaptureIntake           privacy filter, fail-closed        │
│   • Plugin Host             two transports: in-proc | subproc │
│   • CoreService (facade)    transport-agnostic application    │
│   • API Surface             MCP (Streamable HTTP) + CLI       │
│                                                               │
│  PLUGIN SEAMS (one v1 implementation each):                   │
│   ┌────────────┬────────────┬────────────┬─────────────────┐ │
│   │ Retrieval  │ Graph      │ Capture    │ LLM (Vercel AI  │ │
│   │ → QMD lib  │ → graphify │ → CC hooks │   SDK substrate)│ │
│   │  (TS,      │  (Python,  │ (install + │   Claude /      │ │
│   │   in-proc) │  subproc)  │  normalise)│   OpenAI /      │ │
│   │            │            │            │   Ollama        │ │
│   └────────────┴────────────┴────────────┴─────────────────┘ │
└──────────────────────────────────────────────────────────────┘
              ↕ shared store + SQLite jobs queue
┌──────────────────────────────────────────────────────────────┐
│  WORKER PROCESS — detached, spawned per job (dream | ingest)  │
│   • All LLM work happens here; orchestrator owns no LLM       │
│   • Writes a git branch + manifest.json; exits                │
│   • Crash/overrun NEVER touches the core daemon               │
└──────────────────────────────────────────────────────────────┘
```

### 2.2 Why this shape (rationale recap)

- **Microkernel** — four boundaries with real plurality day one: LLM
  (Claude/OpenAI/Ollama — 3), capture (Claude Code first, more later),
  retrieval (QMD now — v0.9 risk = insurance), graph (graphify now). Seams
  are not speculative.
- **Detached worker process** — honors *"decoupled memory… independent
  process… not affecting the main agent."* A heavy LLM dream cannot stall
  `memory.recall`. Crash isolation is the only honest guarantee.
- **CoreService facade** — transport-agnostic application layer; MCP, CLI,
  and the v2 dashboard are thin adapters over it. Prevents logic drift.
- **CaptureIntake fixed** — privacy filter is security-critical and must not
  live in a swappable plugin.

### 2.3 Plugin contract (closes OQ-A: one contract, two transports)

```ts
const PLUGIN_CONTRACT_VERSION = "1.0";

type PluginKind  = "retrieval" | "graph" | "capture" | "llm";
type Transport   = "in-process" | "subprocess";

interface PluginManifest {
  kind: PluginKind;
  name: string;
  contractVersion: string;       // semver; checked at init
  transport: Transport;
  config: Record<string, unknown>;
}

type PluginErrorKind =
  | "unavailable"   // engine down — core may degrade or retry
  | "invalid-input" // caller bug — do not retry
  | "transient"     // timeout / rate-limit — retry with backoff
  | "fatal";        // contract/version mismatch — host unloads
interface PluginError { kind: PluginErrorKind; message: string; retryable: boolean; }

interface PluginLifecycle {
  init(manifest: PluginManifest): Promise<void>;   // idempotent
  health(): Promise<{ ok: boolean; detail?: string }>;
  shutdown(): Promise<void>;
}

interface MemoryRef { id: MemoryId; path: string; frontmatter: Frontmatter; }

interface RetrievalPlugin extends PluginLifecycle {
  index(ref: MemoryRef): Promise<void>;             // non-fatal on failure
  update(ref: MemoryRef): Promise<void>;
  deindex(id: MemoryId): Promise<void>;
  search(query: string, opts: SearchOpts): Promise<ScoredHit[]>;
  rebuild(store: StorePath): Promise<void>;
}

interface GraphPlugin extends PluginLifecycle {
  // graphify is a rebuildable DERIVED INDEX, not a mutable graph DB (Spike 3, §6.2):
  ingestEdges(refs: MemoryRef[]): Promise<void>; // batch update/re-extract, NOT live edge write
  removeNode(id: MemoryId): Promise<void>;        // rebuild/filter after file delete, NOT live delete
  traverse(from: MemoryId[], opts: TraverseOpts): Promise<GraphResult>; // graphify MCP stdio query tools
  extract(rawPath: string): Promise<ExtractedStructure>;   // CLI `graphify extract` (Ollama)
  rebuild(store: StorePath): Promise<void>;
}

interface CapturePlugin extends PluginLifecycle {       // F-2.4 redefinition
  harness: string;
  install(target: HookInstallTarget): Promise<void>;    // writes hook scripts
  uninstall(target: HookInstallTarget): Promise<void>;
  normalise(raw: unknown): RawObservation;              // wire payload → typed
}
// CC hook scripts POST to engramd's CaptureIntake endpoint (Streamable HTTP).

interface LlmPlugin extends PluginLifecycle {
  complete(prompt: Prompt, opts: LlmOpts): Promise<LlmResult>;
  embed(texts: string[]): Promise<Float32Array[]>;
}
```

**Contract rules:**
1. **Files are truth.** `index`/`update`/`deindex`/`ingestEdges` failures are
   non-fatal: core logs, enqueues retry, the write still succeeds. `rebuild()`
   always recovers correctness.
2. **`search`/`complete`/`embed`/`traverse` failures *are* surfaced** as
   degraded results with an explicit `partial: true` flag — recall never
   silently returns fewer hits.
3. **Health probed on a schedule;** `unavailable` plugin → restart (subproc)
   or marked degraded (in-proc).
4. **`contractVersion` checked at `init`** — mismatch is `fatal`.
5. **All plugins optional to the kernel's boot.** Core starts and serves
   `memory.get`/`list`/`history` from files; recall degrades, dreaming
   unavailable, until plugins are up.

### 2.4 Two transports, one contract

- **in-process** — direct TS module calls. Used by QMD (TS) and the LLM
  plugins.
- **subprocess** — JSON-RPC over stdio. Used by graphify (Python) and any
  future non-TS plugin. Same interface; only the host's invocation differs.

This closes OQ-A cleanly: a replacement engine in any language drops in
behind the same JSON-RPC shape — no FFI.

---

## 3. The Memory Model

### 3.1 Conceptual foundation

The four memory types (Semantic / Episodic / Procedural / Contextual) +
Generative-Agents scoring (Importance × Relevance × Recency) + forgetting ≠
deletion. SPEC v1 had the shape; v2 makes it correct against the literature.

| Type | Holds | Decay default (λ/hour) | Half-life | Notes |
|------|-------|------------------------|-----------|-------|
| **contextual** | current working state | **0.05** | ~14 h | hard-transitions to `dormant` at SessionEnd unless dreaming promotes |
| **episodic** | what happened — events | **0.002** | ~21 d | **immutable during dreaming** (R-4); only consolidated *into* derivatives |
| **semantic** | what is known — facts | **0.0002** | ~200 d | dreaming product; reinforced by access |
| **procedural** | how to do things — patterns | **0.0001** | ~years | dreaming product; near-permanent once corroborated |

**Modulation by importance** (FadeMem): `λ_eff = λ_base · (1 − importance · 0.7)`
— important memories decay slower.

### 3.2 Markdown files vs raw material

- **`raw/`** — sources (videos, PDFs, books, mp3, raw notes). Immutable,
  git-ignored by default.
- **`memories/`** — typed Markdown distilled from raw, or captured from
  sessions, or written by humans/agents/LLM. **Memories are the unit of
  versioning, scoring, access control, retrieval.**

A memory carries `sources: [raw/...]` pointing back. This is the
graphify+book-to-skill pattern generalized.

### 3.3 Store layout

```
<store>/                        # global ~/.engram/  |  project <repo>/.engram/
  raw/                          # immutable sources (git-LFS or ignored)
  staging/                      # capture observations as per-session JSONL
    <agent-id>/<session-id>.jsonl     # append-only; one log per session
  memories/                     # curated typed Markdown — the real store
    semantic/  episodic/  procedural/  contextual/
  .dreaming/                    # dreaming-memory configs + branch metadata
  .engram/                      # SQLite stores + state
    app-log.db                  # AppLog (per-field provenance)
    jobs.db                     # job queue (dream + ingest)
    stats.db                    # recency/access touches (NOT versioned)
    index-state.db              # per-memory last_reindexed
    quarantine/                 # files with broken frontmatter
    schema-version              # store schema version string
    engramd.pid                 # PID lock
    applog-recovery.jsonl       # spool for failed app-log writes
    git-pending.jsonl           # spool for failed git commits
    capture-fallback/           # emergency capture buffer (10MB / 7d)
    agent-reports/              # dream/ingest run audit JSON
  .git/                         # store-wide git history (opt-in)
```

### 3.4 Frontmatter schema (the contract)

```yaml
---
id: mem_01HX...                 # stable ULID; links resolve by id, never path
type: semantic                  # semantic | episodic | procedural | contextual
origin: ingested                # ingested | agent-session | self-authored | human
category: knowledge-base        # free-form domain
title: "JWT auth in Project X"
summary: "One-line preview for index/recall"
tags: [auth, project-x]         # human- or LLM-authored

relations:                      # v1 limits to 2 EDGE KINDS (D-7)
  - {to: mem_abc, kind: derived_from}    # provenance backlinks
  - {to: mem_xyz, kind: related_to}      # everything else
# v2 adds edge kinds: extends, contradicts, uses, replaces
# NOTE (C5): v1 dreaming DETECTS contradictions (§5.2) but records them as
# manifest `queue_review` items — it does NOT write a `contradicts` edge.
# The `contradicts` edge kind itself is v2.

# --- scoring ---
importance: 0.72                # 0–1, LLM-rated at creation, re-weighted by dreaming
# recency + access_count → STATS SIDECAR (.engram/stats.db), NOT frontmatter
# relevance computed per-query (QMD), never stored

# --- confidence + verification (R-1, v1) ---
confidence: 0.80                # 0–1; multiplicatively gates retrieval score
verification_state: unverified  # unverified | self-supported | tool-supported | human-verified
# m_v defaults: human=1.0 | ingested=0.9 | agent-session=0.85 | self-authored=0.7

# --- lifecycle ---
lifecycle: active               # active | dormant | archived  (never deleted)

# --- provenance / access ---
created_by: agent:claude-code   # immutable (OT.5)
last_modified_by: dream:claude-code-work:drun_01HZ...
created: 2026-05-22T08:00:00Z
updated: 2026-05-22T08:30:00Z
scope: agent:claude-code        # owning scope
visibility: shared              # shared | private | hidden
session: sess_01HX...           # set for session-scoped memory

sources: ["raw/architecting-agent-memory.pdf"]   # path-jailed under raw/
ingest_run_id: irun_01HY...     # if origin = ingested
staged_from: [stage_01H...]     # if distilled from staging

version: 7                      # OCC token — incremented per write
---
<body — the memory content, with [[mem_id]] inline links>
```

### 3.5 Identity & filenames (per-origin)
- Stable `id` (ULID) in frontmatter. Links resolve by `id`, never path — so
  dreaming can rename/move/reorganize files freely.
- **Filename rule:**
  - `origin ∈ {human, ingested, self-authored}` → readable slug
    (`semantic/jwt-auth-project-x.md`).
  - `origin = agent-session` → id-based (`episodic/mem_01HX....md`).

### 3.6 Scoring & forgetting (with confidence — R-1)

**Retrieval rank:**
```
rank = ( α_r · recency_norm
       + α_i · importance_norm
       + α_rel · relevance_norm ) × m_v(origin, confidence)
```
- All three components **min-max normalized per query** across the candidate
  set (not global).
- α defaults to 1, all equal (Generative Agents), configurable per dreaming
  memory.
- `recency_norm = exp(−λ_eff · hours_since_last_used)` from the stats
  sidecar; `λ_eff = λ_base · (1 − importance · 0.7)`.

**Importance enters the score twice — by design (C2).** It is an additive term
(`importance_norm`) *and* it modulates decay (`λ_eff`, so important memories stay
fresh longer). This coupling is intentional emphasis (FadeMem + Generative
Agents), and bounded: the decay modulation factor is capped at `0.7`, so a
maximally important memory decays at 30% of base rate, not zero. Documented here
so the interaction is not mistaken for a bug; `α_i` can be lowered per dreaming
memory if the additive term over-weights.

**`m_v` definition (C1, C3).** `m_v` is a per-origin base trust multiplier,
*gated by* `confidence`:

```
m_v(origin, confidence) = base_m_v(origin) · (confidence / default_confidence(origin))
```

so a memory at its origin's default confidence gets exactly `base_m_v`; lower
confidence scales it down, higher scales it up. The result is **clamped to
`[0.1, 1.0]`** (1.0 = the human/verified ceiling; 0.1 floor keeps a low-confidence
memory retrievable, never zeroed).

  | origin | default `confidence` | `base_m_v` |
  |--------|----------------------|------------|
  | human | 0.90 | **1.0** |
  | ingested | 0.80 | **0.9** |
  | agent-session (captured) | 0.70 | **0.85** |
  | self-authored (dreaming) | 0.50 | **0.7** |

- A counterfactual-gated procedural memory written at `confidence: 0.3` (§5.2.4,
  self-authored, default 0.5) therefore gets `m_v = 0.7 · (0.3/0.5) = 0.42` — it
  is retrievable but ranks well below corroborated memory, exactly as intended.
- `verification_state: human-verified` (via `engram memory confirm <id>`)
  **floors** `m_v` to **≥ 0.95** — it raises low-trust origins to high confidence
  and **never lowers** an origin already above 0.95 (C1: human-origin stays
  1.0). Confirmation can only help a memory's rank, never hurt it.

**Importance at creation** — LLM-rated 1–10 ("poignancy"), normalized /10.
Re-weighted by dreaming based on connectedness.

**Forgetting = lifecycle transition, never deletion** (`forget` →
dormant→archived). Dormancy triggered only when composite score is below
threshold for **≥2 consecutive dreaming runs** (R-2; single-sample is noisy).

**Hard-delete** exists only as audited `governance_delete` — secrets, GDPR.

---

## 4. Information Flow (canonical — F-1.1)

Five canonical flows. Each has a triggering event and a terminal state.

### 4.A Capture → staging → dreaming → memories
```
[CC hook fires]
  → CapturePlugin.normalise(raw_payload)
  → POST /capture-intake  (Streamable HTTP, bearer token, 200ms hard timeout)
  → CaptureIntake: privacy filter (multi-layer, fail-closed)
  → append staging/<agent>/<session>.jsonl  (atomic, fsync, exit 0)
... later, on dream trigger ...
  → worker reads staging/<scope>/*.jsonl
  → LLM distill → candidate typed memories with derived_from backlinks
  → writes dream branch + manifest.json (per-change classification)
  → records consumed staging offset watermark
  → exits
  → orchestrator: deterministic safe/gated classification
  → auto-merge additive hunks; queue gated hunks
  → ON MERGE COMMIT: truncate staging logs up to watermark
                     ← staging retained until durable merge (no data loss
                       on rejected dream — F-1.2)
```

### 4.B Ingest (raw → memories) — worker job
```
[memory.ingest(rawPath)  OR  raw/ debounced watch]
  → core path-jails rawPath under <store>/raw/ (S-06)
  → enqueues ingest job on jobs.db
  → worker: GraphPlugin.extract(rawPath) + LLM distill → memories with sources:
  → writes branch + manifest; exits
  → orchestrator merges (ingest output additive → auto-safe)
```
Ingest is a worker job, not core work (F-4.4).

### 4.C Explicit remember
```
[memory.remember(payload)]  OR  [worker self-authored note]
  → CoreService.write:
     ↳ access-control: caller cap + scope/visibility
     ↳ schema-validate frontmatter
     ↳ OCC: capture version (if update); short-lived advisory lock at commit time
     ↳ atomic file write (.tmp → fsync → rename)
     ↳ AppLog append (spool to applog-recovery.jsonl on failure)
     ↳ git commit (spool to git-pending.jsonl on failure)
     ↳ async enqueue RetrievalPlugin.index + GraphPlugin.ingestEdges
  → return {id, version}
```

### 4.D Recall (resolves F-1.4: QMD primary, graph expansion, core fuses)
```
[memory.recall(query, opts)]
  → access-control → readable scope set
  → RetrievalPlugin.search(query, {scopeFilter, limit})  → ScoredHit[]
                                                            (Relevance only)
  → [opts.graphExpand]: GraphPlugin.traverse from top-k → 1-hop neighbours
  → core scoring engine:                                  ← scoring NEVER imports
       rank = f(importance_norm, relevance_norm, recency_norm) × m_v
                                                            the retrieval plugin
  → min-max normalize per-query
  → filter lifecycle != active unless opts.includeDormant
  → return id+summary[]+score (bodies only if opts.expand=true)
  → recency/access touch → stats.db (sidecar, async, NOT git, NOT app-log)
```
**Resolution:** QMD provides Relevance; the core scoring engine fuses
I × R × Recency and applies the m_v gate. Graph is opt-in expansion (adds
recall candidates), not a competing rank. No RRF needed — single ranked source.

### 4.E Dream job lifecycle — see §5.4 state machine.

---

## 5. Dreaming

### 5.1 "Dreaming memory" — first-class persistent object

`.dreaming/<name>.md`:
```yaml
name: claude-code-work
scope: [agent:claude-code]      # which agents/sessions/memories it owns
mode: per-agent                 # per-agent (v1) | cross-agent (v2 — D-3)
llm: claude                     # LlmPlugin provider
schedule: [session-end, nightly]
budget: {max_tokens_per_run: 200000, max_daily_usd: 5, max_monthly_usd: 50}
merge_policy: auto-safe         # auto-safe | always-auto | always-gated
trigger:
  cumulative_importance: 50     # R-2 — fire when the SUM of `importance` (0–1
                                #   each) over this dreaming-memory's unconsumed
                                #   staging observations reaches this value (C10).
                                #   ~50 ≈ tens of high-importance observations.
```

- Multiple coexist, each owns a slice.
- **Per-agent isolated (v1).** Cross-agent deferred to v2 (D-3) — keeps the
  field, no implementation.
- **Config file is daemon-owned** (S-10) — written only by the daemon or
  governance CLI, never by agent MCP calls.

### 5.2 What a dreaming run does (worker-side)

Runs as a detached process; writes to a `dream/<name>/<ts>` git branch.

1. **Distill** — staging observations → curated typed memories. **Episodic
   sources are read-only** (R-4 / MemMachine). Derivatives carry
   `relations: [{to: source_id, kind: derived_from}]`.
2. **Connect** — discover latent relations, insert `[[links]]` (`related_to` /
   `derived_from` edges only in v1). Detect emergent entities (≥3 mentions, no
   dedicated memory).
   **Two-layer contradiction detection** (R-4, C5): semantic
   (`sim>0.85 ∧ |Δimportance|>0.3`) + graph traversal. On hit: manifest entry
   `{kind: contradiction, action: queue_review}`, never auto-resolve. **v1 does
   NOT write a `contradicts` edge** (that edge kind is v2) — the contradiction
   lives only as a manifest review item until a human resolves it.
3. **Re-weight** — adjust `importance`; advance lifecycle.
   **Importance floor 0.05** (worker may not set below).
   **Per-run rate limits**: max `min(50, 5%)` active→dormant, max
   `min(20, 2%)` dormant→archived (failure-safety §8).
4. **Verify & learn** — analyze captured failure traces + user corrections.
   **Counterfactual validation gate** (R-4 / ADM): a procedural memory may
   only be promoted to `active` if corroborated by ≥1 additional independent
   episode in the same staging batch; otherwise written `confidence: 0.3`,
   `verification_state: unverified`, `lifecycle: dormant`, awaiting human
   promotion via `engram memory confirm`.
5. **Commit** to branch + write `manifest.json`. Exit.

**Worker output is schema-validated (C6 / S-05 enforcement path).** The worker
never emits free-form text into memory. Every candidate memory and every
`manifest.json` change record is produced as **structured output validated
against a versioned schema** `src/schemas/dream-output.schema.json`
(`schema_version` embedded in the manifest). Enforcement is two-stage:
1. **Worker-side:** the LlmPlugin call uses structured output (Zod/JSON-schema);
   any field the memory *body* tried to inject as frontmatter is not in the
   schema and is dropped. *(Spike 1b confirmed the substrate rejects
   schema-noncompliant model output rather than passing it through.)*
2. **Orchestrator-side:** before classification/merge, the orchestrator
   re-validates the entire manifest against the schema. **Validation failure →
   job `FAILED`, branch preserved, no merge attempted** (§5.4, §9.4 check 0).

This is the concrete code path behind the S-05 mitigation: a prompt-injection in
a memory body cannot mutate frontmatter because the injected fields never pass
schema validation at either stage.

### 5.3 Manifest format (worker output)

Per-change records (consumed by the orchestrator):
```json
{
  "run_id": "drun_01HZ...",
  "dream_name": "claude-code-work",
  "base_versions": {"mem_01HX...": 7, "mem_01HY...": 3},
  "changes": [
    { "kind": "new", "id": "mem_NEW...", "memory_type": "semantic" },
    { "kind": "modify", "id": "mem_01HX...",
      "field_diffs": [{"field":"importance","from":0.4,"to":0.72,
                       "reason":"connected to 5 other memories"}] },
    { "kind": "lifecycle", "id": "mem_01HY...",
      "transition": "active→dormant",
      "score_breakdown": {"importance":0.05,"relevance":0.0,"recency":0.02,"m_v":0.7,"total":0.04} },
    { "kind": "contradiction", "ids": ["mem_a","mem_b"], "action": "queue_review" }
  ],
  "staging_watermark": {"sess_01HX...": 4823},
  "tokens_in": 18204, "tokens_out": 3120, "cost_usd": 0.41
}
```

### 5.4 Job state machine (closes F-1.5; persisted in `.engram/jobs.db`)

```sql
CREATE TABLE jobs (
  id            TEXT PRIMARY KEY,         -- ULID
  kind          TEXT NOT NULL,            -- 'dream' | 'ingest'
  target        TEXT NOT NULL,            -- dreaming-memory name | raw path
  state         TEXT NOT NULL,
  pid           INTEGER,
  heartbeat_at  TEXT,
  branch        TEXT,
  manifest      TEXT,
  attempts      INTEGER NOT NULL DEFAULT 0,
  error         TEXT,
  created_at    TEXT NOT NULL,
  updated_at    TEXT NOT NULL
);
```

```
QUEUED ── (orchestrator claims, transactional) ──▶ SPAWNED
SPAWNED ── (worker starts heartbeating) ─────────▶ RUNNING
RUNNING ── (worker exit 0 + branch + manifest) ──▶ MERGING
RUNNING ── (worker exit ≠ 0 OR pid dead) ─────────▶ FAILED
RUNNING ── (heartbeat stale > 5 min) ─────────────▶ TIMED_OUT
RUNNING ── (budget exhausted, partial checkpoint) ▶ ABORTED
MERGING ── (orchestrator merge ok) ────────────────▶ MERGED
MERGING ── (gated hunks pending review) ───────────▶ REVIEW_PENDING
REVIEW_PENDING ── (engram dream review --approve) ─▶ MERGED
TIMED_OUT/FAILED/ABORTED ── (engram dream resume) ─▶ QUEUED  (idempotent re-run)
ANY ── (engram dream rollback) ────────────────────▶ ROLLED_BACK  (git revert)
```

**Rules:**
1. **Atomic claim (C7)** — `QUEUED→SPAWNED` + `pid` write in one SQLite
   transaction (no double-spawn); the worker advances `SPAWNED→RUNNING` on its
   first heartbeat. The startup crash-recovery scan (§9.9 step 2) reaps any
   `SPAWNED` *or* `RUNNING` job whose `pid` is dead → `FAILED`, so a crash in the
   spawn→first-heartbeat window is handled.
2. **Heartbeat 30 s.** Watchdog reaps stale (>5 min) → `TIMED_OUT`.
3. **Idempotent retry.** Re-spawn reads checkpoint, skips completed stages;
   staging only truncated on `MERGED` → no observation loss.
4. **Single-flight per dreaming-memory** — only one non-terminal job per
   name; `dream.trigger` returns `already-running`.
5. **Merge OCC** (F-5.3, C12) — orchestrator re-validates each hunk against the
   `base_versions` recorded in the manifest at job start. This is a **distinct
   OCC path** from the real-time advisory lock (§7.2): the advisory lock guards
   live `memory.update` writes only; dream-merge OCC uses manifest base_versions
   and **never retries** — a base-version mismatch (an agent wrote the memory
   since the job started) routes the hunk to the review queue. Field-disjoint
   changes get **three-way merge**; field-overlap → review queue.
6. **Merge validation** (failure-safety §3): YAML parse OK; version tokens
   not regressed; lifecycle transitions within rate limits; importance ≥
   0.05; **absolute active-pool floor** check (R-5): if merge would leave
   fewer than `min(100, 20% of total)` active memories, gate unconditionally
   regardless of `merge_policy`.
7. **Dream branches** retained 7 days post-merge (rollback window); FAILED
   branches retained 90 days (forensics).

### 5.5 Safe-vs-gated classification — deterministic (S-11)

NOT LLM-decided. Daemon-side predicate over the manifest diff:

| Change kind | Default |
|-------------|---------|
| new file (validates against dream-output schema — C6) | **auto-safe** |
| new link / `relations` addition (`related_to`/`derived_from` in v1) | **auto-safe** |
| importance change `|Δ| < 0.2` | **auto-safe** |
| importance change `|Δ| ≥ 0.2` | **gated** |
| lifecycle transition | **gated** |
| visibility / scope change | **gated** |
| contradiction → queue_review | **gated** |
| delete (governance only) | **gated** |
| body modification of episodic | **REJECTED** (R-4 immutability) |

Override: `merge_policy: always-gated` (everything reviewed) or
`always-auto` (still subject to active-pool floor and rate limits).

### 5.6 Triggers
`SessionEnd` hook · cron (nightly/scheduled) · MCP `dream.trigger` ·
cumulative-importance threshold (R-2, §5.1 config) · dashboard button (v2).

### 5.7 Decoupling guarantees
Separate OS process; own budget; own git branch. A dream crash/overrun
**never** touches the core daemon or the main agent's session.

---

## 6. Capture, Ingest & the MCP Contract

### 6.1 Capture (per-harness plugin)

Claude Code first. Two hook classes (G1):

- **Capture hooks — fire-and-forget:** `PostToolUse`, `PostToolUseFailure`,
  `Stop`, `SessionEnd`, `UserPromptSubmit`, `SubagentStart/Stop`. These only
  *push* observations; they never need a response.
- **Re-inject hook — request/response:** `PreCompact` is **NOT** fire-and-forget.
  Its job is to *return* memory context to the host before compaction, so it
  performs a synchronous `memory.recall` (bounded timeout, e.g. 1s) and returns
  the result as hook output. It does not write to staging. If engramd is slow or
  down, it returns empty (host proceeds with normal compaction). This is a
  separate code path from the fire-and-forget capture below.

Each **capture** hook is fire-and-forget:

- Hook process posts a normalised payload to engramd's `/capture-intake`
  endpoint (Streamable HTTP + bearer token).
- **200ms hard timeout** — host session never stalls.
- On any error/timeout: exit 0 silently (host unaffected).
- **CaptureIntake** (kernel) applies the **multi-layer privacy filter** (S-01):
  1. Known-pattern regex (AWS, GitHub, JWT, PEM, etc.).
  2. Entropy-based detection (high-entropy strings ≥ N chars).
  3. Path blocklist (`~/.ssh/`, `~/.aws/`, `/run/secrets/`, etc.).
  4. Optional LLM sweep (opt-in).
- **Fail-closed**: filter failure ⇒ drop observation. Log at ERROR (without
  payload content). `filter.audit_log` records what was stripped.
- Filtered observation appended to `staging/<agent>/<session>.jsonl`
  (per-session append-only log — F-1.2).
- **Backpressure** (failure-safety §4): staging cap default 500 MB / 10 000
  files. Excess → drop new + log + emergency dream queued.
- **engramd down**: hook writes to local `capture-fallback/` (10 MB / 7 d).
  Daemon drains it on next start (S-09 boundary — fallback entries are
  re-filtered before staging).
- **MAC secret provisioning (G2 / S-02):** the per-agent MAC secret used to
  attest staging entries is **not** passed via environment (env leaks through
  `/proc/<pid>/environ`). The daemon writes it to a `0600` file at
  `~/.engram/agent-secrets/<agent-id>.mac` (daemon-owned) at `memory.init` /
  `engram agent add` time. The capture hook reads this file at invocation,
  computes the MAC over the normalised payload, and discards the secret from
  memory on exit. The secret is rotated on `engram agent rotate <id>` and
  revoked with the agent token.

### 6.2 Ingest

Worker job. `memory.ingest(rawPath)` enqueues; the worker calls
`GraphPlugin.extract` (graphify) + `LlmPlugin.complete` to distill typed
memories with `sources:` provenance and LLM-built `tags`.

**Path safety**: `rawPath` is `path.resolve()`d and asserted to start with
`<storeRoot>/raw/` (S-06). graphify subprocess receives only post-jail paths
(S-16 — argument arrays, never shell strings).

**Markdown extraction caveat** (research finding): graphify routes Markdown
to the LLM. **Ollama backend is a hard Phase-7 prerequisite** (not just a
mitigation) — `graphify extract --backend ollama --max-concurrency 1` keeps cost
local and removes the network dependency (Spike 3 confirmed the flag).

**GraphPlugin = rebuildable derived index, NOT a mutable graph DB (Spike 3).**
graphify is fundamentally `extract → graph.json → query`; its MCP stdio server
(`python -m graphify.serve`) exposes **query-only** tools (`query_graph`,
`get_neighbors`, `shortest_path`, `get_community`, `god_nodes`, `graph_stats`)
and there is **no live `add_node`/`remove_node`/edge-write op**. The GraphPlugin
contract (§2.3) maps as:
- `extract(rawPath)` → CLI `graphify extract` (worker job, Ollama backend).
- `traverse(from, opts)` → MCP stdio query tools.
- `rebuild(store)` → CLI `extract`/`update` over `memories/`.
- `ingestEdges(refs)` → **batch update**: CLI `graphify update` (AST, no-LLM,
  fast) or a scheduled/post-dream re-extract of the affected subset. The graph
  is refreshed as a derived index, not mutated per-write — exactly like QMD
  (index failures are non-fatal; `rebuild()` always recovers correctness, §2.3
  rule 1).
- `removeNode(id)` → governance-cascade rebuild (or `graph.json` filter) after
  the memory file is deleted (§8.5), not a live delete.

This is consistent with "files are truth, indexes are derived & rebuildable" —
it simply states it for the graph the way it was already stated for retrieval.

### 6.3 MCP contract — Streamable HTTP, bearer-gated (R-3)

**Transport:** Streamable HTTP bound to `127.0.0.1` only (DNS-rebinding
protection per MCP spec). Bearer token on every call:
```
Authorization: Bearer <agentToken>
```

**Token issuance:** `memory.init` (or `engram agent add`) mints an opaque
token; daemon stores it 0600, owned by the daemon process. Maps to
`(agentId, capabilities, scope)`. Revocable via `engram agent revoke <id>`.

**Capabilities (5):**

| Capability | Grants |
|------------|--------|
| `read` | `recall`, `get`, `list`, `history` within readable scope |
| `write` | `remember`, `update`, `forget`, `ingest` |
| `dream` | `dream.trigger`, `dream.configure` for owned dreaming-memories |
| `cross-agent` | recall/dream across scopes beyond caller's own (v2) |
| `govern` | `governance_delete`, `init`, agent-token management |

Default session agent: `{read, write, dream}` scoped to itself.

**Verb table (16 verbs):**

| Verb | Params | Returns | Errors | Cap |
|------|--------|---------|--------|-----|
| `memory.init` | `{scope: "global"\|"project", path?}` | `{storePath, agentToken}` | `already-exists`, `path-denied` | `govern` |
| `memory.remember` | `{type, category?, title, body, tags?, scope?, visibility?, sources?, relations?, confidence?}` | `{id, version}` | `invalid-schema`, `scope-denied` | `write` |
| `memory.update` | `{id, version, patch}` | `{id, version}` | `version-conflict`, `not-found`, `scope-denied` | `write` |
| `memory.recall` | `{query, limit?, type?, category?, includeDormant?, graphExpand?, expand?}` | `{hits:[{id, summary, score:{importance,relevance,recency,m_v,total}, body?}], partial, degraded?}` | `retrieval-unavailable` | `read` |
| `memory.get` | `{id, expand?}` | `{memory}` | `not-found`, `scope-denied` | `read` |
| `memory.list` | `{type?, category?, scope?, lifecycle?, limit?, cursor?}` | `{refs:[MemoryRef], cursor?}` | — | `read` |
| `memory.forget` | `{id, version}` | `{id, lifecycle}` | `version-conflict`, `not-found` | `write` |
| `memory.ingest` | `{rawPath}` | `{jobId}` (async) | `not-found`, `unsupported-type`, `path-denied` | `write` |
| `memory.history` | `{id}` | `{events:[{ts, agent, field, old, new, reason, dream_run_id?}]}` | `not-found` | `read` |
| `memory.confirm` | `{id, confidence?}` | `{id, verification_state: "human-verified"}` | `not-found` | `write` |
| `memory.governance_delete` | `{id, justification, purgeHistory?}` | `{id, deleted:true}` | `not-found` | `govern` |
| `dream.list` | `{}` | `{dreamingMemories:[{name, scope, mode, schedule, budget}]}` | — | `read` |
| `dream.configure` | `{name, config}` | `{name}` | `invalid-config`, `scope-denied` | `dream` |
| ↳ C11: v1 rejects `config.mode: cross-agent` with `invalid-config` (cross-agent is v2). | | | | |
| `dream.trigger` | `{name, resumeJobId?}` | `{jobId}` | `not-found`, `budget-exceeded`, `already-running`, `scope-denied` | `dream` |
| `dream.status` | `{jobId}` | `{state, progress?, startedAt}` | `not-found` | `read` |
| `dream.result` | `{jobId}` | `{summary, autoMerged:[…], reviewQueue:[…], cost_usd}` | `not-found`, `not-complete` | `read` |
| `system.status` | `{}` | `{plugins:[{kind,health}], worker, storePath, version}` | — | `read` |

`memory.confirm` is the R-1 `engram memory confirm <id>` MCP-side. It
bumps `verification_state: human-verified` and optionally raises
`confidence`.

**Error semantics** (research §1.3): protocol errors → JSON-RPC `error`
field; tool-execution errors → `result.isError: true` + `content`. engram's
internal `{ok, data, error}` envelope lives in `CoreService`; the MCP
adapter translates to the native MCP shape. `outputSchema` declared per
verb.

**`system.status`** additionally exposed as an MCP **resource**
(`engram://system/status`) with subscription, so monitoring clients learn of
plugin state changes without polling.

`dream.trigger` enforces: caller's `agentId` ∈ dreaming-memory's `scope`,
else `scope-denied` (S-13).

---

## 7. Access Control, Versioning, Concurrency

### 7.1 Access control

Each memory has `scope` (owning agent), `visibility` (`shared|private|hidden`),
`created_by`, optional `session`. Enforced by CoreService on every call:

- An agent reads: its own memory + everything `shared`.
- Session-scoped: only same `session`.
- **Dreaming output visibility invariant** (S-12): a dream-produced memory
  has visibility ≤ the **least permissive** source. If any input is
  `private`, output is at most `private`. Cross-agent dreaming (v2) cannot
  produce `shared` output from `private`/`hidden` inputs.
- **Hidden memory** (S-14, v1 guarantee): MCP layer never returns it in
  list/recall for other agents; stored in a per-agent subdir with 0700
  permissions owned by daemon process. Encryption-at-rest deferred to v2.

### 7.2 Optimistic concurrency (OCC)

Every memory carries `version`. Write protocol (§4.C):
1. Read file, capture `version`.
2. Compute change.
3. Write to `.tmp` in same directory.
4. **Acquire short-lived advisory lockfile.**
5. Re-read current `version`; if moved, release lock, retry (max 3, 50 ms
   jitter); else increment, `fsync(.tmp)`, `rename` to final, release lock.

This is Git-style — OCC with a tiny commit-time lock window, safe on POSIX.

**Three-way field merge** for dream-vs-main conflicts (F-5.3): manifest
records base versions; on conflict the orchestrator merges field-disjoint
changes automatically; field-overlap → review queue.

`recency`/`access_count` updates do NOT use OCC — they go to the **stats
sidecar** (F-1.7 / S-18, daemon-managed, agent cannot write them).

### 7.3 Versioning — hybrid (D-4: SQLite-only for v1)

- **Git** (store root, opt-in) — every memory write = a commit, author = agent
  identity. Branches for dreaming. `raw/`, `staging/`, `.engram/` git-ignored.
- **AppLog** (SQLite, always-on) — `.engram/app-log.db`. Per-field provenance.
  Event types: `write`, `lifecycle`, `access` (1-in-10 sampled), `governance`,
  `mcp_call`, `mcp_denied` (C14 — denied calls, source for `engram log --denied`),
  `capture`, `dream_run`, `ingest_run`, `occ_conflict`, `migration`.
  Retention: mutation/lifecycle/governance/dream_run never pruned.
  **AppLog is NOT git-tracked (C8 / OQ-L):** it lives inside the git-ignored
  `.engram/` and is its own authoritative SQLite store — git is the forensic
  backup of *memory files*, AppLog is the queryable history; neither is
  authoritative for the other (§10.1 principle 5). They are not redundant.

**v1 chooses SQLite-only** (D-4) — queryable via `engram history`;
tamper-evidence (hash-chaining, S-17) deferred to v2 under the single-user
trust model.

### 7.4 Write ordering & atomicity (closes F-5.4)

Canonical order:
1. Atomic file write (`.tmp` → fsync → rename).
2. AppLog append. On failure: spool to `applog-recovery.jsonl`.
3. Git commit (if enabled). On failure: spool to `git-pending.jsonl`.

**Markdown is authoritative.** AppLog and git are derived and reconciled on
daemon startup (scan for files newer than last app-log entry / last git
commit and replay). This makes every other store rebuildable — consistent
with "files are truth."

### 7.5 Provenance (split — OT.5)

`author` is split:
- **`created_by`** — immutable, set on birth.
- **`last_modified_by`** — updated on every mutation (summary; full history
  in AppLog).

Plus `ingest_run_id` (origin=ingested) and `staged_from` (distilled from
staging) for complete lifecycle traceability.

---

## 8. Threat Model & Security (D-6)

### 8.1 Trust boundary
engram is **local-first, single-user**. The OS-user boundary is the security
boundary. Multi-user → separate daemon instances.

| Actor | Trust |
|-------|-------|
| OS owner | Full |
| Registered agent (valid token) | Scoped (own scope + shared) |
| Unregistered process (same OS user) | None (cannot present valid token) |
| Network actor | None (no network surface; HTTP bound to 127.0.0.1) |

**Filesystem note**: any same-user process can read the store files;
engram's AC is at the MCP layer. `hidden` memories use 0700 dirs to add a
filesystem layer.

### 8.2 Assets

| Asset | Sensitivity | Protection |
|-------|-------------|------------|
| Memory content | High | AC at MCP; hidden at filesystem |
| Agent tokens | Critical | Per-agent files, 0600, daemon-owned |
| Staging | High | MAC-attested (S-02); not git-tracked |
| AppLog | High | SQLite (v1); hash-chained (v2) |
| Dreaming budget/config | Medium | Daemon-owned; agents cannot write |
| QMD/graphify indexes | Medium | Derived; purged in governance cascade |

### 8.3 v1 CRITICAL mitigations (required)

| Threat | Mitigation |
|--------|------------|
| **Memory poisoning via staging** (S-02/S-09) | Staging entries carry a per-agent MAC (capture plugin holds the secret). Worker verifies MAC before processing. Promotion rate limits + counterfactual gate (§5.2). |
| **Agent identity / impersonation** (S-03) | Per-agent bearer token, 0600, daemon-owned. Verb-scoped capabilities. Session tokens expire at `SessionEnd`. Revocable. |
| **Prompt injection via stored memory** (S-05) | Worker prompts treat memory bodies as untrusted data using explicit delimiters. Structured output (JSON schema) only — no free-form. Output validated against schema; frontmatter fields the body tried to inject are rejected. |
| **Safe/gated classification gaming** (S-11) | Deterministic predicate over the manifest diff; LLM cannot self-classify (§5.5). |
| **Visibility laundering by dreaming** (S-12) | Hard invariant: `visibility(dream_output) ≤ min(visibility(inputs))`. |
| **Secrets in git history** (S-08) | `engram governance scrub <pattern>` runs `git filter-repo`. `memory.governance_delete --purge-history` triggers history rewrite. **Documentation: do not push your store to a public remote.** |

### 8.4 v1 HIGH mitigations

- **Path traversal** (S-06): all `sources:` and `rawPath` are
  `path.resolve()`d + jailed under `<storeRoot>/raw/`.
- **Symlink escape** (S-07): file ops use `O_NOFOLLOW`; store-open scan
  rejects outbound symlinks inside `raw/`/`memories/`.
- **Budget DoS** (S-10): daemon enforces a hard token/USD ceiling at spawn
  time, sealed; dreaming worker cannot exceed it regardless of config.
- **Subprocess argument injection** (S-16): `execFile`/`spawn` with args
  array; never shell strings; allowlist-validated paths.
- **dream.trigger scope** (S-13): restricted to dreaming-memories whose
  `scope` includes the calling agent's identity. Cross-agent (v2) requires
  `cross-agent` capability.

### 8.5 Governance delete cascade

`memory.governance_delete --purge-history` purges from:
1. The Markdown file (working tree).
2. Git history (`git filter-repo`).
3. The AppLog (`event_type: governance_delete` recorded; the deleted
   memory's mutation rows are tombstoned).
4. QMD index (`RetrievalPlugin.deindex(id)`).
5. graphify graph (`GraphPlugin.removeNode(id)`).

Atomic from the user's perspective; dry-run by default.

### 8.6 Out of scope (v1)
Network attackers · multi-user/multi-machine · supply-chain LLM compromise ·
OS-owner-account compromise.

---

## 9. Failure Behaviour & Safety (adopted from failure-safety review)

### 9.1 Recall degradation chain
`memory.recall` NEVER fails hard.
```
1. QMD vector + BM25 hybrid       300 ms timeout    (primary)
2. QMD BM25 only                  150 ms timeout    (vector down/corrupt)
3. Filesystem grep                2000 ms timeout   (QMD unavailable)
4. Partial result + degraded:true                   (final)
```
Response carries `degraded?: {reason, tier_used}`. Stale hits annotated
`stale: true` when `file.mtime > last_reindexed`.

### 9.2 Retry policy (unified)
Exponential backoff with full jitter: `delay = rand(0, min(30s, 200ms·2^n))`.

| Operation | Retryable | Max attempts | Notes |
|-----------|-----------|--------------|-------|
| LLM `complete` | yes | 5 | non-retryable on 400 / context overflow → abort job |
| LLM `embed` | yes | 5 | same |
| OCC conflict | yes | 10 | no backoff; immediate re-read; 10 consecutive → surface to caller |
| graphify subproc | yes | 3 | on 3 failures: mark degraded, continue |
| QMD reindex | yes | 3 | on 3 failures: schedule full rebuild + stale flag |
| git commit | yes | 3 | on 3 failures: spool to `git-pending.jsonl` |
| AppLog append | yes | 5 | spool to `applog-recovery.jsonl` |

All retryable operations are **idempotent** (atomic file writes, append-only
log, OCC version tokens).

### 9.3 Dream job state machine — §5.4 above.

### 9.4 Merge validation (before touching main)
0. **Manifest schema validation (C6 / S-05):** validate the entire
   `manifest.json` against `dream-output.schema.json`. Failure → job `FAILED`,
   branch preserved, no merge. (This is the orchestrator-side stage of the S-05
   enforcement path; §5.2.)
1. YAML parse every modified frontmatter (reject parse errors).
2. Version tokens not regressed against base (OCC re-validation).
3. Lifecycle transitions within rate limits (§5.2.3).
4. `importance ≥ 0.05` (floor).
5. **Active-pool floor** (R-5): `active_count_after ≥ min(100, 0.2 · total)`.
   Violation → gate unconditionally regardless of `merge_policy`.
6. Visibility-invariant check (§7.1).

Failure → job `FAILED`, branch preserved.

### 9.5 Forgetting safety rails
- Per-dream rate limits: `min(50, 5%)` active→dormant, `min(20, 2%)`
  dormant→archived.
- Importance floor 0.05.
- Absolute active-pool floor (R-5).
- Rollback: `engram dream rollback <run_id>` = `git revert` of the merge
  commit. Branches retained 7 days post-merge.
- `engram lifecycle revive --since <ts>` for human override after a
  mass-dormancy bug.

### 9.6 Capture fire-and-forget — §6.1 above.

### 9.7 Write ordering & atomicity — §7.4 above.

### 9.8 Data integrity — `engram doctor`
Abbreviated on startup; full check weekly.

| Check | Detection | Action |
|-------|-----------|--------|
| Broken frontmatter | YAML parse error | ERROR; move to `.engram/quarantine/`; do not process |
| Missing `id` | parse | Assign ULID + write back; WARN |
| Duplicate `id` | id-index | Keep older (`created`); reassign newer; WARN |
| Dangling `relations:` edge | id-index check | WARN; `--fix` removes |
| `relations` → archived | lifecycle check | INFO (valid referent) |
| Missing `sources:` file | path check | WARN; `--fix` marks `missing` |
| Out-of-range value | range check | WARN; `--fix` clamps |
| Index/file divergence | row count + ids vs files | Schedule reindex |
| Orphaned dream branches | git branches vs jobs.db | Log; user confirms prune |

`--fix` enables auto-repair for safe issues. Unfixable → quarantine.

### 9.9 Startup / shutdown
**Startup gates** (ordered):
1. Acquire `.engram/engramd.pid` (handle stale PIDs).
2. Open jobs.db → crash-recovery scan: SPAWNED/RUNNING jobs with dead PID →
   `FAILED`.
3. Abbreviated `engram doctor`.
4. Plugin init order: LLM → Retrieval → Graph. **All plugins optional to boot
   (C13, consistent with §2.3 rule 5).** LLM init failure = **degraded**, not
   fatal: the daemon starts and serves `memory.get`/`list`/`history`/`recall`
   (recall via grep fallback), but **dreaming and ingest are unavailable** —
   `dream.trigger`/`memory.ingest` return `plugin-unavailable` until the LLM
   plugin is healthy. Retrieval failure = degraded (grep fallback). Graph
   failure = WARN, graph-dependent features disabled.
5. Bind MCP server `127.0.0.1` (retry 3× on port-in-use).
6. Drain `git-pending.jsonl` + `applog-recovery.jsonl` + `capture-fallback/`.
7. Backfill schedule for overdue dream jobs.
8. Log `engramd started` with version, store path, plugin health.

**Graceful shutdown (SIGTERM):**
1. Return 503 to new MCP requests.
2. Drain in-flight (10 s max).
3. SIGTERM dream workers; wait 30 s for checkpoint; SIGKILL + `TIMED_OUT`
   beyond that.
4. Flush AppLog buffer; drain reindex queue (5 s max).
5. Release PID lock; exit 0.

### 9.10 Schema versioning
`.engram/schema-version` file (semver). Daemon refuses to start against a
newer schema. `engram migrate` applies forward migrations explicitly (never
automatic). Migrations log to AppLog (`event_type: migration`).

---

## 10. Observability & Traceability (adopted from observability review)

### 10.1 Design principles
1. **No silent mutations** — every change emits an AppLog record.
2. **No silent capture** — every hook invocation logged, including
   suppressions and filtering.
3. **Dream runs self-document** — `.engram/agent-reports/drun_<id>.json`,
   git-tracked.
4. **Metrics over docs** — CLI + structured logs answer "is it healthy"; no
   log-grepping.
5. **AppLog = queryable history; git = forensic backup** — overlapping by
   design; neither is authoritative for the other's domain.

### 10.2 AppLog (`.engram/app-log.db`)

SQLite, single table `events`. Schema (sketch):
```sql
CREATE TABLE events (
  event_id      TEXT PRIMARY KEY,    -- ULID
  ts            TEXT NOT NULL,
  event_type    TEXT NOT NULL,       -- write|lifecycle|access|governance|
                                     -- mcp_call|capture|dream_run|
                                     -- ingest_run|occ_conflict|migration
  memory_id     TEXT,                -- nullable for non-memory events
  agent_id      TEXT,                -- created_by / last_modified_by / caller
  session_id    TEXT,
  field         TEXT,                -- mutation: which field
  old_value     TEXT,
  new_value     TEXT,
  reason        TEXT,                -- score breakdown / LLM rationale
  version_from  INTEGER,
  version_to    INTEGER,
  dream_run_id  TEXT,
  ingest_run_id TEXT,
  data_json     TEXT                 -- structured per-event payload
);
CREATE INDEX events_by_memory ON events(memory_id, ts);
CREATE INDEX events_by_agent  ON events(agent_id, ts);
CREATE INDEX events_by_type   ON events(event_type, ts);
```

CLI surface:
```
engram history <memory_id>           # per-memory timeline
engram history --agent <id>          # all changes by an agent
engram history --dream <run_id>      # all changes from a dream run
engram log --type governance         # governance actions
engram log --denied                  # denied MCP calls (audit; mcp_denied events — C14)
engram capture status                # capture summary for active session
```

**CLI verbs referenced elsewhere, defined here (C9)** — `govern` cap unless noted:
```
engram lifecycle revive --since <ts> # human override: re-activate memories
                                     #   dormant'd after <ts> (mass-dormancy bug
                                     #   recovery, §9.5). Dry-run by default.
engram governance scrub <pattern>    # bulk secret scrub: git filter-repo over
                                     #   history for <pattern> + deindex (§8.3).
                                     #   Dry-run by default; --apply to execute.
engram repair [--fix]                # remediation companion to `engram doctor`
                                     #   (§9.8): applies safe auto-repairs;
                                     #   unfixable → quarantine.
```

`access` events sampled 1-in-10 by default; configurable. Mutation /
lifecycle / governance / dream_run never pruned.

### 10.3 Daemon observability
- **CLI:** `engram status` reports daemon uptime, store stats, plugin
  health, queue depth, recall p50/p99, staging backlog, dream cost MTD.
- **Structured logs:** JSON to `~/.engram/logs/engramd.log` with fields
  `ts/level/component/msg + structured context`. Daily rotation; 14-day
  retention via logrotate (Linux) / newsyslog (macOS).
- **MCP `system.status` verb + resource** (engram://system/status with
  subscription) for programmatic access.
- **No HTTP metrics port in v1** (D-5 + S-15) — `engram status` is the
  surface; HTTP `/metrics` is a v2 dashboard concern. Note: the MCP server
  *is* a localhost TCP port (R-3 transport) but it is bearer-gated; this is
  a deliberate, separate surface from a metrics port.

### 10.4 Dream run audit
`.engram/agent-reports/drun_<id>.json`, git-tracked. Mandatory fields:
`run_id, dream_name, triggered_by, started_at, completed_at, status,
llm_model, tokens_in, tokens_out, estimated_cost_usd,
staging_files_considered, staging_files_distilled,
staging_files_discarded (with per-file reason), memories_created,
memories_modified (with old/new/reason), links_added, links_removed,
emergent_entity_proposals, review_queue_items, branch, commit,
merge_strategy, errors`.

Budget tracking: each run records consumed tokens against the
dreaming-memory's daily/monthly budget; daemon WARNs (log + `engram
status`) at 80% utilization.

### 10.5 Capture observability
- Filtered observations (no payload, only the matched rule) appended to a
  separate `capture-audit` channel in AppLog.
- `engram capture status` shows: active session, staging backlog (count +
  bytes), filter suppressions (last hour, by rule), capture-fallback size,
  hook failure count.

### 10.6 Operational
- `engram install` writes a systemd user unit (Linux) or launchd plist
  (macOS); installs log rotation. User-level — no root.
- Graceful restart (`engramd --reload` or SIGUSR2). Dream workers in flight
  complete on the old binary (process isolation).
- `engram backup` / `engram restore` for git-off stores or point-in-time
  snapshots.
- `engram migrate` for schema migrations (explicit; logged).

---

## 11. Technology Decisions

### 11.1 Final stack (v1)

| Area | Choice | Source |
|------|--------|--------|
| Language / runtime | **TypeScript / Node ≥22** | OQ-H resolved (research §6); QMD lib, MCP SDK, Vercel AI SDK all TS-first |
| Retrieval plugin | **QMD library (`@tobilu/qmd` ≥2.5.2), in-process (ESM)** | Spike 1 CONFIRMED in-process library mode (`createStore().search/searchLex/searchVector/update/embed`); no subprocess fallback needed |
| Graph plugin | **graphify (`graphifyy`) — MCP stdio for query, CLI `extract` for build** | Spike 3; local Ollama backend (hard prereq). Derived-index semantics — see §6.2 |
| LLM substrate | **Vercel AI SDK (`ai` package)** behind `LlmPlugin` | Spike 1b CONFIRMED (`generateObject`/`embed`); requires a **structured-output-capable model per provider** (worker validates output vs schema — C6) |
| Claude provider | raw `@anthropic-ai/sdk` inside `ClaudeLlmPlugin` (for `cache_control`) | research §5.4 |
| Dreaming worker | **Raw API loop via `LlmPlugin`** — NOT the Claude Agent SDK | OQ-J resolved (Agent SDK is Anthropic-only) |
| MCP transport | **Streamable HTTP over `127.0.0.1`** + bearer token | R-3 (research §1.5) |
| MCP SDK | `@modelcontextprotocol/sdk` | research §1.1 |
| Source of truth | Markdown files | locked |
| AppLog | SQLite (`.engram/app-log.db`) | D-4 |
| Job queue | SQLite (`.engram/jobs.db`) | OQ-I resolved (architecture §5.1) |
| Stats sidecar | SQLite (`.engram/stats.db`) | F-1.7 |
| Versioning | git store-wide (opt-in) + SQLite AppLog (always-on) | hybrid |
| Concurrency | OCC + short advisory lock at commit | OCC literature (research §6) |
| Distribution | public repo `github.com/Truncuso/engram` | locked |

### 11.2 Rejected (with reasons)

| Option | Why |
|--------|-----|
| Kuzu | archived 2025-10-10 |
| agentmemory as dep / fork | session-memory-shaped; iii engine dep; release cadence |
| Cognee / Neo4j / Mem0 | extraction pipelines (we already have explicit frontmatter edges) |
| Pure monolith | collapses dreaming process-decoupling |
| Thin-kernel pure microkernel | over-abstraction |
| Claude Agent SDK as worker engine | Anthropic-only, breaks LlmPlugin seam (research §5.4) |
| stdio MCP transport | wrong for always-up daemon serving multiple sessions (research §1.5) |
| unix-socket MCP + SO_PEERCRED | no SDK support; custom transport; bearer-token on HTTP is simpler (R-3) |
| RRF fusion at recall time | only one ranked source (QMD); graph is expansion, not parallel rank (F-1.4) |

### 11.3 Studied as patterns (no code adopted)

- **agentmemory** — staging buffer, 4-tier model, privacy filter,
  circuit-breaker, self-healing (`memory_diagnose`/`heal` → engram's
  `engram doctor`/`engram repair`), citation provenance, git snapshots. The
  51-tool surface is over-built for engram; the 16-verb set is the right
  trim.
- **book-to-skill** — `memory.ingest` generalizes this.
- **obsidian-wiki** — frontmatter richness, cost-ordered retrieval echoed;
  35-skill transplant rejected.
- **FadeMem** — per-type decay + importance-modulated `λ`.
- **MemMachine** — episodic immutability.
- **ElephantBroker** — two-layer contradiction detection +
  verification-state multiplier.
- **Generative Agents** — per-query min-max normalization + reflection
  trigger as cumulative-importance threshold.

---

## 12. v1 Scope, Open Questions, Success Criteria

### 12.1 v1 scope — final

**v1 includes:** §1–§11 of this spec, plus the v1 mitigations in §8. The
trims (D-7) apply: 2 relation kinds (`derived_from`, `related_to`);
cross-agent dreaming deferred (D-3); HTTP metrics port deferred (D-5);
hash-chained AppLog deferred (D-4).

**v1 explicitly OUT:** dashboard UI · cross-agent dreaming · capture plugins
beyond Claude Code · team/multi-machine sync · auto hard-delete · hosted/SaaS.

### 12.2 Open questions (remaining)

| OQ | Status | Lean |
|----|--------|------|
| OQ-A (transport-per-plugin) | RESOLVED | one contract, two transports |
| OQ-B (verb namespacing) | RESOLVED | `memory.*`/`dream.*`/`system.*` |
| OQ-C (recall return shape) | RESOLVED | id+summary+score, `expand` opt-in |
| OQ-D (store paths) | RESOLVED (v2.1) | `~/.engram/` global, `<repo>/.engram/` project |
| OQ-E (agent identity) | RESOLVED | per-agent bearer token via Streamable HTTP |
| OQ-F (QMD indexes staging/raw?) | RESOLVED | only `memories/` |
| OQ-G (raw/ git-LFS vs ignore) | RESOLVED (v2.1) | git-ignored default, LFS opt-in |
| OQ-H (language) | RESOLVED | TypeScript / Node ≥22 |
| OQ-I (job queue) | RESOLVED | SQLite `jobs.db` |
| OQ-J (Agent SDK vs raw API) | RESOLVED | raw API via `LlmPlugin` |
| OQ-K (AppLog SQLite vs JSONL) | RESOLVED | SQLite (D-4) |
| OQ-L (AppLog git-tracked) | RESOLVED (v2.1) | NO — AppLog is its own SQLite store in git-ignored `.engram/` (C8) |
| OQ-M (health surface) | RESOLVED | CLI `engram status` (no HTTP metrics in v1) |
| OQ-N (sample access events) | RESOLVED | 1-in-10 default |
| OQ-O (capture-fallback limits) | RESOLVED | 10MB / 7 days |
| OQ-P (dream audit location) | RESOLVED | both AppLog event + JSON file |
| OQ-Q (doctor auto-repair) | RESOLVED | report-only by default; `--fix` opt-in |
| OQ-R (SO_PEERCRED) | RESOLVED (dropped) | superseded by bearer token + HTTP |
| OQ-S (governance delete cascade) | RESOLVED | §8.5 |
| OQ-T (staging granularity) | RESOLVED | per-session JSONL |

### 12.3 v1 Success Criteria

- [ ] `engram init` scaffolds a store (global or project), idempotent.
- [ ] Agent `remember`s a fact; later session `recall`s it, with scored
      ranking and a confidence-multiplier breakdown.
- [ ] Raw PDF in `raw/` is `ingest`ed (worker job) into typed Markdown
      memories with `sources:` provenance + LLM-built tags.
- [ ] Capture hooks record a Claude Code session's observations + failures
      into `staging/` per-session JSONL; privacy filter strips a planted
      secret (and the filter audit log records the strip).
- [ ] A dreaming run distills staging → memories with `derived_from`
      backlinks, inserts links, re-weights importance, writes ≥1 procedural
      memory from a failure pattern at confidence 0.3 (uncorroborated → not
      auto-active); a second corroborating episode promotes it to active.
- [ ] Two-layer contradiction detection catches a planted contradiction and
      routes it to the review queue.
- [ ] `forget` moves a low-score memory to dormant (after ≥2 consecutive
      runs below threshold); still searchable on demand; never deleted.
- [ ] A concurrent write losing an OCC race is rejected, retried cleanly,
      and the conflict logged in AppLog.
- [ ] `memory.history <id>` shows per-field provenance with dream_run
      linkage.
- [ ] Access control: agent B cannot recall agent A's `private` memory; a
      planted cross-agent dreaming attempt is denied in v1.
- [ ] A dream worker crash leaves the core daemon healthy; the orphaned job
      transitions to `TIMED_OUT`; `engram dream resume` re-runs idempotently
      from checkpoint with no staging loss.
- [ ] A planted prompt-injection in a memory body fails to mutate frontmatter
      because schema validation rejects the injected fields.
- [ ] `dream.trigger` from an agent outside the dreaming-memory's scope
      returns `scope-denied`.
- [ ] Active-pool floor blocks a mass-archive dream run regardless of
      `merge_policy: always-auto`.
- [ ] `engram doctor` detects a planted broken-frontmatter file and
      quarantines it without breaking daemon startup.
- [ ] `governance_delete --purge-history` purges a secret from working tree,
      git history, AppLog (tombstoned), QMD index, and graphify graph.
- [ ] Streamable HTTP MCP on `127.0.0.1` accepts a bearer token, denies
      missing/bad tokens, exposes 16 verbs + `engram://system/status`
      resource.
- [ ] `engram status` reports daemon healthy with plugin health for QMD,
      graphify, LLM.

---

## 13. Implementation Phases (preview for `writing-plans`)

Not a plan — a sequencing preview that the plan skill will refine.

1. **Core scaffold** — store layout, frontmatter schema, AppLog, jobs.db,
   stats.db, CoreService skeleton, OCC, write ordering, `engram init`,
   `engram doctor` (abbreviated), schema-version.
2. **Plugin host + LLM plugin (Vercel AI SDK)** — manifest, lifecycle, both
   transports; ClaudeLlmPlugin (with `cache_control`), OpenAiLlmPlugin,
   OllamaLlmPlugin behind the same `LlmPlugin` interface.
3. **Retrieval (QMD)** — verify Node-22 library mode; if blocked, MCP
   subprocess transport. Wire `index/update/deindex/search/rebuild`. Stats
   sidecar wiring for recency.
4. **Scoring engine + recall** — Importance × Relevance × Recency × m_v,
   per-query normalization, degradation chain, sidecar touch.
5. **MCP server + CoreService facade** — Streamable HTTP, bearer-token
   gating, 16 verbs, system.status resource.
6. **Capture (Claude Code) + CaptureIntake + staging** — per-session JSONL,
   multi-layer privacy filter, MAC attestation, fail-closed, fire-and-forget,
   capture-fallback.
7. **Ingest worker** — graphify subprocess (local Ollama backend), path
   jail, raw→memories pipeline.
8. **Dreaming worker + orchestrator** — manifest, deterministic safe/gated,
   rate limits, active-pool floor, three-way merge, state machine, watchdog
   + reaper, idempotent resume, rollback.
9. **Threat-model hardening** — every CRITICAL mitigation (§8.3) verified
   with a planted-attack test.
10. **Governance + cascade delete** — `--purge-history`, deindex cascade,
    documentation around public-remote risk.
11. **E2E verification** — every §12.3 success criterion as an automated
    test.

---

## 14. Round-3 Verification

A lighter follow-up review (the `judge` skill or one architect-reviewer
pass) is recommended before implementation, to confirm SPEC v2 is internally
consistent now that all rounds are folded in. Findings against this
document, not against v1.

---

*SPEC v2 complete. Folds D-1…D-7, R-1…R-5, the 4 review contracts, and the
4 research reports. Living document — append decisions and findings here
as the design hardens during implementation.*
