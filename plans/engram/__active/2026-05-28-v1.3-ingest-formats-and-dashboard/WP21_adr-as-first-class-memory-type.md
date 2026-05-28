---
name: wp21-adr-as-first-class-memory-type
title: ADR-as-first-class-memory-type (KbPlugin + per-project adr/ directory + procedural-track recall)
type: work-package
severity: HIGH
created: 2026-05-28
updated: 2026-05-28
plan: 2026-05-28-v1.3-ingest-formats-and-dashboard
tags: [adr, memory-type, kbplugin, recall]
status: TODO
relationships:
  blocked_by: [wp13-multi-kb-registry-and-kbplugin-seam]
  blocks: [wp23-v1-3-e2e-acceptance-gate]
sources: [SRC-ADR-0009, SRC-SPEC-v2.3-SC-31]
---
<!-- Template: WP v2 (frontmatter-first) -->

# WP21: ADR-as-first-class-memory-type

SC-31 (SPEC v2.3 §15): a new ADR written into a project's `adr/` directory is
auto-ingested by the ADR KbPlugin and surfaces on `recall(track: "procedural")`
with Decision/Context/Consequences sections preserved.

Per ADR-0009, ADRs are a `KbPlugin` instance — **not** a new top-level memory
type — preserving the §3 four-type invariant. The plugin registers as `adr`,
one instance per project scope, and reuses the registration, lifecycle, and
recall plumbing from ADR-0005 and ADR-0006 at zero cost to the microkernel.

This WP is **independent** of the PDF/YouTube ingest WPs (WP18–20). The only
shared dependency is the KbPlugin registry seam from WP13.

## Scope

- `src/plugins/kb/adr/schema.ts` — Zod schema: frontmatter (`adr_id`,
  `adr_status`, `decision`, `supersedes`, `related`) + body section detection
  (Context, Decision, Consequences, Alternatives)
- `src/plugins/kb/adr/matcher.ts` — detects ADR files at `<repo>/adr/NNNN-*.md`,
  `<repo>/docs/adr/NNNN-*.md`, `<repo>/decisions/NNNN-*.md`; configurable via
  `kb.adr.paths` registry row (`adr_glob`)
- `src/plugins/kb/adr/index.ts` — `KbPlugin` instance; registers as `adr`;
  per-KB lifecycle per ADR-0006; extends `validate()` to per-file granularity
  (`validate(file) → ok | quarantine`) as required by ADR-0009 §Consequences
- `src/plugins/kb/adr/extractor.ts` — yields one memory per ADR with
  `kind: "adr"` discriminator; preserves `adr_id`, `adr_status`, `decision`,
  `supersedes`, `related`, and full section bodies; sets `type: procedural`,
  `o/: ingested`, `confidence: 0.9` (accepted) / `0.5` (proposed/missing fm)
- Recall integration — include ADR memories in `procedural` track when
  `kbs: ["adr"]` or `kbs: ["*"]`; `kind: "adr"` discriminator flows to
  callers for optional branch/render
- Cross-link support — `related: [memory:…]` in ADR frontmatter wired through
  bridge layer (ADR-0007); `engram memory why <id>` traverses the links
- Per-project `<repo>/.engram/kb/adr/` created on first ADR detection;
  memories also written to `memories/procedural/` for global scoring + dreaming

## AppLog Events

| Event | Trigger |
|-------|---------|
| `kb.adr.detected` | Matcher finds a new ADR file |
| `kb.adr.ingested` | Memory written successfully |
| `kb.adr.rebuilt` | Full per-file re-ingest on file change |
| `kb.adr.schema_failed` | Frontmatter validation fails (field-level error) |

Failure mode: malformed frontmatter → `kb.adr.schema_failed` with field-level
Zod path; ADR is quarantined (logged, **not** silently dropped).

## Implementation Tasks

| # | Task | File | Status |
|---|------|------|--------|
| 1 | Define Zod schema for frontmatter + body section detection | `src/plugins/kb/adr/schema.ts` | TODO |
| 2 | Implement matcher; configurable glob via `kb.adr.paths` | `src/plugins/kb/adr/matcher.ts` | TODO |
| 3 | Implement `KbPlugin` (`manifest`, `validate`, `layout`, `ingest`, `lifecycleJobs`); extend `validate` to per-file | `src/plugins/kb/adr/index.ts` | TODO |
| 4 | Implement extractor; degrade gracefully on absent frontmatter (body-only parse, `confidence: 0.5`) | `src/plugins/kb/adr/extractor.ts` | TODO |
| 5 | Modify recall fan-out to include ADR memories in `procedural` track | `src/core/recall/fan-out.ts` | TODO |
| 6 | Wire cross-links from `related:` through bridge layer (ADR-0007) | `src/core/kb/bridges.ts` | TODO |
| 7 | Register `adr` plugin in plugin host startup | `src/plugins/kb/adr/index.ts` | TODO |
| 8 | Emit AppLog events (detected, ingested, rebuilt, schema_failed) | `src/plugins/kb/adr/index.ts` | TODO |
| 9 | Unit tests: schema validation, matcher, extractor | `tests/unit/kb/adr/**` | TODO |
| 10 | Integration test: fixture ADR drop → recall returns it in procedural track | `tests/integration/kb-adr-recall.test.ts` | TODO |
| 11 | Edge-case tests: malformed ADR, superseded ADRs, ADR with no `related:` | `tests/unit/kb/adr/edge-cases.test.ts` | TODO |
| 12 | Hook trace tests (kb.adr.detected / kb.adr.ingested in trace journal) | `tests/unit/kb/adr/hooks.test.ts` | TODO |

## Verification Matrix

| ID | Scenario | Expected | Method |
|----|----------|----------|--------|
| W21-1 | Drop fixture ADR into `<repo>/adr/`; call recall | Memory returned in `procedural` track | Integration test |
| W21-2 | Recall memory body inspected | Decision/Context/Consequences sections intact | Integration test |
| W21-3 | Superseded ADR ingested | Memory present with `adr_status: superseded`; recallable with status flag | Unit + integration |
| W21-4 | Malformed frontmatter file ingested | `kb.adr.schema_failed` fired with field-level Zod path; file quarantined | Unit test |
| W21-5 | ADR has `related: ["mem_01HX…"]` | Bridge layer surfaces link; `engram memory why <id>` traverses it | Integration test |
| W21-6 | `kb.adr.paths` overridden to `decisions/` | Matcher detects files under `decisions/NNNN-*.md` | Unit test |
| W21-7 | `memory.types()` called after ADR plugin registered | Returns exactly 4 types; `adr` is not a fifth | Unit test |

## Risk & Mitigation

| Risk | Mitigation |
|------|-----------|
| ADR naming conventions vary per project | Configurable `adr_glob` in KB registry row |
| Existing ADRs lack frontmatter | Degrade gracefully: body-only parse, infer `adr_status` from heading text, set `confidence: 0.5` |
| Reentrant rebuild on rapid file change | Per-KB lifecycle (ADR-0006) handles idempotent re-ingest |
| `validate(file)` API extension to `KbPlugin` contract | Small additive extension to ADR-0005 interface; WP13 must expose the hook |

## Recommended Agents

| Role | Agent | Notes |
|------|-------|-------|
| impl | `typescript-pro` | Zod schema + KbPlugin contract; strict ESM |
| review | `code-reviewer` | Verify graceful degradation and quarantine path |

## Review

| Reviewer | Verdict | Notes |
|----------|---------|-------|
| — | — | — |

---

## Follow-up Issues

| Issue | Severity | Status |
|-------|----------|--------|
| — | — | — |
