---
title: "engram — Failure Behaviour & Safety Review"
project: engram
created: 2026-05-22
author: Claude (debugger agent)
spec_version: SPEC.md (design approved 2026-05-22)
type: design-review
---

# engram — Failure Behaviour & Safety Review

Design-level reliability audit of SPEC.md. No code exists yet. Each section
names the failure mode, the current spec gap, and the recommended behaviour the
spec must define before implementation begins.

---

## 1. Recall Failure

### Failure modes
- QMD process is down or failed to start.
- Index is corrupt (SQLite FTS5 or sqlite-vec tables are inconsistent).
- Index is stale — file on disk updated since last reindex.
- Query times out (embedding call hangs; sqlite-vec scan too slow on large stores).
- Query returns empty — either correctly (no match) or incorrectly (index gap).
- graphify subprocess unreachable; graph traversal unavailable.

### Current spec gap
The spec states QMD is the retrieval plugin and that the index is "derived" and
"rebuildable" (§2.4, §8.1). It does not define what `memory.recall` returns or
does when QMD is unavailable, what the timeout budget is, or whether there is
any fallback.

### Recommended behaviour

**Fallback chain** (ordered by cost/reliability; every tier is optional but the
chain must be defined):

```
1. QMD vector+BM25 hybrid  (primary)
2. QMD BM25 only           (if vector index corrupt or embedding call fails)
3. Filesystem grep         (if QMD is down — scan memories/ with ripgrep/grep)
4. Return partial + warn   (if grep also fails — return what we have, attach
                            degraded:true + reason to the response)
```

`memory.recall` must **never fail hard** by default; it must degrade
gracefully and signal degradation in the response envelope:

```ts
interface RecallResponse {
  hits: ScoredHit[];
  degraded?: { reason: string; tier_used: "vector+bm25" | "bm25" | "grep" | "none" };
}
```

**Timeout policy**: each tier has a hard timeout (suggested: vector+BM25 300ms,
BM25 150ms, grep 2000ms). If a tier times out, fall through; log at WARN level.

**Stale index**: after any write (`memory.remember`, ingest, dreaming merge),
the kernel must enqueue an async reindex of the affected memory. `recall` issued
within the reindex window returns the pre-update result plus `stale:true` if the
memory was written within the last N seconds and the index has not caught up.
This is explicit and observable, not silently wrong.

**Empty result**: distinguish "no memories match" (correct empty) from "index
is empty because rebuild has not run" (incorrect empty). The latter is detected
by checking the index row count; if zero and `memories/` is non-empty, return
`degraded: index_empty` and schedule an immediate rebuild.

---

## 2. Retry Logic

### Failure modes
- LLM API call (dreaming consolidation, ingest distillation) fails: network
  error, 429 rate-limit, 5xx, timeout, context-window overflow.
- Embedding API call fails: same classes.
- OCC write conflict: `version` mismatch on `memory.remember`.
- Plugin call (graphify subprocess) fails or returns error exit code.
- QMD reindex call fails.
- `git commit` in the dreaming worker fails.
- App-log append fails.

### Current spec gap
OCC conflict is mentioned — "rejected and retried" (§6.2) — but no retry
policy is defined. LLM/embedding retries are not mentioned at all. The spec
says dreaming works on a git branch but does not define what happens when the
commit itself fails.

### Recommended behaviour

**Unified retry policy** for all retryable operations:

```
Base:         exponential backoff with full jitter
              delay = random_between(0, min(cap, base * 2^attempt))
              base = 200ms, cap = 30s, max_attempts = 5
Rate-limit:   honour Retry-After header if present; use it as the floor
Budget:       LLM/embedding retries count against the dreaming job's token
              budget; abort job if budget exhausted before success
Idempotency:  every write operation must be idempotent so retries are safe
              (see §7 for file-write atomicity)
```

**Per-operation specifics**:

