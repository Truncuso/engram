---
name: 2026-05-26-v1-build
title: engram v1 — bottom-up build
type: plan
status: active
feature: engram
created: 2026-05-26
updated: 2026-05-26
tags: [engram, memory, bottom-up, sdd]
work_packages: [wp00-repo-scaffold-tooling-baseline, wp01-core-scaffold-store-schemas-applog-jobs-occ-coreservice-doctor-init, wp02-plugin-host-llmplugin-vercel-ai-sdk, wp03-retrieval-plugin-qmd-in-process-stats-sidecar, wp04-scoring-engine-recall-degradation-chain, wp05-mcp-server-coreservice-facade-16-verbs-bearer, wp06-capture-captureintake-staging, wp07-ingest-worker-graphify-graphplugin-ollama, wp08-dreaming-worker-orchestrator, wp09-threat-model-hardening-planted-attack-tests, wp10-governance-cascade-delete, wp11-e2e-verification-18-success-criteria, wp12-current-memory-system-removal-9-stage-cutover]
relationships: []
sources: [SRC-01, SRC-02, SRC-03]
---
<!-- Template: OVERVIEW v2 (frontmatter-first) -->

# engram v1 — bottom-up build

## Executive Summary

Build **engram** — a standalone agentic memory system (TS/Node daemon + MCP
server + detached dreaming worker; Markdown files = source of truth; QMD
retrieval; graphify graph; scored recall) — **bottom-up, core first, one layer at
a time, with a verification gate per work package.** The authoritative design is
`docs/engram-SPEC.md` (v2.1); this plan operationalizes SPEC §13's 11 phases as
13 work packages, splitting the oversized dreaming phase (§13.8 → WP08 phases
8a–8d) and adding repo scaffold (WP00) and the current-memory-system removal
cutover (WP12) as a parallel track.

Design is complete and **de-risked** (4 spikes, `docs/research/spikes-2026-05-26.md`):
QMD confirmed in-process, MCP HTTP multi-session+bearer+subscriptions confirmed
live, Vercel AI SDK substrate confirmed, graphify mapped to a derived-index
GraphPlugin. No architectural unknowns remain.

**Three "it works" milestones** (see Execution Strategy): **M0** remember→recall
via grep (after WP01) · **M1** full scored recall (after WP04) · **M2**
agent-accessible over MCP (after WP05).

---

## Active Work Packages

> Derived from WP frontmatter `stage:` by `update_plan.py --sync`. Do not edit
> the Stage column by hand.

| WP | Title | Severity | Stage | Impact |
|----|-------|----------|-------|--------|
| WP13 | Daemon process envelope + operational layer (engramd lifecycle, migrate, install, logger, retry) | HIGH | hardened | — |
| WP00 | Repo scaffold + tooling baseline | LOW | spec | repo exists; build/test green |
| WP01 | Core scaffold (store, schemas, AppLog, jobs, OCC, CoreService, doctor, init) | HIGH | spec | **M0** remember→recall (grep) |
| WP02 | Plugin host + LlmPlugin (Vercel AI SDK) | HIGH | ready | LLM substrate, multi-provider |
| WP03 | Retrieval plugin (QMD in-process) + stats sidecar | HIGH | ready | relevance + recency source |
| WP04 | Scoring engine + recall (degradation chain) | HIGH | ready | **M1** full scored recall |
| WP05 | MCP server + CoreService facade (16 verbs, bearer) | HIGH | ready | **M2** agent-accessible |
| WP06 | Capture + CaptureIntake + staging | HIGH | hardened | sessions → staging |
| WP07 | Ingest worker (graphify GraphPlugin, Ollama) | MEDIUM | spec | raw → memories |
| WP08 | Dreaming worker + orchestrator (8a–8d) | HIGH | hardened | consolidation + learning |
| WP09 | Threat-model hardening (planted-attack tests) | HIGH | ready | 6 CRITICAL mitigations proven |
| WP10 | Governance + cascade delete | MEDIUM | spec | purge across all stores |
| WP11 | E2E verification (18 success criteria) | HIGH | ready | SPEC §12.3 automated |
| WP12 | Current memory-system removal (9-stage cutover) | MEDIUM | hardened | clean replacement, no broken state |

---

## Archived / Closed / Deleted

| WP | Outcome |
|----|---------|
| — | — |

---

## Corrections Log

