---
title: "engram — Architecture Review"
project: engram
reviews: docs/spec/SPEC.md (DRAFT 2026-05-22), docs/adr/0001
reviewer: architect-reviewer
created: 2026-05-22
status: review — for spec amendment
---

# engram — Architecture Review

Scope: architectural soundness of the microkernel + detached-dreaming-worker
design, with a primary mandate to close the **clean-definition gaps** the spec
author flagged — information flow, plugin contract, MCP contract, and the
core/plugin and core/worker boundaries.

**Verdict.** The macro-architecture is sound: the microkernel choice is
justified (four seams with real plurality), the process-split for dreaming is
correct, and the rejection rationale (Kuzu, monolith, thin-kernel) is honest.
The spec is *strong on vision and weak on contracts*. It is, by its own
admission, a design narrative — not yet an implementable specification. Six of
the ten open questions (OQ-A, C, E, F, I, J) are not "decide later" items;
they are load-bearing architectural decisions that block a clean module
decomposition. This review resolves them and supplies the four missing
contract definitions verbatim for lifting into the spec.

Severity legend: **CRITICAL** (correctness/data-loss/blocks design) ·
**HIGH** (significant gap, fix before plan) · **MEDIUM** (maintainability) ·
**LOW** (note).

---

## 1. Information Flow

### 1.1 Findings

**F-1.1 — No single data-flow specification exists. [HIGH]**
The flows are scattered across §3.2, §4, §5.2 as prose. There is no diagram or
table that traces a unit of information end-to-end. A reader cannot answer
"what happens between a `PostToolUse` hook firing and a procedural memory
existing" without stitching together three sections. The spec must adopt one
canonical data-flow section. Definition supplied in §1.2 below.

**F-1.2 — `staging/` lifecycle is undefined. [CRITICAL]**
§4.1 says capture writes to `staging/` and dreaming "distills staging into real
memories and discards noise." Unspecified:
- What is a staging record? File format? One file per observation, or an
  append log? (Capture is high-volume — one-file-per-observation will produce
  tens of thousands of inodes.)
- When is a staging record deleted? After distillation? After the dream branch
  *merges*? If deleted at distill time and the dream branch is later rejected
  in review, the raw observation is lost — **data loss on a rejected dream**.
- Is staging scoped per-agent/per-session? Two agents capturing concurrently
  write to the same `staging/` — collision and access-control hole.
- Is staging crash-safe? If `engramd` dies mid-hook, is the observation lost?

**F-1.3 — Ingest concurrency and trigger are undefined. [HIGH]**
§4.2: "a file dropped into `raw/` … is processed." By what? A filesystem
watcher? Polling? Only on explicit `memory.ingest`? §3.3 says `raw/` is
git-ignored — so a dropped file has no version event. The trigger mechanism is
absent. Also: ingest runs graphify + LLM distillation — this is a *heavy*
operation with the same latency profile as dreaming, yet the spec runs it
inside the core daemon (§4.2 implies synchronous). It should use the worker
process, not the core. See F-4.3.

**F-1.4 — `recall` flow ordering between QMD and graph is unspecified. [HIGH]**
§3.6 defines the score `f(I, R, Recency)` with Relevance from QMD. §4.4 calls
recall "scored hybrid retrieval." But the graph plugin's role in recall is
never stated. Does recall: (a) QMD-only, graph used elsewhere; (b) QMD then
graph-expand the hit set; (c) parallel QMD + graph with RRF fusion (the
agentmemory pattern §8.4 cites)? This is *the* core retrieval algorithm and it
is undefined. Decision needed — see §1.2 recall flow.

**F-1.5 — Dreaming job lifecycle has no state machine. [HIGH]**
§5 describes what a dream *does* but not the job's states. `dream.status` (§4.4)
"polls a running/queued dream job" — implying at least `queued`/`running`/
`done`, but the full set, the transitions, and the terminal failure states are
absent. Without this, the worker, the orchestrator, and the `dream.status` verb
cannot be specified consistently. State machine supplied in §1.2 and §5.

**F-1.6 — Branch→merge step has no actor. [MEDIUM]**
§5.2 step 5 "commit to a `dream/<name>/<ts>` branch"; §5.3 says auto-safe
changes are "auto-merged." *Who* merges, and *when*? The worker exits after
committing (§2.2). So the core daemon's dreaming-orchestrator must do the
merge — but that means the orchestrator runs `git merge`, classifies each
change as additive vs gated, and splits a single branch into auto-applied +
review-queued hunks. That is substantial undescribed logic. Either the worker
pre-classifies (writes a manifest), or the orchestrator does. Pick one — see
§1.2.

**F-1.7 — `recency` touch-on-recall mutates a memory on every read. [MEDIUM]**
§3.4/§3.6: recall "touches" `recency` and `access_count`. So `memory.recall`
is a *write*. Under OCC (§6.2) that means every recall increments `version`,
produces an app-log entry, and (if git-on) a commit. A read-heavy workload
becomes write-heavy; git history fills with "recall touched recency" commits.
This needs an explicit decoupling: recency/access updates must go to a separate
non-versioned sidecar (e.g. the `.engram/` index cache or a stats table), *not*
the memory file's frontmatter+git. Recommendation in §1.2.