| Operation | Retryable | Max attempts | Notes |
|-----------|-----------|--------------|-------|
| LLM `complete` (dreaming) | yes | 5 | non-retryable on 400/context overflow; abort job |
| LLM `embed` | yes | 5 | same |
| OCC write conflict | yes | 10 | no backoff needed — re-read fresh state, immediate retry; if 10 consecutive conflicts, log and surface to caller |
| graphify subprocess call | yes | 3 | backoff; if 3 failures: mark graph plugin degraded, continue without graph |
| QMD reindex | yes | 3 | on 3 failures: schedule full rebuild; set stale flag |
| git commit (dreaming) | yes | 3 | on 3 failures: abort dream job, persist job-state = failed, leave branch for inspection |
| app-log append | yes | 5 | write to a recovery spool if main log unavailable (see §6) |

**Non-retryable conditions**: 400 Bad Request from LLM (bad prompt), context
window overflow (must chunk differently — not a transient failure), plugin
crash with exit-code 2 (unrecoverable configuration error).

---

## 3. Dreaming Worker Failure

### Failure modes
- Worker crashes (SIGSEGV, unhandled exception, OOM) mid-run.
- Worker runs out of token/cost budget partway through.
- Worker produces a bad merge: overwrites memories with corrupt frontmatter,
  inserts wrong relations, corrupts `lifecycle` fields.
- Worker hangs: never exits; the job queue shows it as running indefinitely.
- Worker is killed externally (SIGKILL from OOM killer or admin).
- Worker leaves a dangling `dream/<name>/<ts>` branch.
- Worker partially writes: some memories updated on the branch, then crash —
  the branch is in a half-baked state.
- Two dream jobs for the same dreaming-memory run concurrently.

### Current spec gap
The spec guarantees process isolation ("a crash never touches the core daemon",
§5.5) but says nothing about: the job state machine, recovery after a crash,
what happens to a dangling branch, how the orchestrator detects a hung worker,
or how the merge step is validated before merging to main.

### Recommended behaviour

**Job state machine** (persisted in SQLite, `.engram/jobs.db`):

```
QUEUED
  → SPAWNED         (worker PID recorded)
  → RUNNING         (worker sends heartbeats every 30s to jobs table)
  → COMPLETED       (worker exited 0; branch ready for merge)
  → FAILED          (worker exited non-zero or crash detected)
  → TIMED_OUT       (watchdog: no heartbeat for 5 min → mark timed-out)
  → MERGING         (merge in progress — brief; set by orchestrator)
  → MERGED          (branch merged to main, branch deleted)
  → REVIEW_PENDING  (gated ops queued; human must act before MERGED)
  → ABORTED         (budget exhausted; partial branch preserved)
```

State is written by the worker (heartbeat + terminal state) and by the
orchestrator (QUEUED → SPAWNED, timeout detection, merge phase). The two
writers are disjoint — no concurrency conflict.

**Crash recovery** (orchestrator startup + watchdog loop):

1. On `engramd` start: scan jobs table for SPAWNED or RUNNING jobs whose PID
   is no longer alive. Transition them to FAILED.
2. A watchdog timer (60s interval) checks heartbeat timestamps for RUNNING jobs.
   Jobs with no heartbeat for >5 min are transitioned to TIMED_OUT.
3. FAILED and TIMED_OUT jobs: preserve their branch for inspection; do not
   auto-delete; emit a `dream.status` event the CLI/dashboard can surface.
4. A FAILED or TIMED_OUT job may be re-queued by the user via `dream.trigger
   --resume <job_id>`. The worker resumes from the last committed checkpoint
   on the branch (see idempotent re-run below).

**Idempotent re-run**: the dream branch records a checkpoint file
(`.dreaming/.checkpoint/<job_id>.json`) listing completed stages (distill /
connect / re-weight / verify). On re-spawn, the worker reads the checkpoint and
skips completed stages, resuming from the first incomplete one. This makes
re-run safe and cheap.

