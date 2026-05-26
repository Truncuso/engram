---
name: wp13-daemon-process-envelope-operational-layer-engramd-lifecycle-migrate-install-logger-retry
title: Daemon process envelope + operational layer (engramd lifecycle, migrate, install, logger, retry)
type: work-package
stage: spec
severity: HIGH
created: 2026-05-26
updated: 2026-05-26
plan: 2026-05-26-v1-build
tags: [daemon, engramd, lifecycle, startup, shutdown, migrate, install, logger, retry, operational]
relationships:
  - depends_on: [[wp05-mcp-server-coreservice-facade-16-verbs-bearer]]
  - depends_on: [[wp01-core-scaffold-store-schemas-applog-jobs-occ-coreservice-doctor-init]]
  - blocks: [[wp11-e2e-verification-18-success-criteria]]
sources: [SRC-01, SRC-03]
phases: [phase-1-engramd-entrypoint-startup-shutdown, phase-2-schema-migration-version-guard, phase-3-logger-retry-symlink-scan, phase-4-operational-cli-install-backup-log-dream-review]
---
<!-- Template: WP-folder OVERVIEW v2 (frontmatter-first) -->

# WP13: Daemon process envelope + operational layer

> Folder work package. Phases live in `phase-N-<slug>.md`. `stage:` advances only
> when all phase `phase_status:` are `done`.

## Problem

WP01–WP12 build engram's **components** (store, plugins, scoring, MCP, capture,
workers, governance) — but **no WP owns the `engramd` process itself**: the
long-running daemon that starts those components in the §9.9 order, holds the PID
lock, drains spools, handles `SIGTERM`, installs as a service, and enforces the
schema-version guard. Every other WP and the WP11 E2E harness (`startDaemon`)
*assume* `engramd` exists and runs; nothing builds it. This WP is the **process
envelope + operational layer** that makes engram an installable, operable daemon.

It also closes three smaller SPEC items the component-decomposition skipped:
the **§8.4 S-07 store-open symlink scan** (only the write-path `O_NOFOLLOW`
exists in WP01; the scan that rejects outbound symlinks in `raw/`/`memories/` is
unowned), the **§9.2 unified retry utility** (each WP currently rolls its own),
and the **§10.3 structured logger** (`~/.engram/logs/engramd.log` + rotation),
which is the observability backbone every component writes to.

This WP was added by the 2026-05-26 gap-scan (see `GAP_REPORT.md`, finding cluster
"daemon process envelope"). It is a verification/assembly layer over WP01–WP05 +
the workers; it depends on WP05 (the assembled startup-to-MCP-bind sequence) and
WP01 (PID/spool/store-layout primitives), and it blocks WP11 (E2E needs a real
startable/stoppable daemon).

SPEC refs: §9.9 (startup/shutdown), §9.10 (schema versioning + `engram migrate`),
§8.4 S-07 (symlink scan), §9.2 (unified retry), §10.3 (structured logs + `engram
status` full fields), §10.6 (`engram install`/`backup`/`restore`, `SIGUSR2`
reload), §5.4 (`engram dream review --approve` REVIEW_PENDING→MERGED), §10.2
(`engram log`).

---

## Target Files

- `src/daemon/index.ts` — `engramd` process entrypoint: acquires `.engram/engramd.pid` (stale-PID handling), runs the §9.9 8-step startup sequence in order, installs SIGTERM/SIGUSR2 handlers, owns the process lifecycle
- `src/daemon/startup.ts` — ordered startup sequencer (the canonical home of `src/core/startup.ts` referenced by WP06): PID lock → crash-recovery scan → abbreviated doctor → plugin init (LLM→Retrieval→Graph) → MCP bind → drain spools/fallback → backfill dream schedule → log started
- `src/daemon/shutdown.ts` — graceful SIGTERM (503 new requests → drain 10s → SIGTERM workers wait 30s → flush AppLog/reindex → release PID lock) and `SIGUSR2`/`engramd --reload`
- `src/core/migrate.ts` — schema-version guard (refuse start against newer schema) + forward-migration runner (logs `event_type: migration`)
- `src/core/logger.ts` — structured JSON logger to `~/.engram/logs/engramd.log`; daily rotation; 14-day retention; `{ts, level, component, msg, ctx}`
- `src/core/retry.ts` — unified retry utility implementing the §9.2 backoff table (LLM 5 / OCC 10 no-backoff / graphify 3 / QMD 3 / git 3 / AppLog 5); `delay = rand(0, min(30s, 200ms·2^n))`
- `src/core/store/symlink-scan.ts` — store-open scan that rejects outbound symlinks in `raw/`/`memories/` (§8.4 S-07, the read/scan half)
- `src/cli/commands/migrate.ts` — `engram migrate`
- `src/cli/commands/install.ts` — `engram install` (systemd user unit / launchd plist; user-level, no root; installs log rotation)
- `src/cli/commands/backup.ts` — `engram backup` / `engram restore`
- `src/cli/commands/log.ts` — `engram log [--type <x>] [--denied]`
- `src/cli/commands/dream-review.ts` — `engram dream review --approve <run_id>` (REVIEW_PENDING→MERGED, §5.4)
- token/MAC `rotate`: extend `src/mcp/auth.ts` (WP05) with `engram agent rotate <id>` — atomic re-issue of bearer + MAC