### 1.2 Clean data-flow definition (to adopt into the spec as §3.x or §4.0)

Five canonical flows. Each is a pipeline with named stages, a triggering event,
and a terminal state.

**Flow A — Capture → staging → dreaming → memories**
```
[CC hook fires]
  → CapturePlugin.onObservation(RawObservation)
  → privacy filter (strip secrets)        [core, synchronous, fast]
  → append to staging/<agent>/<session>.jsonl   [core, one append-only log
                                                  per session — not per-obs]
  → fsync; return to hook                 [crash-safe boundary]
... later ...
[dream trigger]
  → worker reads staging/<scope>/*.jsonl
  → LLM distill → candidate typed memories
  → worker writes to dream branch + manifest.json (per-change classification)
  → worker marks each consumed staging record (offset watermark in manifest)
  → worker exits
  → orchestrator: auto-merge additive hunks; queue gated hunks
  → ON MERGE COMMIT (not before): orchestrator truncates staging logs up to
    the watermark   ← staging is retained until the dream that consumed it
                      is durably merged (fixes F-1.2 data-loss)
```

**Flow B — Ingest (raw → memories)**
```
[memory.ingest(path) OR raw/ watcher debounced event]
  → core enqueues an ingest job on the SAME job queue as dreaming
  → worker (ingest mode): graphify extract structure
                          → LLM distill → typed memories w/ sources:
  → worker writes to a branch + manifest; exits
  → orchestrator merges (ingest output is additive → auto-safe by default)
```
Ingest is a worker job, not core work (fixes F-1.3).

**Flow C — Explicit remember**
```
[memory.remember(payload)]  OR  [worker self-authored note]
  → access-control check (scope/visibility vs caller identity)
  → schema-validate frontmatter
  → OCC: if updating, verify version token
  → write memory file + app-log entry (+ git commit if enabled)
  → RetrievalPlugin.index(memory)   [async, best-effort, see F-2.x]
  → GraphPlugin.ingestEdges([memory])  [async, best-effort]
  → return {id, version}
```

**Flow D — Recall (resolve here: hybrid = QMD primary, graph expansion)**
```
[memory.recall(query, opts)]
  → access-control: caller identity → readable scope set
  → RetrievalPlugin.search(query, {scope filter})  → ScoredHit[] (Relevance)
  → [optional, opts.graph_expand] GraphPlugin.traverse from top-k hits
       → add 1-hop neighbours to candidate set
  → core scoring: rank = f(Importance, Relevance, Recency)   [core, not plugin]
       — Relevance from QMD; Importance, Recency from memory metadata
  → filter lifecycle != active unless opts.include_dormant
  → return id+summary[] (bodies only if opts.expand)
  → recency/access update → STATS SIDECAR (.engram/stats.db), async,
       NOT the memory file, NOT git, NOT app-log   (fixes F-1.7)
```
Fusion model is decided: **QMD provides Relevance; the core scoring engine
fuses I×R×Recency; graph is an opt-in expansion, not a parallel ranker.** RRF
is unnecessary because there is only one ranked source (QMD); the graph adds
recall (candidates), not a competing rank. This keeps the scoring engine the
single ranker and avoids two tunable fusion stages.

**Flow E — Dream job lifecycle** — see §5 state machine.

---

## 2. The Plugin Contract

### 2.1 Audit of the §2.4 sketch

The four interfaces are a *capability sketch*, not a contract. Gaps:

**F-2.1 — No lifecycle. [CRITICAL]** None of the four interfaces has
`init`/`shutdown`/`health`. The Plugin Host (§2.2) is named as "loads +
supervises the 4 plugins" but supervision requires a health probe and a
shutdown hook. Without `init(config)` there is nowhere to pass the QMD store
path, the LLM API key, or the graphify subprocess path.

**F-2.2 — No error semantics. [CRITICAL]** Every method returns `Promise<T>` —
rejection behaviour is unspecified. Is a QMD `search` failure fatal to
`recall`, or does recall degrade to metadata-only? Is an LLM timeout
retryable? Must a `RetrievalPlugin.index` failure roll back the `remember`
write? The contract must classify errors and state core's reaction. (Strong
recommendation: index/ingestEdges failures are **non-fatal** — files are truth,
the index is rebuildable per §2.4's own `rebuild()`. A failed `index` enqueues
a re-index, it does not fail `remember`.)

**F-2.3 — No contract versioning. [HIGH]** The plugin *contract itself* is
unversioned. The spec's whole rationale is swappability over time; a contract
with no version field cannot evolve without silently breaking plugins. Add a
`contractVersion` and a host-side compatibility check at load.

