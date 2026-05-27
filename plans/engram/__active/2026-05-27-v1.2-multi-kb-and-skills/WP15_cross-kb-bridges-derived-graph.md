---
name: wp15-cross-kb-bridges-derived-graph
title: Cross-KB bridges (derived graph layer)
type: work-package
stage: spec
severity: MEDIUM
created: 2026-05-27
updated: 2026-05-27
plan: 2026-05-27-v1.2-multi-kb-and-skills
tags: [kb, bridges, graph, derived-index]
relationships:
  - blocked-by: [[wp13-multi-kb-registry-and-kbplugin-seam]]
  - blocked-by: [[wp14-per-kb-lifecycle-workers]]
sources: [SRC-01]
---
<!-- Template: WP v2 (frontmatter-first) -->

# WP15: Cross-KB bridges (derived graph layer)

## Problem

Multi-KB recall (WP13) returns a ranked union but does not surface
*connections* — a project memory and a wiki entry on the same topic stay
unconnected. The naive answer is live cross-KB edges, but ADR-0004 forbids
live edge mutation and ADR-0007 makes the design choice explicit: bridges are
a **derived index** built by the existing job state machine, recomputed not
mutated, opt-in for recall expansion. Scoring formula stays unchanged
(no RRF).

WP15 ships the `kb.connect.bridge` builder, the `bridges.json` artifact, the
opt-in `expand_via_bridges` recall path, and `engram bridges show <id>`.

## Target Files

- `src/worker/kb/bridges.ts` — bridge builder (consumes per-KB graphify
  graphs, emits `bridges.json`).
- `src/schemas/bridges.ts` — Zod schema for bridge edges; closed `kind` enum
  (`entity-match`, `title-match`, `embedding-near`, `derived-from-citation`,
  `contradicts`).
- `src/core/recall/expand.ts` — `expand_via_bridges` recall path.
- `src/core/mcp/verbs/bridges.ts` — MCP verb `bridges.show`.
- `src/cli/bridges.ts` — `engram bridges show <id>`.
- `tests/unit/bridges/**`, `tests/integration/bridges-roundtrip.test.ts`.

## Verification Gate

| # | Check | Test |
|---|-------|------|
| 1 | `kb.connect.bridge` reads per-KB graphify graphs and emits `bridges.json` matching the schema; closed `kind` enum. | `tests/integration/bridges-roundtrip.test.ts` |
| 2 | A recall query with `expand_via_bridges: true` returns at least the same results as without; expanded results carry `via_bridge`. | `tests/integration/recall-expand.test.ts` (SC-20) |
| 3 | Scoring formula unchanged — bridge expansion adds candidates but does NOT reshape rank. | `tests/integration/recall-no-rank-change.test.ts` |
| 4 | Deleting `bridges/` is safe; next scheduled run rebuilds; recall without expansion is unaffected during the gap. | `tests/integration/bridges-idempotent-rebuild.test.ts` |
| 5 | `engram bridges show <id>` walks one bridge edge and renders `{kind, score, why, from, to}`. | `tests/integration/bridges-show.test.ts` |
| 6 | KB unregister removes that KB's bridges without touching memory files. | `tests/integration/bridges-unregister-cleanup.test.ts` |
| 7 | A KbPlugin cannot invent new bridge `kind`s — closed enum enforced at validation. | `tests/unit/bridges/closed-enum.test.ts` |

## Implementation Steps

| Step | File | State |
|------|------|-------|
| Bridge schema (Zod) | `src/schemas/bridges.ts` | TODO |
| Bridge builder (consumes per-KB graphify graphs) | `src/worker/kb/bridges.ts` | TODO |
| Opt-in `expand_via_bridges` in recall | `src/core/recall/expand.ts` | TODO |
| MCP verb `bridges.show` + CLI | `src/core/mcp/verbs/bridges.ts`, `src/cli/bridges.ts` | TODO |
| Tests | `tests/{unit,integration}/bridges/**` | TODO |

## Verified Evidence

— (none yet — WP in `spec` stage)

## Agents

| Stage | Agent | Reason |
|-------|-------|--------|
| impl | `typescript-pro` | derived-index builder |
| review | `code-reviewer` | ADR-0004/0007 invariants |

## Review

| Reviewer | Verdict | Notes |
|----------|---------|-------|
| — | — | — |

## Follow-up Issues

| Issue | Severity | Status |
|-------|----------|--------|
| — | — | — |
