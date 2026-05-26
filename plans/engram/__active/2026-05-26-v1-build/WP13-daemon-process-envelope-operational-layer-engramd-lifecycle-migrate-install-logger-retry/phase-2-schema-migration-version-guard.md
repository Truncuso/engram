---
name: phase-2-schema-migration-version-guard
title: Schema-version start guard + engram migrate forward-migration runner (§9.10)
type: phase
phase_status: pending
wp: wp13-daemon-process-envelope-operational-layer-engramd-lifecycle-migrate-install-logger-retry
goal: "The daemon refuses to start against a newer .engram/schema-version than it supports; engram migrate applies forward migrations explicitly (never automatic) and logs each as an AppLog event_type: migration."
verify: "npm test tests/integration/migrate — a store with a schema-version newer than the daemon's supported version → daemon start aborts with a clear error; engram migrate on an older-version store applies the forward migration, bumps schema-version, and writes an AppLog migration event; running migrate twice is idempotent (no-op on an up-to-date store)."
---
<!-- Template: WP-folder PHASE v2 (frontmatter-first) -->

# Phase 2: schema-version guard + `engram migrate`

**Goal:** Close §9.10. WP01 creates the `.engram/schema-version` file and reserves
the `migration` AppLog `event_type`, but nothing enforces the version or applies
migrations. This phase adds: (1) a **start guard** — the daemon reads
`schema-version` during startup (phase-1 step 3-adjacent) and **refuses to start
against a newer schema** (forward-only safety); (2) the **`engram migrate`** CLI —
applies forward migration scripts explicitly (never automatic), bumps
`schema-version`, and logs each step as `event_type: migration` in AppLog.

This is load-bearing for any future schema change and is the path WP12's cutover
invokes when migrating content into a fresh store version.

**Verify:** `npm test tests/integration/migrate` — newer-schema store aborts
startup; `engram migrate` applies + bumps + logs; idempotent on an up-to-date
store.

## Steps

| Step | File | State |
|------|------|-------|
| Schema-version comparator + start guard (refuse newer; warn on older → suggest migrate) | `src/core/migrate.ts` | TODO |
| Forward-migration runner (ordered scripts; transactional per step; AppLog `migration` event) | `src/core/migrate.ts` | TODO |
| Wire the guard into the daemon startup sequence (phase-1) | `src/daemon/startup.ts` | TODO |
| `engram migrate` CLI command (dry-run by default; `--apply` executes) | `src/cli/commands/migrate.ts` | TODO |
| Integration tests (newer→abort; older→migrate+log; idempotent) | `tests/integration/migrate.test.ts` | TODO |

## Notes

Forward-only: there is no automatic or backward migration in v1 (§9.10). The
`migration` AppLog row preserves the from/to version and the script id for audit.
Migration scripts live under a versioned dir; the runner picks up only those
between current and target version.
</content>