**F-2.4 — `CapturePlugin.onObservation` is an in-process callback — incompatible
with OQ-A. [CRITICAL]** `onObservation(cb)` registers a JS callback. But CC
hooks are *external OS processes* that fire when no engram code is on the call
stack. A hook cannot invoke a JS callback in `engramd`. The real mechanism is:
the hook process makes an IPC call (HTTP/socket/CLI) to `engramd`. So
`CapturePlugin` is not "a plugin that emits observations" — it is "a plugin
that (a) installs hook scripts and (b) defines how a hook process talks to the
daemon." The interface is modelling the wrong thing. Redefined in §2.3.

**F-2.5 — Asymmetry: no `Memory` ID vs object discipline. [MEDIUM]**
`index(memory: Memory)` takes a full object; `traverse(from: MemoryId)` takes
an id. `ingestEdges(memories: Memory[])` takes objects. The plugin should never
need full `Memory` objects with bodies if it indexes from disk — decide whether
plugins read the store themselves (pass `MemoryId` + store path) or are pushed
hydrated objects. Pushing full objects couples the plugin to the core's
in-memory representation. Recommend: pass `MemoryRef` (id + path + frontmatter),
plugin reads the body if it needs it.

**F-2.6 — `RetrievalPlugin` has no delete/deindex. [HIGH]** `forget` moves a
memory to dormant and `governance_delete` hard-deletes. The index must be told.
There is no `deindex`/`update` method. Currently the only repair is full
`rebuild()` — unacceptable per-operation.

**F-2.7 — OQ-A is resolvable now, and the spec's lean is right but
under-specified. [HIGH]** The lean (in-process for retrieval/LLM, subprocess
for graphify) is correct, but a *mixed* loading model means two host code
paths. Resolve cleanly: **all plugins are addressed through one out-of-process-
capable contract; the host supports two transports — `in-process` (a TS module)
and `subprocess` (JSON-RPC over stdio) — selected per plugin by a `transport`
field in plugin config.** The *interface* is identical either way; only the
host's invocation differs. This means graphify (Python, subprocess) and QMD
(TS, in-process) implement the same contract, and a future Python QMD
replacement needs no contract change. This is the clean resolution of OQ-A and
it should be written into the spec, closing the OQ.

### 2.2 OQ-A — decision

**Decided: one contract, two transports, declared per-plugin.** In-process =
direct TS calls (QMD, LLM). Subprocess = JSON-RPC over stdio (graphify, and any
future non-TS plugin). The host owns transport; plugins are transport-agnostic.
Rationale: a single contract surface, no FFI, no second contract to maintain,
and the seam genuinely protects against the Kuzu-class mistake because a
replacement engine in any language drops in behind the same JSON-RPC shape.

### 2.3 Hardened plugin contract (to replace §2.4 verbatim)

```ts
// ---- contract meta ------------------------------------------------------
const PLUGIN_CONTRACT_VERSION = "1.0";

type PluginKind = "retrieval" | "graph" | "capture" | "llm";
type Transport  = "in-process" | "subprocess";

interface PluginManifest {
  kind: PluginKind;
  name: string;                       // "qmd", "graphify", "claude-code"
  contractVersion: string;            // must satisfy host's semver range
  transport: Transport;
  config: Record<string, unknown>;    // store path, API key ref, exe path…
}

// every plugin error is one of these — core branches on `kind`
type PluginErrorKind =
  | "unavailable"     // plugin/engine down — core may degrade or retry
  | "invalid-input"   // caller bug — do not retry
  | "transient"       // timeout / rate-limit — retry with backoff
  | "fatal";          // contract/version mismatch — host unloads plugin
interface PluginError { kind: PluginErrorKind; message: string; retryable: boolean; }

// ---- lifecycle: EVERY plugin implements this ----------------------------
interface PluginLifecycle {
  init(manifest: PluginManifest): Promise<void>;   // idempotent
  health(): Promise<{ ok: boolean; detail?: string }>;
  shutdown(): Promise<void>;                        // flush + release
}

// a lightweight handle — id + path + frontmatter, NOT the body
interface MemoryRef { id: MemoryId; path: string; frontmatter: Frontmatter; }

// ---- retrieval ----------------------------------------------------------
interface RetrievalPlugin extends PluginLifecycle {
  index(ref: MemoryRef): Promise<void>;             // non-fatal on failure
  update(ref: MemoryRef): Promise<void>;            // re-index one
  deindex(id: MemoryId): Promise<void>;             // forget/governance-delete
  search(query: string, opts: SearchOpts): Promise<ScoredHit[]>;
  rebuild(store: StorePath): Promise<void>;         // full derived rebuild
}
// SearchOpts: { scopeFilter: Scope[]; limit: number; includeDormant: boolean }
// ScoredHit:  { id: MemoryId; relevance: number; summary: string }

// ---- graph --------------------------------------------------------------
interface GraphPlugin extends PluginLifecycle {
  ingestEdges(refs: MemoryRef[]): Promise<void>;    // non-fatal on failure
  removeNode(id: MemoryId): Promise<void>;
  traverse(from: MemoryId[], opts: TraverseOpts): Promise<GraphResult>;
  extract(rawPath: string): Promise<ExtractedStructure>;  // ingest support
  rebuild(store: StorePath): Promise<void>;
}

// ---- capture ------------------------------------------------------------
// NOT a callback emitter. It (a) installs hook scripts that POST to engramd,
// (b) declares the wire shape, (c) normalises an inbound payload.
interface CapturePlugin extends PluginLifecycle {
  harness: string;                                  // "claude-code"
  install(target: HookInstallTarget): Promise<void>;   // write hook scripts
  uninstall(target: HookInstallTarget): Promise<void>;
  normalise(raw: unknown): RawObservation;          // wire payload → typed obs
}
// the daemon exposes an internal capture-ingest endpoint; hook scripts call it.

// ---- llm ----------------------------------------------------------------
interface LlmPlugin extends PluginLifecycle {
  complete(prompt: Prompt, opts: LlmOpts): Promise<LlmResult>;
  embed(texts: string[]): Promise<Float32Array[]>;  // batched
}
// LlmResult: { text: string; tokensIn: number; tokensOut: number }
//            — token counts feed the dreaming budget (§5.1)
```

