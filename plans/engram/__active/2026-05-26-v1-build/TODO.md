---
name: 2026-05-26-v1-build-todo
title: TODO — engram v1 — bottom-up build
type: todo
plan: 2026-05-26-v1-build
updated: 2026-05-26
---
<!-- Template: TODO v2 (frontmatter-first) -->

# TODO — engram v1 — bottom-up build

Mark `[x]` done, `[~]` in-progress. Never delete a completed item. Bump
`updated:` on every change.

## Active Set

> Stage column derived from WP frontmatter — do not edit by hand.

> Stages reflect WP frontmatter as of the 2026-05-26 review. `grill?` = needs
> `grill-with-memory` before execution (see REVIEW_REPORT / OPEN_QUESTIONS).

| WP | Title | Severity | Stage / Next Action |
|----|-------|----------|---------------------|
| WP13 | Daemon process envelope + operational layer (engramd lifecycle, migrate, install, logger, retry) | HIGH | draft |
| WP00 | Repo scaffold + tooling baseline | LOW | spec — execute (lowest risk) |
| WP01 | Core scaffold (store, schemas, AppLog, jobs, OCC, CoreService, doctor, init) | HIGH | spec — human gate spec→hardened; blocks everything |
| WP02 | Plugin host + LlmPlugin (Vercel AI SDK) | HIGH | ready — re-state Spike-1b claim (OQ-06) before exec |
| WP03 | Retrieval plugin (QMD in-process) + stats sidecar | HIGH | ready |
| WP04 | Scoring engine + recall (degradation chain) | HIGH | ready — needs λ_base (OQ-04) before `engine.ts` |
| WP05 | MCP server + CoreService facade (16 verbs, bearer) | HIGH | ready — resolve `forbidden` code (OQ-07) |
| WP06 | Capture + CaptureIntake + staging | HIGH | **spec — GRILL** (OQ-01 drop-vs-strip; OQ-05 entropy) |
| WP07 | Ingest worker (graphify GraphPlugin, Ollama) | MEDIUM | spec — grill? (OQ-06 model; graphify probe) |
| WP08 | Dreaming worker + orchestrator | HIGH | **spec — GRILL** (OQ-02/03/06) |
| WP09 | Threat-model hardening (planted-attack tests) | HIGH | ready |
| WP10 | Governance + cascade delete | MEDIUM | spec — owns `lifecycle revive` |
| WP11 | E2E verification (18 success criteria) | HIGH | ready — reconcile no-skip gate vs Ollama (OQ-06) |
| WP12 | Current memory-system removal (9-stage cutover) | MEDIUM | **spec — GRILL** (manual+sign-off; OQ-10) |

---

## Phase 1: Planning & Research

- [ ] Analyze current state of engram
- [ ] Identify gaps and requirements
- [ ] Write OVERVIEW.md, FINDINGS.md, initial WPs, VERIFICATION.md, TODO.md

### Open Questions (block implementation)

| ID | Question | Blocks |
|----|----------|--------|
|  |

---

## Phase 2: Implementation

- [ ] Implement plan WPs

---

## Phase 3: Verification & Review

- [ ] Run build/test gates (repo-adaptive)
- [ ] Run code-reviewer agent on all changes
- [ ] Run simplify pass
- [ ] Move completed plan to `__completed/`