**Hung worker / timeout**:
- Worker must emit a heartbeat row update every 30 seconds while actively
  processing.
- Watchdog: if no heartbeat for 5 minutes, SIGTERM the PID (if still alive),
  wait 10s, SIGKILL if needed, transition job to TIMED_OUT.
- Budget exceeded: worker checks remaining budget before each LLM call; if
  exhausted, write a partial checkpoint, set job state to ABORTED, exit 0.
  Partial results on the branch are still valid and mergeable for completed
  stages.

**Merge validation before touching main**:
Before merging the dream branch to main, the orchestrator runs a validation
pass:
1. Parse every modified memory file's frontmatter — reject any with YAML
   parse errors.
2. Check `version` tokens: the branch must not have decreased any version
   relative to what main had when the job started (detect stale-read
   overwrites if main moved during the dream).
3. Check lifecycle transitions: reject any transition that skips states
   (e.g., `active` → `archived` in one step is only legal if the scoring
   model explicitly allows it; otherwise flag for review).
4. If validation fails: do not merge; set job state to FAILED; attach
   validation error to the job record.

**Dangling branch cleanup**: a background task runs weekly (or on `engramd`
start) and prunes `dream/` branches older than 30 days whose job state is
MERGED. FAILED/TIMED_OUT branches are retained for 90 days for forensics, then
pruned with a log entry.

**Concurrency guard**: the orchestrator must enforce that at most one dream job
per dreaming-memory name runs at a time. A QUEUED job for a name whose previous
job is still RUNNING/SPAWNED must wait. Use the jobs table as a mutex (SQLite
`SELECT ... FOR UPDATE` equivalent via a `running` flag column per
dreaming-memory name).

---

## 4. Capture Layer Failure

### Failure modes
- The capture hook process (writing to `staging/`) crashes or throws.
- The hook is slow — it blocks the host session's tool execution.
- `engramd` is not running when the hook fires.
- `staging/` fills up — disk full, or dreaming has not run for a long time.
- A hook fires concurrently with another hook for the same session (parallelism
  in the harness).
- The privacy filter fails — a hook observation with secrets passes through.

### Current spec gap
The spec says capture hooks record to `staging/` (§4.1) and mentions a privacy
filter. It does not define: the hook's timeout budget, the fire-and-forget
contract, backpressure, behaviour when `engramd` is down, or staging/ size limits.

### Recommended behaviour

**Fire-and-forget contract** (hard requirement):

The capture hook must be implemented as a best-effort, non-blocking write. The
host agent's session must never be stalled or failed because of memory capture.
Implementation: the hook writes to `staging/` directly (a local file write, not
an RPC call), with a hard timeout of 200ms. If the write exceeds 200ms or
fails for any reason, the hook exits 0 silently and the observation is dropped.
The hook must not attempt to connect to `engramd` at capture time — staging/ is
the decoupling boundary.

**When `engramd` is down**: no problem; hooks write to `staging/` directly. The
daemon processes staging/ when it comes back up. Staging/ is the explicit buffer
designed for this.

**Staging/ backpressure**: define a max staging/ size (default: 500 MB, or 10,000
files — configurable). When the threshold is reached:
1. New hook observations are silently dropped (fire-and-forget; the host session
   is unaffected).
2. `engramd` logs a WARN and emits a `staging.backpressure` metric.
3. The dreaming orchestrator is nudged to trigger an emergency distillation run
   if one is not already queued.

**Hook timeout** (distinct from capture timeout): if the hook child process
(spawned by the harness) does not exit within 500ms, the harness's hook runner
should kill it. This is a harness-side concern; the spec should document the
expected hook exit time budget so harness adapters implement it correctly.

**Concurrent hooks**: multiple hooks may fire in the same session (e.g.,
`PostToolUse` and `Stop` overlap). Each hook write to staging/ must use an
atomic file write (write to `.tmp` then rename) to avoid partial files being
picked up by the dreaming worker. File names include session ID + timestamp +
random suffix to avoid collisions.

