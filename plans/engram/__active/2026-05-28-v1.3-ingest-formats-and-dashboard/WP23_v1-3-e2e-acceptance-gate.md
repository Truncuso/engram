---
name: wp23-v1-3-e2e-acceptance-gate
title: v1.3 milestone E2E acceptance gate (SC-27 through SC-35 + hook test suite)
type: work-package
created: 2026-05-28
updated: 2026-05-28
plan: 2026-05-28-v1.3-ingest-formats-and-dashboard
tags: [e2e, verification, acceptance, hooks, success-criteria, gate]
relationships:
  - blocked_by: [[wp18-pdf-book-ingest-pipeline]]
  - blocked_by: [[wp19-youtube-transcript-ingest]]
  - blocked_by: [[wp20-transcript-file-ingest]]
  - blocked_by: [[wp21-adr-as-first-class-memory-type]]
  - blocked_by: [[wp22-read-only-dashboard-v1]]
sources: [SRC-SPEC-v2.3-SC-27, SRC-SPEC-v2.3-SC-28, SRC-SPEC-v2.3-SC-29, SRC-SPEC-v2.3-SC-30, SRC-SPEC-v2.3-SC-31, SRC-SPEC-v2.3-SC-32, SRC-SPEC-v2.3-SC-33, SRC-SPEC-v2.3-SC-34, SRC-SPEC-v2.3-SC-35]
stage: spec
status: TODO
---
<!-- Template: WP v2 (frontmatter-first) -->

# WP23: v1.3 E2E acceptance gate

## Why

v1.3 is not accepted until every SPEC v2.3 success criterion SC-27 through SC-35
passes an automated E2E test, **plus** the explicit hook test suite that verifies
the full capture chain (capture hook fires → staging → dream worker → typed memory
→ recall) and its failure mode (filter drop → AppLog records it, never silent —
the §6.1 invariant). This WP is the terminal gate for the v1.3 milestone; it owns
the test scaffolding, fixtures, acceptance script, and CI integration. It mirrors
WP11 (the v1 gate) and reuses the same test infrastructure.

## Deliverables

| Item | File | Status |
|------|------|--------|
| Per-SC e2e test files | `tests/e2e/v1.3/sc27-dashboard.spec.ts` … `sc35-install.spec.ts` | TODO |
| Shared fixtures (small PDF, mock YouTube transcript, .vtt/.srt/.txt, fixture ADR, fixture store) | `tests/e2e/v1.3/fixtures/` | TODO |
| Hook test suite (capture → staging → dream → recall + filter-drop injection) | `tests/e2e/v1.3/sc32-hooks.spec.ts` | TODO |
| Acceptance script (per-SC pass/fail, non-zero on any failure) | `scripts/verify-v1.3.sh` | TODO |
| CI workflow | `.github/workflows/v1.3-acceptance.yml` | TODO |
| Acceptance report (filled at gate close) | `plans/engram/__active/2026-05-28-v1.3-ingest-formats-and-dashboard/ACCEPTANCE.md` | TODO |

## Approach

1. Implement per-SC test files in dependency order (after WP18–22 are merged). One file per SC:
   - `sc27-dashboard.spec.ts`, `sc28-pdf.spec.ts`, `sc29-youtube.spec.ts`, `sc30-transcript-file.spec.ts`, `sc31-adr.spec.ts`, `sc32-hooks.spec.ts`, `sc33-dashboard-degraded.spec.ts`, `sc34-youtube-fallback-gate.spec.ts`, `sc35-install.spec.ts`
2. Each test spins up a **real engramd** in an isolated config (test-store dir, test bearer token, ephemeral ports). Assertions go through MCP HTTP — the same surface a real agent uses.
3. The hook test suite (`sc32-hooks.spec.ts`) asserts:
   - **happy path:** capture hook fires on a simulated Claude Code session → observation lands in `staging/` → dream worker promotes it to a typed memory → recall returns it; AppLog shows all four stages in order with no gaps.
   - **failure injection:** the privacy filter blocks an observation → AppLog records a `capture.filter.drop` event with a reason; the observation is NOT silently lost and NO memory is created (the §6.1 invariant).
4. Degraded-mode test (`sc25`) kills engramd mid-session and reasserts the dashboard fallback page (SC-33, ties to WP22 W22-3).
5. Fixtures are small and committed; large PDFs/videos are NOT committed — `fixtures/README.md` links to source URLs. The mock YouTube transcript is a committed JSON fixture so CI never hits the network.
6. `scripts/verify-v1.3.sh` runs the full suite and prints a per-SC table; it exits 0 only when EVERY SC passes. No quarantine list — a flaky test fails the gate.
7. CI workflow runs `verify-v1.3.sh` on PRs to `feat/*` and on push to `master`.
8. At gate close, populate `ACCEPTANCE.md` with per-SC evidence (test name, last run, result).

## Verified Evidence

_(filled at implementation time)_

## Quality Gates

| Gate | Command | Expected |
|------|---------|----------|
| acceptance | `bash scripts/verify-v1.3.sh` | exit 0 only when SC-27…SC-35 all pass |
| every SC covered | `grep -L "sc[0-9]" tests/e2e/v1.3/*.spec.ts` | no SC file missing |
| hook suite | `npm run test:e2e -- sc32-hooks` | happy path + filter-drop both green |
| CI | push to `feat/*` | `v1.3-acceptance` workflow green |

## Verification Matrix

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| W23-1 | `verify-v1.3.sh` gate | exits 0 only when all SC-27…SC-35 pass | script |
| W23-2 | SC coverage | every SC has ≥1 automated test | grep/CI |
| W23-3 | Hook suite | happy path + filter-drop failure injection (per §6.1) both pass | e2e |
| W23-4 | Acceptance report | `ACCEPTANCE.md` fully populated with per-SC evidence at gate close | manual |
| W23-5 | CI gate | workflow runs the gate on push to `feat/*` and `master` | CI |

## Risk & Mitigation

| Risk | Mitigation |
|------|------------|
| E2E flake from real subprocess timing | AppLog-based readiness checks, no fixed `sleep`s |
| Fixture drift | fixtures also exercised by unit tests in WP18–22 |
| Network dependence in CI | mock YouTube transcript is a committed fixture; fallback path disabled in CI default |
| Scope creep (adding v2-feature SCs) | v2 features (editing, cross-agent dreaming) are explicit OUT per SPEC §12.1; gate covers only SC-27…SC-35 |

## Recommended Agents

| Task | Agent | Why |
|------|-------|-----|
| Test design | `tdd-guide` | acceptance-test structure |
| Test plumbing | `typescript-pro` | engramd boot harness + assertions |
| CI workflow | `devops-engineer` | GitHub Actions gate |
| Final pass | `code-reviewer` | coverage + invariant check |

## Review

| Reviewer | Verdict | Notes |
|----------|---------|-------|
| — | — | — |

---

## Follow-up Issues

| Issue | Severity | Status |
|-------|----------|--------|
| — | — | — |
