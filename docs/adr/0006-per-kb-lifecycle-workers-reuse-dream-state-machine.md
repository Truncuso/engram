# ADR-0006: Per-KB lifecycle workers reuse the dream-job state machine

- **Status:** Accepted
- **Date:** 2026-05-27
- **Related:** SPEC v2.2 §15.3; SPEC v2.1 §5 (dreaming) and §5.4 (job state machine); ADR-0001 (detached worker); ADR-0005 (KbPlugin)

## Context

ADR-0005 introduces multi-KB orchestration. Each KB type carries its own
periodic work — a `raw-sources` KB needs daily ingest of new files, a
`markdown-store` needs weekly recall rollup and cross-KB bridge refresh, an
`agent-self` KB needs SessionEnd consolidation, an `obsidian-vault` needs a
Dataview index rebuild.

Three obvious options:

1. A new "KB orchestrator" daemon process alongside engramd and the dreaming
   worker, with its own queue and supervisor.
2. Per-KB cron in user `crontab` / systemd timers.
3. Reuse the existing SQLite-backed dream-job state machine from SPEC §5.4.

Option 1 doubles the failure surface and re-implements the job state machine
engram already specifies (queued → running → succeeded / failed / cancelled,
with retry policy, rate limits, and AppLog audit). Option 2 fragments
observability — failures live in syslog, not AppLog — and breaks Windows
without manual setup. Option 3 reuses what engram already has.

## Decision

Per-KB lifecycle workers are **the same workers**, running **the same state
machine**, against **the same `jobs` table**, with new job *kinds*:

- `kb.daily.ingest` — pull new files into a KB and stage them for the worker.
- `kb.recall.rollup` — produce a per-KB recap of recently-accessed memories.
- `kb.connect.bridge` — rebuild cross-KB bridges (see ADR-0007).
- `kb.lifecycle.archive` — KB-specific archival (e.g., Contextual at SessionEnd).
- `kb.<type>.<custom>` — namespaced job kinds declared by individual KbPlugins
  via `lifecycleJobs()`.

Per-KB schedules live in the registry row (`config.lifecycle = { ingest: "0 4
* * *", rollup: "0 6 * * 1", bridge: "0 5 * * 0" }`). The daemon enqueues at
the right time; the existing detached worker dequeues. No new process, no new
supervisor, no new queue.

Two invariants from SPEC v2.1 stay enforced:

- **The worker is detached.** A `kb.*` job crash cannot touch the core daemon
  or the agent's session (SPEC §5, ADR-0001).
- **Worker output is schema-validated.** `kb.*` jobs that mutate memory
  contents go through the same JSON-schema validation as `dream.*` jobs
  (SPEC §9.4). A KB-job-emitted memory that injects forbidden frontmatter
  fields is rejected; the job FAILS.

A new CLI: `engram kb run <kb> <job-kind>` enqueues an ad-hoc run for a
specific KB instance. `engram kb status <kb>` shows the schedule, last-run
results, and queue depth — backed by the existing AppLog.

## Consequences

- No new daemon, no new queue, no new state machine. The `jobs` table grows a
  `kb_id` column (nullable; null = dream job, set = KB job).
- KB-specific work shows up in the same AppLog and `engram doctor` health
  surface as dreaming runs.
- Per-KB rate limits (e.g., max 5% of memories archived per run, SPEC §9.5
  R-5) apply unchanged — the per-KB worker reads from the same scoring engine
  and respects the same forgetting safety rails.
- Schedules are user-editable per KB without daemon restart (the daemon
  polls the registry row on a 60 s cadence).

## Alternatives considered

- **Separate KB orchestrator daemon.** Larger blast radius; duplicates state
  machine; rejected.
- **OS-level cron / systemd timers.** Splits failure observation between
  engram AppLog and OS logs; rejected.
- **One unified "lifecycle" job per KB.** Bundling ingest + rollup + bridge
  into one job makes partial failure ambiguous (did rollup run if ingest
  failed?). Rejected — keep them separate so failures isolate.