**Privacy filter failure**: if the privacy filter itself throws, the hook must
drop the observation entirely (fail-closed). The observation must not be written
to staging/ without having passed the filter. Log the filter failure at ERROR
level (without the payload content) so it is detectable.

---

## 5. Indexing / Staleness

### Failure modes
- A memory file is written but QMD reindex is not triggered (engramd crash
  between write and reindex).
- A memory file is deleted or moved (by dreaming reorganization) but the index
  still has the old entry.
- graphify graph is rebuilt asynchronously; graph queries return stale edges.
- A full rebuild is triggered on a large store and takes minutes; concurrent
  recalls during rebuild get inconsistent results.
- QMD index file (SQLite) is corrupt or locked.

### Current spec gap
The spec says indexes are "derived" and "rebuildable" (§2.4) and references the
"session-start staleness" pattern (§8.1, graphify integration note). It does not
define: when reindex is triggered, how staleness is detected, what happens
during a rebuild, or how index corruption is handled.

### Recommended behaviour

**Write-through reindex**: every `memory.remember`, ingest completion, and
dreaming merge triggers an async reindex of affected memory files. This is a
fire-and-forget enqueue into a reindex queue (SQLite table); the reindex worker
drains the queue. Queue entries are idempotent (deduped by memory id + mtime).

**Staleness detection**: the kernel tracks a `last_reindexed` timestamp per
memory (stored in `.engram/index-state.db`). On `recall`, for each result hit,
compare `last_reindexed` against the file's mtime. If mtime > last_reindexed,
annotate the hit with `stale:true`. The caller can use this to decide whether
to re-fetch.

**Full rebuild cost and lockout**: define a rebuild is triggered by: (a) explicit
`engram index rebuild`; (b) health-check detecting index corruption; (c)
`engramd` start if the index is older than 24 hours and there are un-indexed
writes. During rebuild: QMD is flagged as `rebuilding`; the recall fallback chain
(§1) activates automatically, falling back to BM25-only or grep as needed.

**graphify staleness**: graphify is rebuilt via the session-start hook
(background). The spec should define: if graph.json mtime < newest file mtime in
memories/, schedule a background rebuild. Graph queries during a rebuild return
stale results; `memory.recall` annotates graph-based hits with
`graph_stale:true`.

**Index corruption**: if QMD's SQLite file fails to open or a query returns a
SQLite error, the kernel marks QMD as degraded and activates the fallback chain.
It schedules a `DROP TABLE + rebuild` on the next maintenance window (or
immediately if `engramd` is otherwise idle). The corrupt index is backed up
before dropping (`.engram/index.db.corrupt.<ts>`).

---

## 6. Versioning Failure

### Failure modes
- The file write succeeds but the subsequent `git commit` fails (git lock,
  corrupt refs, disk full).
- The git commit succeeds but the app-log write fails.
- The app-log write succeeds but the file write failed (write-then-log ordering
  inversion).
- Two concurrent writes race on the same file despite OCC (OCC is implemented
  at the application layer, not at the filesystem layer).
- A git rebase/merge by dreaming produces a conflict on a file that the core
  daemon also modified since the dream started.

### Current spec gap
The spec defines both versioning mechanisms (§6.3) but does not specify the
ordering or atomicity guarantee between the three operations: file write, git
commit, and app-log entry. There is no defined recovery path for partial
sequences.

### Recommended behaviour

**Canonical ordering** (must be enforced by the write path):

```
1. Write memory file atomically (tmp → rename)
2. Append app-log entry            ← if this fails, the file write is the truth;
                                      app-log is rebuilt from git blame on demand
3. git commit (if git-mode)        ← if this fails, retry 3x with backoff;
                                      if all fail, mark the commit as pending
                                      in a recovery spool
```

Rationale: the Markdown file is the source of truth (§3.2). The app-log and
git are derived provenance. Failing to record provenance does not corrupt the
memory itself.

