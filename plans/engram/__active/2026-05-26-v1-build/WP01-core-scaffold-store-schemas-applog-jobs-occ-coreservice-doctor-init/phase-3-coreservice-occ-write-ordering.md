---
name: phase-3-coreservice-occ-write-ordering
title: CoreService write path + OCC + canonical write ordering
type: phase
phase_status: pending
wp: wp01-core-scaffold-store-schemas-applog-jobs-occ-coreservice-doctor-init
goal: CoreService.write performs the canonical ordering (atomic file → AppLog → git) with OCC (version token + commit-time advisory lock, max-3 retry); remember/get/list/history work against files.
verify: "npm test tests/integration/occ — two concurrent updates: one wins, the loser retries cleanly and is rejected after max retries; occ_conflict logged; remember→get→history round-trips."
---
<!-- Template: WP-folder PHASE v2 (frontmatter-first) -->

# Phase 3: CoreService + OCC + write ordering

**Goal:** `CoreService` is the transport-agnostic facade. `write` follows the
canonical order (§7.4): atomic file write → AppLog append (spool on fail) → git
commit (spool on fail). OCC per §7.2: read version → compute → `.tmp` → advisory
lock → re-read version (retry max 3, 50ms jitter) → increment → rename → unlock.
`remember`/`get`/`list`/`history` implemented against files (no plugins yet).

**Verify:** `npm test tests/integration/occ` — concurrent-update race: winner
commits, loser retries then surfaces `version-conflict`; `occ_conflict` in AppLog
(SC-8); `remember → get → history` round-trips with per-field provenance.

## Steps

| Step | File | State |
|------|------|-------|
| CoreService facade skeleton + envelope `{ok,data,error}` | `src/core/coreservice.ts` | TODO |
| OCC write protocol (advisory lock, retry/jitter) | `src/core/occ.ts` | TODO |
| Canonical write ordering + spools | `src/core/write-pipeline.ts` | TODO |
| `remember` / `get` / `list` / `history` (file-backed) | `src/core/coreservice.ts` | TODO |
| Integration tests (OCC race, round-trip) | `tests/integration/occ.test.ts` | TODO |

## Notes

`recency`/`access` updates bypass OCC → stats sidecar (daemon-only). The
advisory lock here is the real-time-write lock; dream-merge OCC (WP08c) is a
distinct path using manifest base_versions (C12).
