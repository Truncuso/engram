---
name: phase-2-applog-jobs-stats-sqlite
title: SQLite AppLog / jobs / stats / index-state + startup reconciliation
type: phase
phase_status: pending
wp: wp01-core-scaffold-store-schemas-applog-jobs-occ-coreservice-doctor-init
goal: The four SQLite stores exist with their schemas (AppLog events, jobs, stats sidecar, index-state); AppLog append spools on failure; startup reconciliation rebuilds AppLog/git from files-are-truth via index-state cursor.
verify: "npm test tests/integration/applog — an event appends and queries back; a simulated append failure spools to applog-recovery.jsonl and drains on restart; reconciliation re-validates only files newer than index-state cursor."
---
<!-- Template: WP-folder PHASE v2 (frontmatter-first) -->

# Phase 2: SQLite stores + reconciliation

**Goal:** `app-log.db` (events, §10.2 schema incl. `mcp_denied`), `jobs.db`
(§5.4 schema), `stats.db` (recency/access, §7.2), `index-state.db`
(`last_reindexed` cursor) exist via `better-sqlite3`. AppLog append spools to
`applog-recovery.jsonl` on failure. **Startup reconciliation is bounded** (risk
review item 7): use `index-state.db` `last_checked` as the cursor — re-validate
only changed files, not a full O(n) scan.

**Verify:** `npm test tests/integration/applog` — append/query an event; a
simulated failure spools + drains on restart; reconciliation touches only files
newer than the cursor.

## Steps

| Step | File | State |
|------|------|-------|
| AppLog schema + append + spool/drain | `src/core/applog.ts` | TODO |
| jobs.db schema (§5.4 table) — created here, driven in WP08a | `src/core/jobs.ts` | TODO |
| stats.db (recency/access touches, daemon-only) | `src/core/stats.ts` | TODO |
| index-state.db (`last_reindexed`/`last_checked`) | `src/core/index-state.ts` | TODO |
| Startup reconciliation (cursor-bounded, files-are-truth) | `src/core/reconcile.ts` | TODO |
| Integration tests | `tests/integration/applog.test.ts` | TODO |

## Notes

AppLog is NOT git-tracked (C8/OQ-L) — lives in git-ignored `.engram/`.
Reconciliation algorithm is the de-hand-waved version from the risk review: cursor
in `index-state.db`, not "scan for files newer than last app-log entry."