Contract rules the spec must state alongside the code:
1. **Files are truth.** `index`/`ingestEdges`/`update`/`deindex` failures are
   `non-fatal`: core logs, enqueues a retry, and the write still succeeds. A
   `rebuild()` always recovers correctness.
2. **`search`, `complete`, `embed`, `traverse` failures *are* surfaced** to the
   caller as a degraded result with an explicit `partial: true` flag — recall
   never silently returns fewer hits.
3. **The host probes `health()` on a schedule;** an `unavailable` plugin is
   restarted (subprocess) or marked degraded (in-process).
4. **`contractVersion` is checked at `init`;** a mismatch is a `fatal` error and
   the plugin is not loaded.
5. **All four plugins are optional to the *kernel's* boot** — the core daemon
   starts, serves `memory.get`/`list`/`history` from files, and reports plugin
   health; recall degrades, dreaming is unavailable, until plugins are up.

---

## 3. The MCP Contract

### 3.1 Audit of §4.4

§4.4 lists 11 verbs as a name+purpose table. No verb has defined params,
returns, errors, or required capability. This is the single largest
implementability gap in the spec.

**F-3.1 — No I/O schemas. [CRITICAL]** Not one verb has a typed signature. The
MCP server cannot be built, and a Claude Code integration cannot be written,
from a purpose sentence.

**F-3.2 — Capability-gating is asserted, not designed. [CRITICAL]** §4.4 and §6
say the server "enforces by agent identity," gated by "access control." But
the *set of capabilities* is never enumerated, and no verb is mapped to a
required capability. OQ-E (how identity is established) is open — and it must
be closed because gating depends on it. Capability model supplied in §3.3.

**F-3.3 — Verb-set coherence: two real problems. [HIGH]**
- `memory.get` / `memory.list` are bundled as one row — they are two verbs.
- `dream.trigger`/`dream.status`/`dream.result` are a fine triad, but there is
  no `dream.list` (what dreaming-memories exist?) and no `dream.configure`
  (§5.1 dreaming-memory objects are user-configurable — via what? editing
  `.dreaming/*.md` by hand only? then say so).
- No `memory.update`. `remember` is described as write-new; OCC (§6.2) is about
  *updates*. Either `remember` does upsert (then say it, and it needs a version
  param) or an `update` verb is missing.
- No health/introspection verb. A client cannot ask "are plugins up, is
  dreaming available." Add `system.status`.

**F-3.4 — `recall` return shape (OQ-C) blocks the schema. [HIGH]** Resolvable
now: id+summary by default, `expand` opt-in (the lean is correct and
token-frugal). Decided below.

### 3.2 OQ-C and OQ-E — decisions

- **OQ-C:** `recall` returns `id + summary + score breakdown`; `expand: true`
  includes bodies. Decided.
- **OQ-E:** Identity = a per-agent token minted at `memory.init` (or `engram
  agent add`), stored by the client, revocable, presented on every MCP call.
  The token maps to an `agentId` and a capability set. Decided.

### 3.3 Capability model

Five capabilities, granted per-agent-token at mint time:

| Capability | Grants |
|------------|--------|
| `read`       | `recall`, `get`, `list`, `history` within readable scope |
| `write`      | `remember`, `update`, `forget`, `ingest` |
| `dream`      | `dream.trigger`, `dream.configure` for owned dreaming-memories |
| `cross-agent`| recall/dream across scopes beyond the caller's own |
| `govern`     | `governance_delete`, `init`, agent-token management |

A default session agent gets `{read, write, dream}` scoped to itself. A
human/admin token gets all five. The MCP server checks the capability **and**
the scope on every call (capability = verb permission; scope = data
permission).

### 3.4 Clean MCP tool contract (to replace §4.4 verbatim)