**App-log failure recovery**: if the app-log append fails (disk full, locked),
write the entry to a recovery spool (`.engram/applog-recovery.jsonl`). On next
`engramd` start (or on a background recovery task), drain the spool into the
main app-log. Mark the entry with `recovered:true` so history consumers know
it may have been replayed out-of-wall-clock order.

**Git commit failure recovery**: pending commits are recorded in
`.engram/git-pending.jsonl`. The git recovery task retries them in order on
`engramd` start and periodically (every 5 min). If a pending commit cannot be
applied (the file has since changed again), the newer state wins; the pending
commit is dropped with a log entry (the newer commit covers it anyway).

**Write-time atomicity**: the file write itself must be atomic: write to a
`.tmp` file in the same directory, `fsync`, then `rename`. On POSIX systems this
is atomic at the filesystem layer. On Windows, use a move with replace. This
ensures no reader sees a partial file.

**Dreaming merge conflict with core writes**: the merge validation step (§3)
must detect when a file on the dream branch was also modified on main after the
dream started. The strategy: use the `version` field. If the dream branch has
`version: N` for a memory and main now has `version: M > N`, the dream branch's
write for that memory is stale. The orchestrator must re-apply the dreaming
changes on top of the current main version (a 3-way merge at the field level)
or queue the conflict for human review if field-level merge is ambiguous.

---

## 7. Data Integrity

### Failure modes
- A memory file has broken YAML frontmatter (mid-write crash, manual edit
  introducing a syntax error).
- A memory file has no `id` field, or a duplicate `id` exists in two files.
- A `relations:` entry points to a memory `id` that does not exist in the store
  (deleted or was never created).
- A `relations:` entry points to a memory whose `lifecycle` is `archived`
  (dormant reference — valid but should be surfaced).
- A `sources:` field points to a file in `raw/` that no longer exists.
- An `access_count` or `importance` value is outside its valid range.
- The app-log has entries for a memory id that no longer has a file.
- The QMD index has rows for memory ids that no longer exist as files.

### Current spec gap
The spec defines the frontmatter schema (§3.4) and states "nothing is ever
deleted" (§3.6), but it does not define health-check tooling, how the system
detects integrity violations, or how it self-heals.

### Recommended behaviour

**`engram doctor` command** (required in v1): a self-healing health-check that
the user can run manually and that `engramd` runs on startup (abbreviated) and
on a weekly schedule (full).

Checks performed:

| Check | Detection | Action |
|-------|-----------|--------|
| Broken frontmatter | YAML parse error | Log as ERROR; move file to `.engram/quarantine/`; do not process |
| Missing `id` | frontmatter parse | Assign a new ULID; write back; log as WARN |
| Duplicate `id` | id-to-path index | Keep the older file's id (by `created` timestamp); assign new id to newer; log as WARN |
| Dangling `relations:` edge | check each `to: mem_xxx` against id-to-path index | Log as WARN; if `--fix` flag: remove the dangling relation entry |
| `relations:` to archived memory | lifecycle check | Log as INFO; no auto-action (archived memories are still valid referents) |
| Missing `sources:` file | check each path in `raw/` | Log as WARN; if `--fix` flag: mark source as `missing` in frontmatter |
| Out-of-range field value | range check on `importance`, `decay`, `confidence` | Log as WARN; clamp to valid range if `--fix` |
| Orphaned app-log entries | cross-reference log ids against file ids | Log as INFO; retain (provenance value) |
| Index/file divergence | compare QMD row count + ids vs memory files | Schedule reindex of divergent set |
| Orphaned dreaming branches | git branch list vs jobs table | Log; user confirms pruning |

**Quarantine**: files with unfixable errors (unparseable frontmatter that cannot
be corrected automatically) are moved to `.engram/quarantine/` rather than
deleted. The user sees them in `engram doctor` output and can manually fix and
restore them.

