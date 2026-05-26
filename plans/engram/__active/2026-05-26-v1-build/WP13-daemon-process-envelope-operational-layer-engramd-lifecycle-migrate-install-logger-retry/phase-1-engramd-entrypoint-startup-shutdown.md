---
name: phase-1-engramd-entrypoint-startup-shutdown
title: engramd entrypoint + §9.9 startup sequence + graceful SIGTERM/SIGUSR2 shutdown + PID lock
type: phase
phase_status: pending
wp: wp13-daemon-process-envelope-operational-layer-engramd-lifecycle-migrate-install-logger-retry
goal: "The engramd process entrypoint exists and runs the §9.9 8-step startup sequence in order (PID lock with stale handling → crash-recovery scan → abbreviated doctor → plugin init LLM→Retrieval→Graph → MCP bind → drain spools/fallback → backfill dream schedule → log started), and a graceful SIGTERM shutdown (503 → drain 10s → SIGTERM workers wait 30s → flush → release PID) plus SIGUSR2/--reload."
verify: "npm test tests/integration/daemon-lifecycle — engramd starts and the 8 startup steps run in §9.9 order (assert ordering via a probe log); a second start while a stale pid file exists recovers (reaps dead-pid jobs to FAILED); SIGTERM returns 503 to a new call, drains an in-flight call, reaps a running worker, releases the PID lock, exits 0."
---
<!-- Template: WP-folder PHASE v2 (frontmatter-first) -->

# Phase 1: engramd entrypoint + startup/shutdown

**Goal:** `src/daemon/index.ts` is the long-running `engramd` process. It owns the
**§9.9 ordered startup gate** and **graceful shutdown**, which today are scattered
across WP01–WP07 as fragments with no sequencer. `src/daemon/startup.ts` is the
canonical home of the `src/core/startup.ts` that WP06/phase-2 referenced as a
drain-wiring afterthought — it is promoted to a first-class owned module here.

**Startup (§9.9, ordered):** (1) acquire `.engram/engramd.pid`, handle stale PIDs;
(2) open jobs.db → crash-recovery scan (SPAWNED/RUNNING with dead pid → FAILED —
the dead-pid path of OQ-08, distinct from the watchdog's stale-heartbeat
TIMED_OUT); (3) abbreviated `engram doctor` (WP01); (4) plugin init order
LLM→Retrieval→Graph, all optional-to-boot (C13, WP02); (5) bind MCP `127.0.0.1`,
retry 3× (WP05); (6) drain `git-pending.jsonl` + `applog-recovery.jsonl` +
`capture-fallback/` (WP01/WP06); (7) backfill overdue dream schedule (WP08);
(8) log `engramd started`.

**Shutdown (SIGTERM):** 503 new MCP requests → drain in-flight (10s) → SIGTERM
dream workers, wait 30s for checkpoint, SIGKILL+TIMED_OUT beyond → flush AppLog +
reindex queue (5s) → release PID lock → exit 0. **SIGUSR2 / `engramd --reload`:**
in-flight workers complete on the old process; reload config.

**Verify:** `npm test tests/integration/daemon-lifecycle` — startup steps run in
§9.9 order; stale-pid restart recovers; SIGTERM drains + releases cleanly.

## Steps

| Step | File | State |
|------|------|-------|
| `engramd` entrypoint: arg parse (`--store`, `--reload`), wire startup→serve→shutdown | `src/daemon/index.ts` | TODO |
| PID lock acquire + stale-PID handling (verify pid is an engram daemon via /proc) | `src/daemon/index.ts` | TODO |
| Ordered startup sequencer (the 8 §9.9 steps); calls WP01/02/05/06/08 primitives in order | `src/daemon/startup.ts` | TODO |
| Crash-recovery scan: dead-pid SPAWNED/RUNNING jobs → FAILED (OQ-08 dead-pid path) | `src/daemon/startup.ts` | TODO |
| Graceful SIGTERM shutdown (5-step drain) | `src/daemon/shutdown.ts` | TODO |
| SIGUSR2 / `--reload` graceful restart hook | `src/daemon/shutdown.ts` | TODO |
| Integration tests (start ordering, stale-pid restart, SIGTERM drain) | `tests/integration/daemon-lifecycle.test.ts` | TODO |

## Notes

This phase is the assembly point WP11's `tests/e2e/helpers/daemon.ts`
(`startDaemon`/`stop`) depends on. It introduces no new component logic — it
sequences existing ones. The dead-pid → FAILED crash-recovery path here is the
counterpart to WP08-8a's stale-heartbeat → TIMED_OUT watchdog; together they
resolve OQ-08 (which transition fires for killed vs hung workers).
</content>