| Previous Claim | Corrected Finding |
|----------------|-------------------|
| QMD may need MCP subprocess (SPEC §11.1) | In-process ESM library confirmed (Spike 1) — no subprocess |
| GraphPlugin does live edge mutation | graphify is extract→query; `ingestEdges`/`removeNode` are batch rebuild (Spike 3, ADR-0004) |
| MCP bearer auth is DIY | SDK ships `requireBearerAuth` + resource subscriptions (Spike 2) |

---

## Execution Strategy

Strict bottom-up. An edge `A → B` means B depends on A (B cannot reach `ready`
until A is `verified`). WP09/10/11 are verification/hardening passes over earlier
WPs. WP12 runs as a **parallel cutover track**, gated on engram milestones, never
removing the old system before its engram replacement is live.

```
WP00 -> WP01 -+-> WP02 -+
              |         +-> (WP02+WP03) -> WP04 -> WP05 -+-> WP06 -+
              +-> WP03 -+                                |         +-> WP08(8a->8b->8c->8d)
                                                         +-> WP07 -+         |
                                                                             +-> WP09
                                                                             +-> WP10
                                                                             +-> WP11

   M0 = after WP01 (remember->recall via grep, full frontmatter, OCC, doctor)
   M1 = after WP04 (I x R x Recency x m_v, degradation chain)
   M2 = after WP05 (MCP bearer, 16 verbs [dream.* stubbed], status resource)
```

> **Edges not drawn above (from per-WP frontmatter — authoritative):** WP07 also
> `depends_on: WP02` and WP08 also `depends_on: WP02` (both need the LLM/plugin
> host). The ASCII shows WP06/WP07 hanging off WP05 for readability; do not start
> WP07/WP08 until WP02 is also `verified`. WP02→WP07, WP02→WP08.
>
> **WP13 (daemon process envelope + operational layer)** — added by the 2026-05-26
> gap-scan; owns the `engramd` process, §9.9 startup/shutdown, schema-migration,
> logger, unified retry, S-07 store-open symlink scan, and the operational CLI.
> Gated on WP05 (assembled startup→MCP-bind) + WP01 (PID/spool primitives);
> **blocks WP11** (E2E needs a real startable daemon). WP05→WP13→WP11; WP01→WP13.

WP12 cutover (parallel, gated): stage1 build→WP05 · stage2 WP06 hooks coexist ·
stage3 migrate content (git-commit backup) · stage4 WP08 dreaming → re-point QMD
collections · stage5 swap SessionStart hook · stage6 skills cutover
(grill-with-memory rewire) · stage7 data removal · stage8 scripts · stage9
CLAUDE.md block. Detail in WP12.

**Key ordering notes (from build-order review):**
- WP05 registers all 16 MCP verbs; `dream.*` return `not-implemented` until WP08d.
- WP08 split because the monolithic dreaming phase was ~18 sub-systems:
  **8a** job state machine + spawning (zero LLM) · **8b** worker domain logic ·
  **8c** orchestrator merge + classification (testable with hand-written
  manifests, zero LLM) · **8d** triggers + rollback + `dream.*` verbs.
- Earliest end-to-end loop is **WP01** (grep recall), not WP04 — a working
  remember→recall before any plugin.

---

## Key Decisions

| Decision | Rationale | Date |
|----------|-----------|------|
| Bottom-up, gate per WP | core-first; cheap verification; early M0 loop (SPEC §13) | 2026-05-26 |
| Split dreaming into 8a–8d | single phase was unverifiable (~18 subsystems) | 2026-05-26 |
| TS single package, one Python seam | smallest repo; graphify only forced non-TS (ADR-0002) | 2026-05-26 |
| Vercel AI SDK, thin, no agent framework | multi-provider substrate; framework deferred (ADR-0003) | 2026-05-26 |
| QMD in-process; plugins = derived indexes | spike-confirmed; files-are-truth (ADR-0004) | 2026-05-26 |
| Removal as gated parallel cutover | never leave user without memory (review Part B) | 2026-05-26 |

---

## Project Sources

| SRC | Title | Type | Relevance |
|-----|-------|------|-----------|
| SRC-01 | `docs/engram-SPEC.md` (v2.1) | spec | authoritative design |
| SRC-02 | `docs/research/spikes-2026-05-26.md` | research | de-risking evidence |
| SRC-03 | `docs/adr/0001-0004` | decisions | locked rationale |

---

## Acceptance Criteria

- [ ] All WPs `verified`.
- [ ] All 18 SPEC §12.3 success criteria pass as automated tests (WP11).
- [ ] M0, M1, M2 milestones demonstrated.
- [ ] Current memory system fully cut over (WP12), no broken state at any stage.
- [ ] Plan moved to `__completed/`; no open questions remain.
