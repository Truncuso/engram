---
name: phase-8a-job-state-machine-spawning
title: 8a — Job state machine + detached spawn + heartbeat + watchdog (zero LLM)
type: phase
phase_status: pending
wp: wp08-dreaming-worker-orchestrator
goal: The jobs.db state machine (§5.4) is driven end-to-end with a detached worker that heartbeats; atomic QUEUED→SPAWNED claim; watchdog reaps stale; idempotent resume; budget ceiling at spawn. No LLM calls.
verify: "npm test tests/integration/jobs — spawn a no-op worker, kill it, watchdog → TIMED_OUT; dream resume re-queues idempotently; a budget-exceeded trigger refuses to spawn; concurrent claims never double-spawn."
---
<!-- Template: WP-folder PHASE v2 (frontmatter-first) -->

# Phase 8a: Job state machine + spawning (zero LLM)

**Goal:** Drive the §5.4 state machine in `jobs.db`: atomic `QUEUED→SPAWNED`+pid
in one transaction (C7), worker advances `→RUNNING` on first heartbeat (30s),
watchdog reaps stale (>5min) → `TIMED_OUT`, budget hard-ceiling sealed at spawn
(S-10), idempotent resume from checkpoint. A detached no-op worker proves
spawn/heartbeat/reap/resume with **zero LLM calls**. Crash isolation: killing the
worker leaves the daemon healthy.

**Verify:** `npm test tests/integration/jobs` — spawn no-op worker → kill →
watchdog `TIMED_OUT`; `dream resume` idempotent re-queue; budget-exceeded refuses
spawn; concurrent claims single-flight (no double-spawn).

## Steps

| Step | File | State |
|------|------|-------|
| jobs.db state transitions + single-flight per dreaming-memory | `src/core/orchestrator/queue.ts` | TODO |
| Atomic QUEUED→SPAWNED + pid (one txn) — C7 | `src/core/orchestrator/claim.ts` | TODO |
| Detached spawn (`child_process` detached); PID liveness via /proc cmdline | `src/core/orchestrator/spawn.ts` | TODO |
| Worker heartbeat (30s) + checkpoint write | `src/worker/heartbeat.ts` | TODO |
| Watchdog reaper (>5min stale → TIMED_OUT); crash-recovery scan on startup | `src/core/orchestrator/watchdog.ts` | TODO |
| Budget ceiling sealed at spawn (S-10) | `src/core/orchestrator/budget.ts` | TODO |
| Idempotent resume (skip completed stages) | `src/core/orchestrator/resume.ts` | TODO |
| Integration tests (kill/reap/resume/budget/concurrent) | `tests/integration/jobs.test.ts` | TODO |

## Notes

PID-reuse hazard: verify the PID belongs to an engram worker (`/proc/<pid>/cmdline`
Linux, `ps` macOS) before declaring alive. Worker is a no-op stub here; real
domain logic is 8b. jobs.db schema was created in WP01 phase-2.
