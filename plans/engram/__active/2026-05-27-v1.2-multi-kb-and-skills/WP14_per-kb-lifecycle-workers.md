---
name: wp14-per-kb-lifecycle-workers
title: Per-KB lifecycle workers (`kb.daily.ingest`, `kb.recall.rollup`, `kb.connect.bridge`)
type: work-package
stage: spec
severity: HIGH
created: 2026-05-27
updated: 2026-05-27
plan: 2026-05-27-v1.2-multi-kb-and-skills
tags: [kb, jobs, workers, lifecycle]
relationships:
  - blocked-by: [[wp13-multi-kb-registry-and-kbplugin-seam]]
  - blocks: [[wp15-cross-kb-bridges-derived-graph]]
sources: [SRC-01]
---
<!-- Template: WP v2 (frontmatter-first) -->

# WP14: Per-KB lifecycle workers

## Problem

Each KB type carries its own periodic work — daily ingest, recall rollup,
bridge build, KB-specific archival (SessionEnd for `agent-self`). Spinning up
a new orchestrator daemon, or fragmenting schedules across OS cron, doubles
the failure surface and breaks AppLog observability. ADR-0006 reuses the
existing dream-job state machine (SPEC §5.4) with new job kinds.

WP14 adds the new job kinds, the per-KB scheduler that enqueues them from
registry rows, and the CLI for ad-hoc runs. All v2.1 worker invariants stay:
detached process, schema-validated output (§9.4), rate limits (§9.5 R-5
active-pool floor), AppLog audit.

## Target Files

- `src/core/jobs/kinds.ts` — extend with `kb.daily.ingest`, `kb.recall.rollup`,
  `kb.connect.bridge`, `kb.lifecycle.archive`, `kb.<type>.<custom>`.
- `src/core/jobs/scheduler.ts` — registry polling loop (60 s cadence); enqueue
  per-KB jobs at the right time.
- `src/worker/kb/ingest.ts` — implements `kb.daily.ingest`.
- `src/worker/kb/rollup.ts` — implements `kb.recall.rollup`.
- `src/worker/kb/archive.ts` — implements `kb.lifecycle.archive`.
- `src/core/storage/sqlite.ts` — add `kb_id` column to `jobs` (nullable).
- `src/cli/kb.ts` — extend with `engram kb run <kb> <job-kind>` and
  `engram kb status <kb>`.
- `src/core/doctor/kb-checks.ts` — per-KB health checks for `engram doctor`.
- `tests/unit/jobs/kb-scheduler.test.ts`, `tests/integration/kb-lifecycle.test.ts`.

## Verification Gate

| # | Check | Test |
|---|-------|------|
| 1 | `kb.daily.ingest`, `kb.recall.rollup`, `kb.connect.bridge` each respect SPEC §5.4 state machine, retry policy, and AppLog audit identically to dream jobs. | `tests/integration/kb-lifecycle-state-machine.test.ts` |
| 2 | Per-KB schedule edits in registry row take effect within 60 s without daemon restart. | `tests/integration/kb-schedule-hot-reload.test.ts` |
| 3 | A `kb.*` job crash never touches the core daemon or an agent session. | `tests/integration/kb-worker-crash-isolation.test.ts` |
| 4 | Active-pool floor (R-5) applies per-KB to `kb.lifecycle.archive`; cannot archive below `min(100, 20% of KB total)`. | `tests/integration/kb-archive-floor.test.ts` |
| 5 | `engram kb run <kb> <kind>` enqueues an ad-hoc run that lands in the same queue as scheduled jobs. | `tests/integration/kb-run-adhoc.test.ts` |
| 6 | `engram kb status <kb>` shows last-run, queue depth, and doctor checks; backed by AppLog. | `tests/integration/kb-status.test.ts` |
| 7 | `kb.lifecycle.archive` for an `agent-self` KB runs at SessionEnd and hard-transitions Contextual memories per R-2. | `tests/integration/kb-agent-self-sessionend.test.ts` |

## Implementation Steps

| Step | File | State |
|------|------|-------|
| Add `kb_id` column to `jobs` + migration | `src/core/storage/sqlite.ts` | TODO |
| Extend job kinds enum + dispatcher | `src/core/jobs/kinds.ts`, `src/core/jobs/dispatcher.ts` | TODO |
| Scheduler loop reads registry, enqueues | `src/core/jobs/scheduler.ts` | TODO |
| Implement worker entries for new kinds | `src/worker/kb/*` | TODO |
| CLI ad-hoc run + status | `src/cli/kb.ts` | TODO |
| Doctor per-KB checks | `src/core/doctor/kb-checks.ts` | TODO |
| Unit + integration tests | `tests/{unit,integration}/**` | TODO |

## Verified Evidence

— (none yet — WP in `spec` stage)

## Agents

| Stage | Agent | Reason |
|-------|-------|--------|
| impl | `typescript-pro` | jobs + worker harness |
| review | `code-reviewer` | reuse-not-fork enforcement |

## Review

| Reviewer | Verdict | Notes |
|----------|---------|-------|
| — | — | — |

## Follow-up Issues

| Issue | Severity | Status |
|-------|----------|--------|
| — | — | — |
