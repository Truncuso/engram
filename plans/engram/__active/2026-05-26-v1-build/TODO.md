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

| WP | Title | Severity | Stage / Next Action |
|----|-------|----------|---------------------|
| WP12 | Current memory-system removal (9-stage cutover) | MEDIUM | draft |
| WP11 | E2E verification (18 success criteria) | HIGH | draft |
| WP10 | Governance + cascade delete | MEDIUM | draft |
| WP09 | Threat-model hardening (planted-attack tests) | HIGH | draft |
| WP08 | Dreaming worker + orchestrator | HIGH | draft |
| WP07 | Ingest worker (graphify GraphPlugin, Ollama) | MEDIUM | draft |
| WP06 | Capture + CaptureIntake + staging | HIGH | draft |
| WP05 | MCP server + CoreService facade (16 verbs, bearer) | HIGH | draft |
| WP04 | Scoring engine + recall (degradation chain) | HIGH | draft |
| WP03 | Retrieval plugin (QMD in-process) + stats sidecar | HIGH | draft |
| WP02 | Plugin host + LlmPlugin (Vercel AI SDK) | HIGH | draft |
| WP01 | Core scaffold (store, schemas, AppLog, jobs, OCC, CoreService, doctor, init) | HIGH | draft |
| WP00 | Repo scaffold + tooling baseline | LOW | draft |
|  |

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
