---
name: wp10-governance-cascade-delete
title: Governance + cascade delete
type: work-package
stage: spec
severity: MEDIUM
created: 2026-05-26
updated: 2026-05-26
plan: 2026-05-26-v1-build
tags: [governance, cascade-delete, git-filter-repo, applog, qmd, graphify, cli, security]
relationships:
  - depends_on: [[wp05-mcp-server-coreservice-facade-16-verbs-bearer]]
  - depends_on: [[wp07-ingest-worker-graphify-graphplugin-ollama]]
sources: [SRC-01]
---
<!-- Template: WP v2 (frontmatter-first) -->

# WP10: Governance + cascade delete

## Problem

`memory.governance_delete` (MCP verb, requires `govern` capability) and the
`engram governance scrub <pattern>` CLI command both need to purge a memory or
secret across five distinct stores atomically from the user's perspective (SPEC
§8.5, §6.3, §10.2). Without this, sensitive content removed from `memories/` can
survive in git history, the AppLog, QMD index, or graphify graph — all of which
constitute a data-leak vector (S-08, SPEC §8.3 CRITICAL).

The five-step cascade (SPEC §8.5):
1. Delete the Markdown file from the working tree.
2. Rewrite git history: `git filter-repo --path <file> --invert-paths` (or
   `--replace-text <pattern>` for `governance scrub`).
3. AppLog tombstone: insert `event_type: governance` record; mark the deleted
   memory's mutation rows as tombstoned (do not delete AppLog rows — audit trail).
4. QMD deindex: call `RetrievalPlugin.deindex(id)`.
5. graphify graph: call `GraphPlugin.removeNode(id)` (rebuild/filter per §6.2 /
   ADR-0004 — no live delete; rebuild after file removal).

**Dry-run by default.** The cascade does not execute steps 2–5 without `--apply`
(CLI) or `purgeHistory: true` (MCP). Step 1 (working tree delete) is always shown
in dry-run output.

The `engram governance scrub <pattern>` CLI variant (§8.3 S-08, §10.2) runs
`git filter-repo --replace-text` over the entire store history to scrub a pattern
(e.g. a leaked API key), then deindexes and rebuilds derived indexes. It is a
bulk operation; dry-run shows matched commits and files, `--apply` executes.

The `engram lifecycle revive --since <ts>` and `engram repair [--fix]` CLI verbs
(§10.2 C9) are also implemented in this WP as they share the governance command
module.

This WP depends on WP07 because `GraphPlugin.removeNode` is defined there. It
depends on WP05 because `CoreService.governanceDelete` and the `govern` capability
check live in the MCP server facade.

---

## Target Files

- `src/core/governance.ts` — `GovernanceService`: implements the five-step cascade;
  exposes `deleteMemory(id, opts: {purgeHistory, dryRun})` and
  `scrubPattern(pattern, opts: {dryRun})`; orchestrates plugin calls + AppLog write
- `src/cli/commands/governance.ts` — CLI commands: `engram governance scrub`,
  `engram lifecycle revive`, `engram repair`; thin adapters over `GovernanceService`
  and `CoreService`

---

## Verified Evidence

- — (no pre-existing implementation; cascade steps are specified in SPEC §8.5)

---

## Implementation Steps