Namespacing: `memory.*`, `dream.*`, `system.*` (resolves OQ-B). All verbs take
an implicit `authToken`; all return a uniform envelope
`{ ok, data, error }` where `error = { code, message }`.

| Verb | Params | Returns | Errors | Cap |
|------|--------|---------|--------|-----|
| `memory.init` | `{scope: "global"\|"project", path?}` | `{storePath, agentToken}` | `already-exists`, `path-denied` | `govern` |
| `memory.remember` | `{type, category?, title, body, tags?, scope?, visibility?, sources?, relations?}` | `{id, version}` | `invalid-schema`, `scope-denied` | `write` |
| `memory.update` | `{id, version, patch}` | `{id, version}` | `version-conflict`(OCC), `not-found`, `scope-denied` | `write` |
| `memory.recall` | `{query, limit?, type?, category?, includeDormant?, graphExpand?, expand?}` | `{hits:[{id, summary, score:{importance,relevance,recency,total}, body?}], partial}` | `retrieval-unavailable` | `read` |
| `memory.get` | `{id, expand?}` | `{memory}` | `not-found`, `scope-denied` | `read` |
| `memory.list` | `{type?, category?, scope?, lifecycle?, limit?, cursor?}` | `{refs:[MemoryRef], cursor?}` | — | `read` |
| `memory.forget` | `{id, version}` | `{id, lifecycle}` | `version-conflict`, `not-found` | `write` |
| `memory.ingest` | `{rawPath}` | `{jobId}` (async — worker job) | `not-found`, `unsupported-type` | `write` |
| `memory.history` | `{id}` | `{events:[{ts, author, field, old, new, reason}]}` | `not-found` | `read` |
| `memory.governance_delete` | `{id, reason}` | `{id, deleted:true}` | `not-found` | `govern` |
| `dream.list` | `{}` | `{dreamingMemories:[{name, scope, mode, schedule}]}` | — | `read` |
| `dream.configure` | `{name, config}` | `{name}` | `invalid-config`, `scope-denied` | `dream` |
| `dream.trigger` | `{name}` | `{jobId}` | `not-found`, `budget-exceeded`, `already-running` | `dream` |
| `dream.status` | `{jobId}` | `{state, progress?, startedAt}` | `not-found` | `read` |
| `dream.result` | `{jobId}` | `{summary, autoMerged:[…], reviewQueue:[…]}` | `not-found`, `not-complete` | `read` |
| `system.status` | `{}` | `{plugins:[{kind, health}], worker, storePath}` | — | `read` |

16 verbs, each fully specified. This is the minimal coherent set: it adds the
four missing verbs from F-3.3 and splits `get`/`list`. `dream.configure` is
the documented way to edit a dreaming-memory (also editable as `.dreaming/*.md`
on disk — the verb and the file are two views of one object).

---

## 4. Core / Plugin Boundary

**F-4.1 — Boundary is drawn correctly; one leak. [HIGH]** The fixed-core set
(store, scoring, versioning, access control, dreaming-orchestrator, plugin-host,
API) is the right identity surface, and the four seams are the right seams. The
one mistake: **`recall` as currently described couples scoring to retrieval.**
§3.6 has Relevance "computed per query (QMD vector similarity)." If the scoring
engine *calls QMD* it depends on the retrieval plugin. Correct decoupling:
`RetrievalPlugin.search` returns a `relevance` number as raw output; the
**scoring engine consumes that number and never knows it came from QMD**. The
scoring engine's input is `{importance, relevance, recency}` — three floats. As
long as the spec states scoring takes a `relevance: number` and does not import
the retrieval plugin, the boundary is clean. Make that explicit (it currently
reads as if scoring reaches into QMD).

**F-4.2 — Scoring must not depend on the graph. [LOW]** §5.2 step 2/3 has
dreaming re-weight `importance` "based on how connected/central the memory is"
— i.e. graph centrality. That is a *dreaming-time* computation done by the
worker (which may call the graph plugin), then written to frontmatter. The
*scoring engine* still only reads the stored `importance`. Confirm the spec
keeps centrality in the worker, not in the core scoring engine. As written it
is fine — flag it so it stays that way.

**F-4.3 — Dreaming-orchestrator vs worker split is under-drawn. [HIGH]**
"Dreaming Orchestrator" is in the fixed core; the worker is a separate process.
The spec never states the division of labour. Clean split:
- **Orchestrator (core):** owns the job queue; decides *when* to dream
  (schedule, triggers, budget check); spawns the worker; on worker exit, reads
  the manifest, merges auto-safe hunks, populates the review queue; updates job
  state. Owns no LLM logic.
- **Worker (detached):** owns *all* LLM work — distill, connect, re-weight,
  verify; reads staging + memories; writes a branch + manifest; exits. Owns no
  scheduling, no merge, no queue.