**Startup abbreviated check**: on `engramd` start, run: broken frontmatter scan
(fast — just parse headers), duplicate id check, index count sanity. Full checks
(relation walking, source path validation) are deferred to the weekly run or
explicit `engram doctor` invocation.

---

## 8. Forgetting Safety

### Failure modes
- A bug in the re-weighting logic sets `importance: 0` for all memories in a
  dreaming run, causing mass transition to `dormant`.
- A bug in the lifecycle-advancement logic transitions active memories to
  `archived` in a single step, bypassing the `dormant` intermediate state.
- A cascading effect: one dream run archives 80% of the store because of a
  miscalibrated threshold.
- The `always-auto` merge policy (not the default, but configurable) auto-merges
  a mass-archive batch without human review.

### Current spec gap
The spec defines the lifecycle states and states that "dashboard-gated" merge is
the default for archiving (§5.3). But it does not define: maximum lifecycle
transitions per dream run, a rollback procedure for a bad dream merge, or
programmatic safeguards against mass-archive.

### Recommended behaviour

**Lifecycle transition rate limits per dream run** (enforced by the merge
validation step in §3):

```
Max memories transitioned active→dormant in a single dream run:
  min(50, 5% of total active memories)
Max memories transitioned dormant→archived in a single dream run:
  min(20, 2% of total dormant memories)
```

If a dream run's output exceeds these limits, the merge validation step must:
1. Downgrade the job's effective merge policy to `always-gated` for that batch,
   regardless of the configured `merge_policy`.
2. Surface the count in `dream.status` output with a WARN.
3. Require explicit human confirmation via `engram dream review --approve <job_id>`.

**`importance` floor**: the dreaming worker must not set `importance` to zero.
The valid range for dreaming-written importance is [0.05, 1.0]. A zero or
below-floor value is treated as a validation error in the merge step.

**`always-auto` guard**: even with `merge_policy: always-auto`, the rate-limit
check applies. `always-auto` only skips the human-review queue for normal-volume
runs; mass transitions are always gated.

**Dream rollback**: because dreaming writes to a branch that is only merged (not
force-pushed to main), rollback is a `git revert` of the merge commit. The spec
must document this as the first-response procedure for a bad dream. The
orchestrator must retain the dream branch after merging (do not delete it
immediately) for at least 7 days to enable rollback without branch reconstruction.
After a rollback, the job state is set to `ROLLED_BACK`; the dreaming-memory's
schedule is paused pending investigation.

**Human override for dormant revival**: if a mass-dormancy event occurs, the
user can run `engram lifecycle revive --since <ts>` to batch-transition memories
back to `active` that were dormanted after a given timestamp. This operation
is git-committed and app-logged.

---

## 9. Startup / Shutdown

### Failure modes
- `engramd` starts while a dream job is in RUNNING state (previous daemon
  crashed while a worker was running).
- `engramd` starts while git is locked (another process holds `.git/index.lock`).
- `engramd` receives SIGTERM while a dreaming merge is in progress (MERGING state).
- `engramd` crashes (panic/OOM) mid-request — an in-flight `memory.remember` is
  half-written.
- The MCP server port is already in use on startup.
- The QMD or graphify plugin fails to initialize on startup.

### Current spec gap
The spec mentions "daemon + MCP server" (§2.2) and process isolation for dreaming
(§5.5) but does not define startup sequence, graceful shutdown behavior, in-flight
request handling, or plugin initialization failure modes.

### Recommended behaviour

**Startup sequence** (ordered; each step is a gate):

