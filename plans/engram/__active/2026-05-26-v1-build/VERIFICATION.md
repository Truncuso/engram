---
name: 2026-05-26-v1-build-verification
title: Verification Matrix — engram v1 — bottom-up build
type: verification
plan: 2026-05-26-v1-build
updated: 2026-05-26
---
<!-- Template: VERIFICATION v2 (frontmatter-first) -->

# Verification Matrix — engram v1 — bottom-up build

## Build Verification Gates (All WPs)

Every WP must pass these before its `stage:` advances to `verified`
(per engram-typescript rule: strict TS, Vitest, ≥80% coverage).

| Gate | Command | Expected |
|------|---------|----------|
| Typecheck | `npm run typecheck` (`tsc --noEmit`) | exit 0, 0 errors |
| Lint | `npm run lint` (`eslint`) | exit 0, 0 errors |
| Unit + integration | `npm test` (`vitest run`) | all green |
| Coverage | `vitest run --coverage` | ≥ 80% |
| E2E (WP11 gate) | `npm run e2e` | 18 SC files, 0 failures, 0 skips* |
| CI | push → GitHub Actions | green |

> *The "0 skips" E2E gate is in tension with the Ollama hard-prereq for
> SC-3/5/6/16 (OQ-06 / FINDINGS W-9). Resolve the CI strategy (self-hosted
> Ollama runner, or recorded `graph.json` fixtures + cloud-model fallback) before
> WP11 runs, or the final gate cannot pass on a bare runner.

---

## WP01: engram v1 — bottom-up build

| ID | Test | Expected Result | Method | Result |
|----|------|-----------------|--------|--------|
|  |

> `Result` column: `pass | fail | pending`. Updated during `/verify`.

---

## Live / CLI Verification

| ID | Command / Action | Expected Observation | Result |
|----|------------------|----------------------|--------|
|  |

---

## Verification Status

| WP | Build | Tests | Live/CLI | Review | Simplify | Overall |
|----|-------|-------|----------|--------|----------|---------|
|  |