The danger called out in the task ("does dreaming-orchestrator leak into the
worker") is real *if* the worker does its own merge or its own scheduling. It
must not. The worker's only outputs are a git branch and a manifest file. The
orchestrator never runs LLM calls. State this in §5.

**F-4.4 — Ingest is mis-placed. [HIGH]** As noted in F-1.3, §4.2 implies ingest
runs in core. Ingest is LLM+graphify heavy — same isolation argument as
dreaming. Move ingest to a worker job (Flow B). The core only enqueues.

---

## 5. Job Queue / IPC Boundary

OQ-I leans "SQLite table in `.engram/`." That is the right primitive; the spec
does not yet specify the *protocol*.

**F-5.1 — No job record schema, no state machine. [CRITICAL]** "Shared store +
job queue" is a phrase, not a design. Supplied below.

**F-5.2 — Worker-death mid-job is unhandled. [CRITICAL]** §5.5 guarantees a
crash "never touches the core daemon" — true, process isolation gives that. But
the *job* is now orphaned: stuck in `running` forever, `dream.status` polls a
dead job, and the next scheduled dream may not start (`already-running` per
§3.4). There must be a liveness mechanism and a reaper.

**F-5.3 — Concurrent dreams over overlapping scopes can race on merge. [HIGH]**
Two dreaming-memories with overlapping scope, or a dream + a live agent write,
both touch the same memory files. The dream worked off a branch point; by merge
time the file moved. OCC (§6.2) covers field-level conflicts on `remember`, but
a *git merge* of a dream branch is a different conflict surface. The spec must
state: dream branches merge with OCC re-validation per memory (manifest carries
the base `version` of each touched memory; orchestrator rejects a hunk whose
base version moved → that hunk goes to the review queue as a conflict).

**F-5.4 — SQLite + git + Markdown = three stores, one missing transaction
story. [HIGH]** A write touches: the `.md` file, the app-log, git, and
(later) the index. A crash between any two leaves inconsistency. The spec needs
an ordering + recovery rule. Recommended invariant: **the Markdown file is
committed first and is authoritative; app-log, git commit, and index are
derived and reconciled on startup** (startup scans for memories newer than the
last app-log entry / last git commit and replays). This makes every other
store rebuildable — consistent with "files are truth."

### 5.1 Clean job-queue definition (to adopt as §5.x)

Single SQLite DB at `.engram/jobs.db`, one `jobs` table. SQLite gives ACID
enqueue/claim for free; WAL mode allows the core (writer) and worker (status
updater) to operate concurrently.

```sql
CREATE TABLE jobs (
  id           TEXT PRIMARY KEY,        -- ULID
  kind         TEXT NOT NULL,           -- 'dream' | 'ingest'
  target       TEXT NOT NULL,           -- dreaming-memory name | raw path
  state        TEXT NOT NULL,           -- see state machine
  pid          INTEGER,                 -- worker OS pid, NULL until spawned
  heartbeat_at TEXT,                    -- worker updates every N seconds
  branch       TEXT,                    -- dream/<name>/<ts>, set by worker
  manifest     TEXT,                    -- path to manifest.json
  attempts     INTEGER NOT NULL DEFAULT 0,
  error        TEXT,
  created_at   TEXT NOT NULL,
  updated_at   TEXT NOT NULL
);
```

**Job state machine** (also the `dream.status` vocabulary — fixes F-1.5):
```
queued ──(orchestrator spawns worker)──▶ running
running ──(worker commits branch+manifest, exits 0)──▶ merging
running ──(worker exits non-zero)──────────────────▶ failed
running ──(heartbeat stale > 3×interval)───────────▶ orphaned
merging ──(orchestrator merge done)────────────────▶ done
merging ──(merge conflict)─────────────▶ done (w/ review-queue conflicts)
orphaned ──(reaper, attempts < max)────────────────▶ queued   (retry)
orphaned ──(reaper, attempts >= max)───────────────▶ failed
failed/done = terminal
```

**Rules:**
1. **Claim is atomic.** Orchestrator transitions `queued→running` and writes
   `pid` in one SQL transaction — no double-spawn.
2. **Heartbeat.** Worker updates `heartbeat_at` every 10 s. A reaper in the
   core scans for `running` jobs with `heartbeat_at` older than 30 s, verifies
   the `pid` is dead, and transitions to `orphaned` (fixes F-5.2).
3. **Idempotent retry.** A retried job re-reads staging from the watermark; the
   prior dream branch is discarded. Because staging is only truncated on merge
   (Flow A), a crashed dream loses no observations.
4. **Single-flight per dreaming-memory.** `dream.trigger` on a `name` with a
   non-terminal job returns `already-running` — no overlapping dreams of the
   same scope.
5. **Merge is core-side and OCC-validated** (F-5.3): the manifest records each
   touched memory's base `version`; the orchestrator applies a hunk only if the
   base version is unchanged, else routes it to the review queue.
6. The job queue is **only** for worker jobs (dream, ingest). Synchronous MCP
   verbs (`remember`, `recall`, `get`) never touch it.

---

## 6. Module Decomposition

The §2.2 core lists seven modules. Assessment of cohesion / single-purpose /
testability:

| Module | Cohesion | Notes |
|--------|----------|-------|
| Markdown Store | good | CRUD + frontmatter parse + layout. Independently testable on a temp dir. Keep schema validation here. |
| Scoring Engine | good *if F-4.1 applied* | Pure function `f(I,R,recency)→rank`. Trivially unit-testable — *as long as it takes three floats and does not call QMD*. |
| Versioning | **doing two things** | git ops and the app-log are two different mechanisms (§6.3) at two granularities. Split into `GitVersioning` and `AppLog` — they have different failure modes, different tests, and git is opt-in while the app-log is always-on. [MEDIUM] |
| Access Control | good | Pure policy: `(identity, capability, scope, visibility) → allow/deny`. Keep it a pure module, no I/O. |
| Dreaming Orchestrator | good *if F-4.3 applied* | Scheduling + queue + merge. No LLM. Testable with a fake worker. |
| Plugin Host | good | Load, transport, health, supervise. Testable with fake plugins. |
| API Surface | **doing two things** | §2.2 bundles "MCP server + dashboard REST/WS". These are two adapters over one core service layer. Introduce an internal `CoreService` facade; MCP and REST/WS are thin transports over it. Prevents the dashboard (v2) and MCP from drifting. [MEDIUM] |

**F-6.1 — Add an explicit `CoreService` application layer. [HIGH]** Right now
verbs are described as "the MCP server enforces access control" — putting
policy in the transport. Access control, OCC, schema validation, and
orchestration belong in a transport-agnostic service layer; MCP, CLI, and the
v2 REST/WS API are three thin adapters over it. This is the single change that
keeps the v2 dashboard from re-implementing logic. The spec's §2.2 box should
show `CoreService` between `API Surface` and the rest of the core.

**F-6.2 — "Privacy filter" has no home. [MEDIUM]** §4.1's secret-stripping
filter is mentioned but assigned to no module. It runs on the capture
ingest path. Make it an explicit core component (`CaptureIntake`) — it is
security-critical and must not live inside a swappable plugin (a third-party
capture plugin must not be trusted to strip secrets).

Overall the decomposition is good — two modules to split (`Versioning`,
`API Surface`), one layer to add (`CoreService`), one component to name
(`CaptureIntake`).

---

## 7. Complexity — Speculative / Over-Built for v1

**F-7.1 — `relations` taxonomy is over-specified for v1. [MEDIUM]** Six typed
relation kinds (`derived_from|extends|contradicts|uses|replaces|related_to`)
plus emergent-entity detection (§5.2 step 2). v1 dreaming will not reliably
populate six semantically-distinct edge types. Ship v1 with `related_to` +
`derived_from` (the two the pipelines actually produce — links and provenance);
add the rest when dreaming demonstrably needs them. Reduces scoring/graph code
branching now.

**F-7.2 — `confidence` field is unused in v1. [LOW]** §3.4 has `confidence`;
§5.2 says explicit-flag confidence boosting is "v2 refinement." A frontmatter
field with no v1 reader is speculative. Keep it in the schema (cheap, forward-
compatible) but the spec should mark it explicitly "v1: stored, not consumed."

**F-7.3 — `mode: cross-agent` dreaming is a v2 surface in a v1 object. [MEDIUM]**
Cross-agent dreaming (§5.1) needs the `cross-agent` capability, cross-scope
access reasoning, and conflict handling across agent boundaries. v1 should ship
`per-agent` only and defer `cross-agent` with the dashboard. The dreaming-memory
schema can keep the `mode` field; the *implementation* of `cross-agent` is v2.

**F-7.4 — Three lifecycle states may be two. [LOW]** `active → dormant →
archived`. `dormant` (searchable on demand) and `archived` (cold partition)
differ only by which retrieval default includes them. v1 could ship
`active`/`dormant` and treat "archived" as a `dormant` memory in a cold folder
— but the cold-partition move is cheap and the three-state model is in the
source material. Low priority; flag only.

**F-7.5 — Not over-built: the seams.** The four plugin seams are sometimes a
complexity smell, but here they are justified — each has ≥2 real
implementations on the horizon and the Kuzu post-mortem makes the retrieval
seam concrete insurance. No change. The microkernel is the right amount of
structure; the *contracts* are simply unfinished, which is the opposite
problem from over-abstraction.

**F-7.6 — App-log + git is justified, not redundant. [LOW]** Two version
mechanisms looks like over-build, but they serve different granularities
(field-level provenance vs store-wide history) and git is opt-in. Keep — just
split the module (F-6.1 / table row).

---

## 8. Consolidated Findings

| ID | Severity | Issue | Recommendation |
|----|----------|-------|----------------|
| F-1.2 | CRITICAL | `staging/` lifecycle undefined — data loss on rejected dream | Append-only per-session JSONL; truncate only on dream *merge* (Flow A) |
| F-2.1 | CRITICAL | Plugin contract has no lifecycle | Add `PluginLifecycle` (init/health/shutdown) to all 4 |
| F-2.2 | CRITICAL | No plugin error semantics | `PluginError` kinds; index/graph failures non-fatal, search/llm surfaced |
| F-2.4 | CRITICAL | `CapturePlugin.onObservation` callback can't work cross-process | Redefine: install hooks + normalise wire payload; daemon exposes intake endpoint |
| F-3.1 | CRITICAL | MCP verbs have no I/O schemas | Adopt the 16-verb contract table (§3.4) |
| F-3.2 | CRITICAL | Capability-gating asserted, not designed | 5-capability model + per-verb mapping (§3.3) |
| F-5.1 | CRITICAL | Job queue is a phrase, not a design | Adopt `jobs` table + state machine (§5.1) |
| F-5.2 | CRITICAL | Worker death mid-job orphans the job | Heartbeat + reaper → `orphaned` → retry |
| F-1.1 | HIGH | No canonical data-flow section | Adopt Flows A–E (§1.2) |
| F-1.3/F-4.4 | HIGH | Ingest trigger undefined + mis-placed in core | Worker job on the shared queue (Flow B) |
| F-1.4 | HIGH | recall QMD/graph fusion undefined | QMD = Relevance; core fuses I×R×Recency; graph = opt-in expansion |
| F-1.5 | HIGH | Dream job has no state machine | Adopt §5.1 state machine |
| F-2.3 | HIGH | Plugin contract itself unversioned | `contractVersion`, checked at init |
| F-2.6 | HIGH | Retrieval plugin has no deindex/update | Add `update`/`deindex` |
| F-2.7 | HIGH | OQ-A unresolved | One contract, two transports per-plugin (§2.2) |
| F-3.3 | HIGH | Verb set incomplete/incoherent | Split get/list; add update, dream.list, dream.configure, system.status |
| F-3.4 | HIGH | OQ-C blocks recall schema | id+summary default, `expand` opt-in |
| F-4.1 | HIGH | Scoring appears coupled to retrieval | Scoring takes `relevance:number`, never imports the plugin |
| F-4.3 | HIGH | Orchestrator/worker labour split undrawn | Orchestrator: queue+schedule+merge; worker: all LLM, branch+manifest only |
| F-5.3 | HIGH | Dream-branch merge can race agent writes | Manifest carries base versions; OCC-validate each hunk at merge |
| F-5.4 | HIGH | No transaction story across md/git/log/index | Markdown authoritative; others derived, reconciled on startup |
| F-6.1 | HIGH | No transport-agnostic service layer | Introduce `CoreService` facade |
| F-1.6 | MEDIUM | Branch→merge has no actor | Worker writes manifest; orchestrator merges |
| F-1.7 | MEDIUM | recency-touch makes recall a versioned write | recency/access → `.engram/stats.db` sidecar, not git/app-log |
| F-2.5 | MEDIUM | Plugin takes full `Memory` objects | Pass `MemoryRef` (id+path+frontmatter) |
| F-6.2 | MEDIUM | Versioning module does two jobs | Split `GitVersioning` / `AppLog` |
| F-6.2b | MEDIUM | Privacy filter unhomed | Explicit core `CaptureIntake` component |
| F-7.1 | MEDIUM | 6 relation kinds over-specified for v1 | v1: `related_to`+`derived_from` only |
| F-7.3 | MEDIUM | cross-agent dreaming is v2 logic in v1 | Ship per-agent only; defer cross-agent |
| F-7.2/7.4/7.6 | LOW | confidence unused; 3 lifecycle states; dual versioning | Keep; annotate v1 behaviour |

---

## 9. Recommended Spec Amendments (priority order)

1. **Add §4.0 "Information Flow"** — Flows A–E from §1.2 of this review.
2. **Replace §2.4** with the hardened plugin contract (§2.3) and close OQ-A
   with the two-transport decision.
3. **Replace the §4.4 verb table** with the 16-verb MCP contract (§3.4) and add
   the capability model (§3.3); close OQ-B, OQ-C, OQ-E.
4. **Add §5.x "Job Queue & Worker Protocol"** — the `jobs` schema, state
   machine, heartbeat/reaper, merge OCC rule; close OQ-I.
5. **Amend §5** with the explicit orchestrator/worker labour split (F-4.3) and
   move ingest to a worker job (F-4.4 / Flow B).
6. **Amend §2.2** — add `CoreService` and `CaptureIntake`; split `Versioning`
   into `GitVersioning` + `AppLog` in the module list.
7. **Add a transaction/recovery invariant** (F-5.4): Markdown authoritative,
   all else derived and reconciled at startup.
8. **Trim v1 scope** (F-7.1, F-7.3): two relation kinds, per-agent dreaming
   only; annotate `confidence` as stored-not-consumed.

The macro-architecture needs no change. The work is finishing the four
contracts — plugin, MCP, job-queue, data-flow — all four supplied above for
direct lifting into the spec. Resolving OQ-A/C/E/I here removes the four open
questions that genuinely block the implementation plan; OQ-B is folded into
§3.4, OQ-F into Flow A (staging searched by the worker, not QMD-indexed — the
spec's lean holds).