```
1. Acquire PID lock (.engram/engramd.pid). If stale PID exists:
   check if process is alive; if not, overwrite. If alive, exit with error
   "engramd already running (pid X)".

2. Open jobs.db (SQLite). Run crash-recovery scan: SPAWNED/RUNNING jobs
   whose PID is not alive → transition to FAILED.

3. Run abbreviated data integrity check (§7 startup check).
   Log WARNs; do not block startup for non-critical issues.
   Quarantine any unparseable memory files found.

4. Initialize plugins in order: LLM plugin → Retrieval plugin (QMD) →
   Graph plugin (graphify subprocess). If a plugin fails to initialize:
   - LLM: FATAL — cannot function without LLM plugin. Exit with clear message.
   - Retrieval: WARN — start in degraded mode (grep fallback); schedule
     rebuild; emit degraded:true on all recall responses until healthy.
   - Graph: WARN — start without graph; graph-dependent features disabled.

5. Start MCP server. If port is in use: retry 3 times with 1s delay
   (handles slow previous-instance teardown). If still in use, exit with error.

6. Drain git-pending.jsonl and applog-recovery.jsonl (§6 recovery).

7. Check if dreaming schedule has any overdue jobs (last_run + schedule_interval
   < now). If yes, queue them immediately.

8. Log "engramd started" with version, store path, plugin health summary.
```

**Graceful shutdown** (SIGTERM handler):

```
1. Stop accepting new MCP requests (return 503 for new connections).
2. Allow in-flight requests to complete; wait up to 10 seconds.
3. For in-flight requests that have not completed after 10s: log their
   state and abort them. File writes that are in-progress but not
   committed will be detected by the startup crash-recovery.
4. Send SIGTERM to all RUNNING dream workers. Wait up to 30 seconds
   for them to checkpoint and exit. After 30s, SIGKILL and mark jobs
   TIMED_OUT.
5. Flush the app-log buffer; drain the reindex queue (best-effort, 5s max).
6. Release the PID lock.
7. Exit 0.
```

**Plugin health reporting**: expose `engram status` CLI command that reports:
```
engramd: running (pid 12345, uptime 3h 14m)
store:   ~/.engram/ (1,247 memories, 3.2 GB)
qmd:     healthy (last reindex: 4 min ago)
graphify: healthy (graph age: 2h 14m)
llm:     claude-3-7-sonnet (healthy)
jobs:    0 running, 2 queued, 1 review-pending
staging: 42 files (18 MB)
```

---

## Proposed Spec Section: Failure Behaviour & Safety

The following should be inserted as **§ 13** (or an appendix) in SPEC.md before
implementation work begins:

---

### 13.1 Recall Degradation Chain

`memory.recall` degrades gracefully through: vector+BM25 → BM25-only → grep →
partial+degraded. It never fails hard. All degradation is signalled in the
response via `degraded:{reason, tier_used}`. Stale results are flagged
`stale:true`. Each tier has a hard timeout; timeout triggers fallthrough.

### 13.2 Retry Policy

Exponential backoff with full jitter: `delay = rand(0, min(30s, 200ms * 2^n))`.
Max 5 attempts for LLM/embedding; 10 for OCC conflicts (immediate re-read, no
delay); 3 for plugin calls; 3 for git commits. Rate-limit responses honour
`Retry-After`. Non-retryable: 400 from LLM, context overflow, plugin exit 2.
All retryable operations are idempotent.

### 13.3 Dreaming Job State Machine

States: `QUEUED → SPAWNED → RUNNING → COMPLETED → MERGING → MERGED` (happy
path); `→ FAILED`, `→ TIMED_OUT`, `→ ABORTED` (budget), `→ REVIEW_PENDING`,
`→ ROLLED_BACK` (error paths). Persisted in `.engram/jobs.db`. Watchdog detects
hung workers (5 min no heartbeat → TIMED_OUT). Crash recovery on `engramd`
start. Idempotent resume from checkpoint. Max one running job per dreaming-memory
name. Dream branches retained 7 days post-merge for rollback; FAILED branches
retained 90 days.

### 13.4 Merge Validation

Before merging a dream branch: parse all modified frontmatter (reject YAML
errors); check version tokens for stale overwrites; check lifecycle transitions
against rate limits; check importance floor (0.05). Any validation failure →
job transitions to FAILED, branch preserved.

### 13.5 Forgetting Safety Rails