## Phases

| Phase | Goal | Status |
|-------|------|--------|
| [phase-1](phase-1-engramd-entrypoint-startup-shutdown.md) | `engramd` entrypoint + §9.9 8-step startup + graceful SIGTERM/SIGUSR2 shutdown + PID lock | pending |
| [phase-2](phase-2-schema-migration-version-guard.md) | Schema-version start guard + `engram migrate` forward-migration runner (§9.10) | pending |
| [phase-3](phase-3-logger-retry-symlink-scan.md) | Structured logger (§10.3) + unified retry utility (§9.2) + S-07 store-open symlink scan (§8.4) | pending |
| [phase-4](phase-4-operational-cli-install-backup-log-dream-review.md) | Operational CLI: `install`/`backup`/`restore`/`log`/`dream review --approve`/`agent rotate` (§10.6, §10.2, §5.4) | pending |

## Verification

> Required before `stage: ready`. Aggregates the per-phase `verify:` checks.

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| W13-1 | `engramd` start runs §9.9 steps in order; PID lock acquired; stale PID handled | daemon reaches "started" with all 8 gates; second start with stale pid recovers | integration |
| W13-2 | SIGTERM graceful shutdown | 503 to new calls; in-flight drained; workers SIGTERMed then reaped; PID released; exit 0 | integration |
| W13-3 | start against a newer `schema-version` | daemon refuses to start; `engram migrate` applies forward migration + logs `migration` AppLog event | integration (§9.10) |
| W13-4 | structured log lines written to `~/.engram/logs/engramd.log` | JSON lines `{ts,level,component,msg,ctx}`; rotation honored | integration (§10.3) |
| W13-5 | unified retry backoff matches §9.2 table | OCC = 10 no-backoff; LLM/AppLog = 5; graphify/QMD/git = 3; formula `rand(0,min(30s,200ms·2^n))` | unit (§9.2) |
| W13-6 | S-07 store-open scan rejects an outbound symlink planted in `raw/` | daemon refuses/quarantines; AppLog records it; no read traverses the symlink | integration (§8.4 S-07) |
| W13-7 | `engram dream review --approve <run_id>` drives REVIEW_PENDING→MERGED | gated hunks merge after approval; capability-checked; re-validated before merge | integration (§5.4) |
| W13-8 | `engram install` writes a user-level service unit (no root) | systemd user unit (Linux) / launchd plist (macOS) written; log rotation installed | integration/manual (§10.6) |
| W13-9 | `engram log --type governance` / `--denied` | reads AppLog channels; prints rows; `mcp_denied` surfaced via `--denied` | integration (§10.2) |
| W13-10 | `engram backup` then `engram restore` round-trips a store | restore reproduces store state (esp. for git-off stores) | integration (§10.6) |
| W13-11 | `engram agent rotate <id>` | re-issues bearer + MAC atomically; old token/MAC invalidated; sessions re-auth | integration (§8.3) |

## Recommended Agents

| Phase | Agent | Rationale |
|-------|-------|-----------|
| phase-1 | `typescript-pro`, `code-reviewer` | process lifecycle, signal handling, ordered startup; crash-isolation correctness |
| phase-2 | `database-reviewer`, `typescript-reviewer` | migration runner + schema-version guard; forward-only safety |
| phase-3 | `security-reviewer`, `typescript-pro` | S-07 symlink scan (HIGH mitigation); logger has no secret leakage; retry idempotency |
| phase-4 | `devops-engineer`, `security-reviewer` | service-unit install (no root); `dream review` capability gate; `agent rotate` atomicity |
| all | `code-reviewer`, `tdd-guide` | per-phase gate |

## Review

| Reviewer | Verdict | Notes |
|-----------|---------|-------|
| — | — | — |

## Follow-up Issues

| Issue | Severity | Status |
|-------|----------|--------|
| `engram backup`/`restore` scope may be trimmable to v2 if point-in-time git stores suffice; confirm during grilling | LOW | Open |
</content>
