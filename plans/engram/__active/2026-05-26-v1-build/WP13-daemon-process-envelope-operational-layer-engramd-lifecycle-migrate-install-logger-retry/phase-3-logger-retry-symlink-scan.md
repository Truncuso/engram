---
name: phase-3-logger-retry-symlink-scan
title: Structured logger (§10.3) + unified retry utility (§9.2) + S-07 store-open symlink scan (§8.4)
type: phase
phase_status: pending
wp: wp13-daemon-process-envelope-operational-layer-engramd-lifecycle-migrate-install-logger-retry
goal: "A shared structured JSON logger writes to ~/.engram/logs/engramd.log with daily rotation + 14-day retention; a single retry utility implements the §9.2 backoff table used by all retryable ops; the store-open scan rejects outbound symlinks in raw/ and memories/ (the read/scan half of S-07, complementing WP01's write-path O_NOFOLLOW)."
verify: "npm test tests/unit/retry + tests/integration/logger + tests/integration/symlink-scan — retry attempts/backoff per op match the §9.2 table (OCC 10 no-backoff; LLM/AppLog 5; graphify/QMD/git 3; delay=rand(0,min(30s,200ms·2^n))); a log call emits a JSON line {ts,level,component,msg,ctx} to engramd.log and rotates at the day boundary; a symlink planted in raw/ pointing outside the store is rejected at store-open with an AppLog record and is never traversed."
---
<!-- Template: WP-folder PHASE v2 (frontmatter-first) -->

# Phase 3: logger + unified retry + S-07 store-open symlink scan

**Goal:** Three cross-cutting primitives the component WPs assumed but no WP owns.

1. **Structured logger (§10.3):** `src/core/logger.ts` writes JSON lines
   `{ts, level, component, msg, ctx}` to `~/.engram/logs/engramd.log`; daily
   rotation; 14-day retention. This is the observability backbone every component
   (plugin health, OCC events, capture, dream runs) writes to. **Never logs
   secrets or payload content** (consistent with the capture audit channel).
2. **Unified retry (§9.2):** `src/core/retry.ts` implements the one backoff table —
   LLM complete/embed 5, OCC 10 (no backoff), graphify subprocess 3, QMD reindex
   3, git commit 3, AppLog append 5 — with `delay = rand(0, min(30s, 200ms·2^n))`.
   All retryable ops in WP01–WP08 call this instead of rolling their own, so the
   §9.2 table is enforced, not aspirational. All retried ops must be idempotent.
3. **S-07 store-open symlink scan (§8.4):** `src/core/store/symlink-scan.ts` walks
   `raw/` and `memories/` at store-open and **rejects outbound symlinks** (links
   resolving outside the store root). WP01 ships the write-path `O_NOFOLLOW`; this
   is the missing read/scan half WP09 maps to S-07. Wired into the daemon startup
   doctor step (phase-1) and `engram doctor`.

**Verify:** `npm test tests/unit/retry + tests/integration/logger +
tests/integration/symlink-scan` — backoff matches §9.2; log line + rotation;
planted outbound symlink rejected and audited.

## Steps

| Step | File | State |
|------|------|-------|
| Structured JSON logger (levels, component tag, ctx); no-secret guard | `src/core/logger.ts` | TODO |
| Log file rotation (daily) + 14-day retention; `logs/` added to store layout | `src/core/logger.ts` | TODO |
| Unified retry utility: §9.2 per-op attempt table + `rand(0,min(30s,200ms·2^n))` | `src/core/retry.ts` | TODO |
| S-07 store-open symlink scan: walk raw/+memories/, reject outbound symlinks | `src/core/store/symlink-scan.ts` | TODO |
| Wire scan into startup doctor step (phase-1) + `engram doctor` (WP01) | `src/daemon/startup.ts` | TODO |
| Tests: retry table (unit), logger+rotation (integration), planted symlink (integration) | `tests/unit/retry.test.ts`, `tests/integration/{logger,symlink-scan}.test.ts` | TODO |

## Notes

These are shared utilities — ideally landed early so WP01–WP08 import them rather
than duplicating. Because WP13 depends on WP05 (assembly), the utilities here are
the canonical versions; if an earlier WP needs retry/logger before WP13 executes,
extract a thin `src/core/retry.ts`/`logger.ts` stub in WP01 and have WP13 complete
it (note this during grilling — see GAP_REPORT F-4/F-7). S-07 scan is a HIGH
mitigation: pair with WP09's `s07` planted-attack test (currently WP09 maps S-07
to "WP01 store-open" — update that mapping to WP13 during grilling).
</content>