Per-dream rate limits: max `min(50, 5%)` active→dormant and `min(20, 2%)`
dormant→archived transitions. Excess triggers forced `always-gated` merge,
overriding configured policy. `importance` minimum is 0.05 (dreaming-written).
Rollback via `git revert` of the merge commit; branch retained 7 days.

### 13.6 Capture Fire-and-Forget Contract

Hooks write directly to `staging/` (no daemon RPC). Hard timeout: 200ms. On
any failure or timeout: exit 0 silently (host session unaffected). Privacy
filter failure is fail-closed (drop observation). Staging backpressure: at 500
MB or 10,000 files, new observations are dropped and an emergency dream is
queued.

### 13.7 Write Ordering & Atomicity

Canonical order: (1) atomic file write (tmp → rename), (2) app-log append, (3)
git commit. App-log failures spool to `.engram/applog-recovery.jsonl`, drained
on next start. Git commit failures spool to `.engram/git-pending.jsonl`. Source
of truth is always the Markdown file.

### 13.8 Data Integrity (`engram doctor`)

Abbreviated check on startup; full check weekly. Checks: frontmatter parse,
id uniqueness, dangling relations, missing sources, field ranges, index
divergence, orphaned branches. Unfixable files quarantined to
`.engram/quarantine/`. `--fix` flag enables auto-repair for safe issues.

### 13.9 Startup / Shutdown

Startup gates: PID lock → crash recovery → integrity check → plugin init (LLM
fatal; retrieval/graph degradable) → MCP bind → pending journal drain → schedule
backfill. Graceful shutdown: drain in-flight (10s) → checkpoint dream workers
(30s) → flush logs → release PID lock. `engram status` reports live health.

---

## 12-Line Critical Gap Summary

The following are the twelve most critical failure-mode gaps the spec must close
before implementation begins:

1. **No recall fallback chain defined.** `memory.recall` has no degraded-mode
   behaviour when QMD is unavailable; the system will silently return nothing
   or hard-error.

2. **OCC retry is mentioned but completely unspecified.** No backoff, no max
   attempts, no idempotency guarantee. Concurrent agents will loop indefinitely
   on a live conflict.

3. **Dream job has no state machine.** There is no persisted job state, so a
   crashed or hung worker is undetectable and unrecoverable without a restart.

4. **No dreaming merge validation step.** A dream branch with corrupt frontmatter
   or stale version tokens can be merged silently, overwriting correct state.

5. **Forgetting safety rails are absent.** No rate limit on lifecycle transitions
   per dream run; `always-auto` merge policy + a scoring bug can
   cascade-archive the entire store without any gate.

6. **No write ordering or atomicity guarantee.** The ordering between file write,
   app-log, and git commit is undefined; a crash between any two steps produces
   an ambiguous state with no defined recovery path.

7. **Capture hook failure contract is undefined.** Nothing in the spec specifies
   that hooks must be fire-and-forget with a timeout; a slow or crashing hook
   could block the host agent's session.

8. **Staging backpressure is not addressed.** No size limit or overflow policy
   for `staging/`; a dormant dreaming schedule + active capture can fill the
   disk with no circuit-breaker.

9. **`engramd` startup sequence is not defined.** On restart after a crash,
   there is no specified procedure for crash recovery, plugin initialization
   order, or handling of FAILED dream jobs from the previous run.

10. **No `engram doctor` / health-check concept.** Broken frontmatter, duplicate
    ids, and dangling `relations:` edges have no detection or remediation path
    in the current spec.

11. **Dreaming branch lifecycle is undefined.** No specification for when branches
    are deleted, how long FAILED branches are retained for forensics, or how
    rollback of a bad dream merge is performed.

12. **Index staleness is not actively managed.** There is no defined mechanism to
    detect or signal that a recall result is based on a stale index, and no
    trigger policy for reindex after writes — leaving the system open to
    silent data drift between the file store and the retrieval layer.
