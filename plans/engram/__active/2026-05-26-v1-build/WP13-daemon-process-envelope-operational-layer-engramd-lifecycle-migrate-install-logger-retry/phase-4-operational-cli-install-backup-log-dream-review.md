---
name: phase-4-operational-cli-install-backup-log-dream-review
title: Operational CLI — install/backup/restore/log/dream-review-approve/agent-rotate (§10.6, §10.2, §5.4)
type: phase
phase_status: pending
wp: wp13-daemon-process-envelope-operational-layer-engramd-lifecycle-migrate-install-logger-retry
goal: "The operational CLI surface exists: engram install (user-level systemd/launchd unit + log rotation, no root), engram backup/restore, engram log [--type|--denied], engram dream review --approve <run_id> (REVIEW_PENDING→MERGED, capability-checked + re-validated), and engram agent rotate <id> (atomic bearer+MAC re-issue)."
verify: "npm test tests/integration/operational-cli — engram install writes a user-level service unit (no root) and a log-rotation config; engram backup then engram restore round-trips a store; engram log --denied prints mcp_denied rows; engram dream review --approve drives a REVIEW_PENDING job to MERGED only with the right capability and only after re-validation; engram agent rotate re-issues bearer+MAC atomically and invalidates the old ones."
---
<!-- Template: WP-folder PHASE v2 (frontmatter-first) -->

# Phase 4: operational CLI

**Goal:** The user-/operator-facing commands the SPEC specifies but no WP owns.

- **`engram install` (§10.6):** write a **user-level** systemd unit (Linux) or
  launchd plist (macOS) — no root — that supervises `engramd`; install the log
  rotation config that §10.3 retention depends on.
- **`engram backup` / `engram restore` (§10.6):** snapshot/restore a store
  (esp. for git-off stores or point-in-time recovery).
- **`engram log [--type <x>] [--denied]` (§10.2):** read AppLog channels;
  `--denied` surfaces `mcp_denied` (C14); `--type governance` etc. filter by
  event type. (`engram history` already exists in WP01; this is the broader log
  surface.)
- **`engram dream review --approve <run_id>` (§5.4):** the only path
  `REVIEW_PENDING → MERGED`. Capability-checked (`dream`/`govern`); re-validates
  the gated hunks against current `base_versions` before merging (a write since
  job start re-routes to conflict). Without this, gated dream hunks pile up with
  no resolution path.
- **`engram agent rotate <id>` (§8.3):** atomically re-issue the bearer token and
  the MAC secret; invalidate the old ones; active sessions must re-auth. WP05
  ships `add`/`revoke`; `rotate` is referenced by WP06 but owned here.

**Verify:** `npm test tests/integration/operational-cli` — install writes a
no-root unit + rotation; backup→restore round-trips; `log --denied` prints
`mcp_denied`; `dream review --approve` merges only with capability + re-validation;
`agent rotate` re-issues atomically.

## Steps

| Step | File | State |
|------|------|-------|
| `engram install` — user systemd unit / launchd plist + log rotation; no root | `src/cli/commands/install.ts` | TODO |
| `engram backup` / `engram restore` — store snapshot + restore | `src/cli/commands/backup.ts` | TODO |
| `engram log [--type] [--denied]` — AppLog channel reader | `src/cli/commands/log.ts` | TODO |
| `engram dream review --approve <run_id>` — REVIEW_PENDING→MERGED, capability + re-validate | `src/cli/commands/dream-review.ts` | TODO |
| `engram agent rotate <id>` — atomic bearer+MAC re-issue (extends WP05 auth) | `src/mcp/auth.ts` | TODO |
| Integration tests (install/backup-restore/log/dream-review/rotate) | `tests/integration/operational-cli.test.ts` | TODO |

## Notes

`dream review --approve` is the REVIEW_PENDING escape hatch for every gated hunk —
its capability gate and pre-merge re-validation are security-relevant; pair with a
WP09 test (GAP_REPORT N-3 notes WP09 currently has no test for this path).
`engram backup`/`restore` may be trimmable to v2 if git-tracked stores suffice —
resolve during grilling (WP13 follow-up issue). `engram install` makes §10.3's
"14-day retention via logrotate" real; without it the logger's rotation has no
installer.
</content>
