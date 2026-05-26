---
name: wp01-core-scaffold-store-schemas-applog-jobs-occ-coreservice-doctor-init
title: Core scaffold (store, schemas, AppLog, jobs, OCC, CoreService, doctor, init)
type: work-package
stage: spec
severity: HIGH
created: 2026-05-26
updated: 2026-05-26
plan: 2026-05-26-v1-build
tags: [core, store, schema, occ, applog]
relationships:
  - depends_on: [[wp00-repo-scaffold-tooling-baseline]]
  - blocks: [[wp02-plugin-host-llmplugin-vercel-ai-sdk]]
  - blocks: [[wp03-retrieval-plugin-qmd-in-process-stats-sidecar]]
sources: [SRC-01]
phases: [phase-1-schemas-and-store, phase-2-applog-jobs-stats-sqlite, phase-3-coreservice-occ-write-ordering, phase-4-init-doctor-grep-recall]
---
<!-- Template: WP-folder OVERVIEW v2 (frontmatter-first) -->

# WP01: Core scaffold (store, schemas, AppLog, jobs, OCC, CoreService, doctor, init)

> Folder work package. Phases live in `phase-N-<slug>.md`. `stage:` advances only
> when all phase `phase_status:` are `done`.

## Problem

The fixed core kernel must exist before any plugin: the Markdown store layout,
the frontmatter/manifest **schemas** (single source of truth, Zod + JSON-schema
export), the **SQLite** stores (AppLog, jobs.db, stats.db, index-state.db),
**OCC** + atomic write ordering, the **CoreService** facade with
`remember`/`get`/`list`/`history`/`recall(grep)`, `engram init`, and an
abbreviated `engram doctor`. Delivers **Milestone M0**: remember a fact → recall
it via grep with importance+recency scoring — a working loop before any plugin.

SPEC refs: §3.2–3.6 (model/store/schema/scoring), §4.C (remember), §7.2/§7.4
(OCC, write ordering), §9.8 (doctor), §9.9 (startup), §13.1.

## Target Files

- `src/schemas/frontmatter.ts` — Zod schema + TS type for memory frontmatter (§3.4)
- `src/schemas/manifest.ts`, `src/schemas/dream-output.ts` — manifest + dream-output (JSON-schema export)
- `src/core/store/` — store layout, atomic file write (`.tmp`→fsync→rename), ULID ids, slug/id filenames (§3.3, §3.5)
- `src/core/applog.ts`, `src/core/jobs.ts`, `src/core/stats.ts` — `better-sqlite3` stores (§10.2, §5.4, §7.2)
- `src/core/occ.ts` — version token + advisory-lock write protocol (§7.2)
- `src/core/coreservice.ts` — facade: `remember/get/list/history/recall` (§4.C)
- `src/core/doctor.ts` — abbreviated integrity check + quarantine (§9.8)
- `src/cli/index.ts` — `engram init`, `engram doctor` (§13.1)

## Phases

| Phase | Goal | Status |
|-------|------|--------|
| [phase-1](phase-1-schemas-and-store.md) | Schemas (Zod+JSON-schema) + store layout + atomic write | pending |
| [phase-2](phase-2-applog-jobs-stats-sqlite.md) | SQLite AppLog/jobs/stats/index-state + reconciliation | pending |
| [phase-3](phase-3-coreservice-occ-write-ordering.md) | CoreService write path + OCC + canonical write ordering | pending |
| [phase-4](phase-4-init-doctor-grep-recall.md) | `engram init` + `doctor` + grep-recall (**M0**) | pending |

## Verification

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| W01-1 | `engram init` scaffolds store, idempotent | re-run = no-op; tree matches §3.3 | e2e (SC-1) |
| W01-2 | remember → recall (grep) with score | fact returns, importance+recency scored | e2e (M0) |
| W01-3 | concurrent OCC race | one write rejected, retried, `occ_conflict` in AppLog | integration (SC-8) |
| W01-4 | `memory.history <id>` per-field provenance | events with field/old/new | integration (SC-9 partial) |
| W01-5 | `doctor` quarantines broken frontmatter | bad file → `.engram/quarantine/`, startup ok | integration (SC-15) |
| W01-6 | frontmatter schema rejects bad/injected fields | Zod parse error | unit |

## Recommended Agents

| Phase | Agent | Rationale |
|-------|-------|-----------|
| schemas | `typescript-pro` | Zod + type derivation |
| sqlite | `database-reviewer` | AppLog/jobs/stats schema |
| occ/write | `typescript-reviewer` | concurrency correctness |
| all | `code-reviewer`, `tdd-guide` | per-phase gate |