| Step | File | State |
|------|------|-------|
| 1. Implement `GovernanceService.deleteMemory(id, {purgeHistory, dryRun})`: resolve memory path from id; verify `govern` capability on calling agent | `src/core/governance.ts` | TODO |
| 2. Step 1 of cascade: delete `memories/<type>/<slug>.md` from working tree; if `dryRun`, report what would be deleted and return | `src/core/governance.ts` | TODO |
| 3. Step 2 (history rewrite, `purgeHistory` only): spawn `git filter-repo --path <relpath> --invert-paths` as `execFile` with args array (S-16); on exit non-zero: surface error, leave AppLog record of attempt | `src/core/governance.ts` | TODO |
| 4. Step 3 (AppLog tombstone): insert `event_type: governance` row with `memory_id`, `agent_id`, `justification`; UPDATE tombstone flag on prior mutation rows for `memory_id` (schema: add `tombstoned BOOLEAN DEFAULT 0` column if not present) | `src/core/governance.ts` | TODO |
| 5. Step 4 (QMD deindex): call `RetrievalPlugin.deindex(id)`; log failure as WARN but do not abort cascade | `src/core/governance.ts` | TODO |
| 6. Step 5 (graphify removeNode): call `GraphPlugin.removeNode(id)` (triggers rebuild/filter of `graph.json`); log failure as WARN but do not abort cascade | `src/core/governance.ts` | TODO |
| 7. Implement `GovernanceService.scrubPattern(pattern, {dryRun})`: spawn `git filter-repo --replace-text <tmpfile>` (pattern file, not shell arg); then call `rebuild()` on both RetrievalPlugin and GraphPlugin; AppLog governance event | `src/core/governance.ts` | TODO |
| 8. CLI: `engram governance scrub <pattern> [--apply]` — dry-run by default; print matched commits/files; `--apply` calls `GovernanceService.scrubPattern(pattern, {dryRun: false})` | `src/cli/commands/governance.ts` | TODO |
| 9. CLI: `memory.governance_delete` verb in CoreService calls `GovernanceService.deleteMemory`; already stubbed in WP05; implement the delegation | `src/core/governance.ts` | TODO |
| 10. CLI: `engram lifecycle revive --since <ts>` — query AppLog for memories transitioned to dormant/archived after `<ts>`; batch set lifecycle back to `active`; write AppLog events; dry-run by default | `src/cli/commands/governance.ts` | TODO |
| 11. CLI: `engram repair [--fix]` — call `CoreService.doctor({fix: !!opts.fix})`; print structured report; calls WP01's doctor implementation | `src/cli/commands/governance.ts` | TODO |

---

## Verification

### Build Gates

| Gate | Command | Expected |
|------|---------|----------|
| TypeScript compiles | `npm run build` | exit 0 |
| Governance unit tests pass | `npm test -- --testPathPattern=core/governance` | all green |
| CLI governance tests pass | `npm test -- --testPathPattern=cli/commands/governance` | all green |
| Integration test SC-16 | `npm test -- --testPathPattern=integration/governance` | SC-16 passes |

### Specific Tests

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| SC-16 (§12.3) | Call `governance_delete --purge-history` on a memory containing a planted secret | (1) Markdown file absent from working tree; (2) `git log -S <secret>` returns no commits; (3) AppLog contains `event_type: governance` + tombstone flag on prior mutation rows; (4) QMD `search(<secret>)` returns 0 hits; (5) `GraphPlugin.traverse` no longer returns the deleted node ID | Integration test in `tests/integration/governance.test.ts` |
| Dry-run default | Call `engram governance scrub <pattern>` without `--apply` | Command prints matched files/commits, exits 0, no git history rewritten, no deindex | Integration test |
| `--apply` executes | Call `engram governance scrub <pattern> --apply` | git filter-repo runs; history scrubbed; QMD and graphify rebuilt | Integration test |
| AppLog tombstone | After `governance_delete`, query `engram history <id>` | Returns governance event; prior mutation rows marked tombstoned | Unit test |
| lifecycle revive | Dormant memories revived via `engram lifecycle revive --since <ts>` | Lifecycle → active; AppLog events written; dry-run shows list without writing | Integration test |
| govern capability gate | Call `memory.governance_delete` with agent lacking `govern` cap | Returns `scope-denied`; no cascade steps executed | Unit test |
| git filter-repo arg array | Inspect `execFile` call for history rewrite | No shell:true; `--path` and path passed as separate array elements | Code inspection + test with space-in-path fixture |

---

## Recommended Agents

| Phase | Agent | Rationale |
|-------|-------|-----------|
| Implementation | security-reviewer | S-08 is a CRITICAL mitigation; history-rewrite and AppLog tombstone logic must be reviewed before merge |
| Implementation | code-reviewer | Five-step cascade has partial-failure modes; each step's error handling and logging must be correct |
| Implementation | tdd-guide | SC-16 acceptance criterion is well-defined; write the E2E test before implementing the cascade |

---

## Review

| Reviewer | Verdict | Notes |
|----------|---------|-------|
| — | — | — |

---

## Follow-up Issues

| Issue | Severity | Status |
|-------|----------|--------|
| — | — | — |
