---
name: wp17-episodic-semantic-linkage-hardening
title: Episodic↔semantic linkage hardening (`derived_from` mandatory + counterfactual gate suite + `engram memory why`)
type: work-package
stage: spec
severity: HIGH
created: 2026-05-27
updated: 2026-05-27
plan: 2026-05-27-v1.2-multi-kb-and-skills
tags: [provenance, dreaming, counterfactual, episodic]
relationships:
  - blocked-by: [[wp13-multi-kb-registry-and-kbplugin-seam]]
  - blocks: [[wp16-engram-skill-subsystem-and-installer]]
sources: [SRC-01, SRC-03]
---
<!-- Template: WP v2 (frontmatter-first) -->

# WP17: Episodic↔semantic linkage hardening

## Problem

v2.1 says dreaming products carry `derived_from` backlinks (R-4); that
property is recommended but not enforced. With multi-KB instances (WP13), a
derived semantic memory in the project KB can reference an episodic in the
`agent-self` KB or in a different `markdown-store` — the provenance chain
becomes cross-KB. WP17 promotes `derived_from` to a tested contract,
hardens the counterfactual gate (R-4) with a regression suite, adds
`engram memory why <id>` to walk the chain, and renders cross-KB chains via
the bridge layer (WP15).

This WP is what makes the user's "connecting events in time" requirement
explicit and auditable.

## Target Files

- `src/schemas/memory.ts` — make `derived_from` required for memories whose
  `origin = dreaming-merge`. Reject empty arrays.
- `src/core/scoring/validate.ts` — validation extended to enforce the rule.
- `src/worker/dream/merge.ts` — emit `derived_from` populated from the
  episodic IDs consumed by the merge.
- `src/worker/dream/counterfactual.ts` — counterfactual gate for procedural
  promotion. Failure leaves the memory at confidence 0.3, semantic.
- `src/core/mcp/verbs/memory.ts` — extend with `memory.why` verb.
- `src/cli/memory.ts` — `engram memory why <id>` walks the chain.
- `src/core/recall/chain-render.ts` — chain rendering across KB instances
  via bridges (`derived-from-citation` kind from WP15).
- `tests/regression/counterfactual-gate/**` — planted-promotion traps
  inspired by WP09's planted-attack suite.
- `tests/integration/derived-from-enforcement.test.ts`,
  `tests/integration/memory-why-cross-kb.test.ts`.

## Verification Gate

| # | Check | Test |
|---|-------|------|
| 1 | A merge job that produces a memory without `derived_from` FAILS schema validation (§9.4) and the job FAILS. | `tests/integration/derived-from-enforcement.test.ts` (SC-24) |
| 2 | The counterfactual gate keeps a planted procedural promotion at confidence 0.3 when the counterfactual is unsatisfied. | `tests/regression/counterfactual-gate/**` (SC-25) |
| 3 | `engram memory why <id>` walks `derived_from` recursively and renders the chain (episodic → semantic → procedural). | `tests/integration/memory-why-basic.test.ts` (SC-26) |
| 4 | `engram memory why <id>` follows the chain across KB instances via `derived-from-citation` bridges (WP15). | `tests/integration/memory-why-cross-kb.test.ts` |
| 5 | Episodics carry `session_id` and `tool_use_id` where available; the chain renderer includes them when present. | `tests/integration/memory-why-session-correlation.test.ts` |
| 6 | The regression suite is part of CI; a new planted trap can be added by dropping a YAML fixture. | `tests/regression/counterfactual-gate/README.md` + CI wiring |
| 7 | Existing v1 dreaming output continues to validate after the schema change (no false regressions). | `tests/integration/dream-regression-v1.test.ts` |

## Implementation Steps

| Step | File | State |
|------|------|-------|
| Make `derived_from` mandatory for `origin = dreaming-merge` | `src/schemas/memory.ts` | TODO |
| Wire validation into merge worker | `src/worker/dream/merge.ts` | TODO |
| Counterfactual gate implementation | `src/worker/dream/counterfactual.ts` | TODO |
| `memory.why` verb + CLI | `src/core/mcp/verbs/memory.ts`, `src/cli/memory.ts` | TODO |
| Cross-KB chain renderer (via bridges from WP15) | `src/core/recall/chain-render.ts` | TODO |
| Counterfactual regression suite + CI wiring | `tests/regression/counterfactual-gate/**` | TODO |
| Integration tests | `tests/integration/**` | TODO |

## Verified Evidence

— (none yet — WP in `spec` stage)

## Agents

| Stage | Agent | Reason |
|-------|-------|--------|
| impl | `typescript-pro` | schema + worker change |
| design | `architect-reviewer` | counterfactual-gate semantics |
| review | `code-reviewer`, `security-reviewer` | merge-validation invariants |

## Review

| Reviewer | Verdict | Notes |
|----------|---------|-------|
| — | — | — |

## Follow-up Issues

| Issue | Severity | Status |
|-------|----------|--------|
| — | — | — |
