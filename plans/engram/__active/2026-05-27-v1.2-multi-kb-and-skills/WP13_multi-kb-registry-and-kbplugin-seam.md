---
name: wp13-multi-kb-registry-and-kbplugin-seam
title: Multi-KB registry + `KbPlugin` seam (5th plugin)
type: work-package
stage: spec
severity: HIGH
created: 2026-05-27
updated: 2026-05-27
plan: 2026-05-27-v1.2-multi-kb-and-skills
tags: [kb, plugin, registry, kernel]
relationships:
  - blocked-by: [[wp05-mcp-server-coreservice-facade-16-verbs-bearer]]  # v1 M2
  - blocks: [[wp14-per-kb-lifecycle-workers]]
  - blocks: [[wp15-cross-kb-bridges-derived-graph]]
  - blocks: [[wp16-engram-skill-subsystem-and-installer]]
  - blocks: [[wp17-episodic-semantic-linkage-hardening]]
sources: [SRC-01, SRC-02, SRC-03]
---
<!-- Template: WP v2 (frontmatter-first) -->

# WP13: Multi-KB registry + `KbPlugin` seam

## Problem

v2.1 hosts exactly one engram store per scope. v2.2 §15 requires multiple
named KB instances of different shapes (`markdown-store`, `llm-wiki`,
`obsidian-vault`, `raw-sources`, `agent-self`). Encoding five shapes in the
kernel violates Simplicity-First; federating engramd instances per KB doubles
deployment cost. ADR-0005 picks the middle ground: a fifth plugin seam
(`KbPlugin`) where the type is the plugin and the instance is a registry row.

WP13 ships the seam, the registry, and the five built-in `KbPlugin`s, with
per-KB QMD index and per-KB graphify graph provisioned on register. Recall
fans out across connected KBs and the existing scoring engine (§3.6) ranks
the union — no per-KB rank, no RRF.

## Target Files

- `src/core/kb/registry.ts` — registry + `kbs` SQLite table.
- `src/core/kb/plugin.ts` — `KbPlugin` contract types (Zod schemas + derived
  TS types in `src/schemas/kb.ts`).
- `src/plugins/kb/markdown-store/` — KbPlugin for the existing v2.1 layout
  (the current store becomes a `markdown-store` instance).
- `src/plugins/kb/llm-wiki/` — KbPlugin for the Pratiyush/llm-wiki layout.
- `src/plugins/kb/obsidian-vault/` — KbPlugin for an Obsidian vault, including
  Dataview-aware fallback retrieval.
- `src/plugins/kb/raw-sources/` — KbPlugin for read-mostly ingest pools.
- `src/plugins/kb/agent-self/` — KbPlugin for per-agent scratch KBs.
- `src/core/mcp/verbs/kb.ts` — MCP verbs `kb.register`, `kb.list`, `kb.status`,
  `kb.connect`, `kb.disconnect`, `kb.unregister`, `kb.route`.
- `src/core/recall/fan-out.ts` — multi-KB fan-out before the unified scoring
  pass.
- `src/cli/kb.ts` — `engram kb {register, list, status, connect, disconnect,
  unregister}`.
- `tests/unit/kb/**` — registry, plugin contract, fan-out unit tests.
- `tests/integration/kb-multi-recall.test.ts` — register 3 KBs, recall fan-out
  returns a unified ranked set.

## Verification Gate

| # | Check | Test |
|---|-------|------|
| 1 | `kb.register(type, root)` validates via the plugin's `validate()`, writes a `kbs` row, provisions per-KB QMD + graphify, enqueues `lifecycleJobs()`. | `tests/integration/kb-register.test.ts` |
| 2 | All 5 built-in `KbPlugin`s pass the contract round-trip (manifest, validate, layout, ingest, lifecycleJobs). | `tests/unit/kb/<type>.test.ts` |
| 3 | Recall fans out across connected KBs and the **same scoring engine** ranks the union; no RRF; no per-KB rank. | `tests/integration/kb-multi-recall.test.ts` |
| 4 | `kb.unregister` removes registry row + per-KB indexes + per-KB bridges; **never** touches source files. | `tests/integration/kb-unregister-isolation.test.ts` |
| 5 | Auto-discovery: `engram doctor --discover` proposes registration for an Obsidian vault and an llm-wiki tree under a watched root. | `tests/integration/kb-discover.test.ts` |
| 6 | A second `obsidian-vault` instance pointing at a different vault registers without collision. | `tests/integration/kb-two-instances-same-type.test.ts` |
| 7 | Privacy filter, access control, safe/gated classification stay in the kernel — a `KbPlugin` cannot opt out. | `tests/unit/kb/security-invariants.test.ts` |

## Implementation Steps

| Step | File | State |
|------|------|-------|
| Define `KbPlugin` contract (Zod + TS) | `src/schemas/kb.ts`, `src/core/kb/plugin.ts` | TODO |
| Add `kbs` table + migration | `src/core/storage/sqlite.ts` | TODO |
| Implement registry + per-KB index/graph provisioning | `src/core/kb/registry.ts` | TODO |
| Implement 5 built-in `KbPlugin`s | `src/plugins/kb/*/` | TODO |
| Wire MCP verbs `kb.*` (7 verbs total, additive to existing 16) | `src/core/mcp/verbs/kb.ts` | TODO |
| Implement multi-KB recall fan-out | `src/core/recall/fan-out.ts` | TODO |
| CLI `engram kb …` | `src/cli/kb.ts` | TODO |
| Auto-discovery scanner (§15.6) | `src/core/kb/discover.ts` | TODO |
| Unit + integration tests | `tests/{unit,integration}/kb/**` | TODO |

## Verified Evidence

— (none yet — WP in `spec` stage)

## Agents

| Stage | Agent | Reason |
|-------|-------|--------|
| design | `architect` | KB seam contract design |
| impl | `typescript-pro` | strict ESM, Zod boundary parse |
| review | `code-reviewer`, `security-reviewer` | privacy invariants |

## Review

| Reviewer | Verdict | Notes |
|----------|---------|-------|
| — | — | — |

## Follow-up Issues

| Issue | Severity | Status |
|-------|----------|--------|
| — | — | — |
