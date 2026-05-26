---
name: wp00-repo-scaffold-tooling-baseline
title: Repo scaffold + tooling baseline
type: work-package
stage: spec
severity: LOW
created: 2026-05-26
updated: 2026-05-26
plan: 2026-05-26-v1-build
tags: [scaffold, tooling]
relationships:
  - blocks: [[wp01-core-scaffold-store-schemas-applog-jobs-occ-coreservice-doctor-init]]
sources: [SRC-01, SRC-03]
---
<!-- Template: WP v2 (frontmatter-first) -->

# WP00: Repo scaffold + tooling baseline

## Problem

engram needs a clean, modern, non-bloating TS repo before any module is built:
strict ESM tooling, the module skeleton, docs/ADRs, thin reuse-global `.claude/`
rules. Most of this is DONE (commit `aeffa3d`, repo `github.com/Truncuso/engram`);
this WP closes the remainder so the build starts green.

## Target Files

- `package.json`, `tsconfig.json`, `.gitignore`, `.editorconfig`, `.nvmrc`, `.env.example` — DONE
- `src/**` module skeleton (`.gitkeep`) — DONE
- `docs/engram-SPEC.md`, `docs/adr/0001-0004`, `docs/research/spikes-*.md` — DONE
- `CLAUDE.md`, `AGENTS.md`, `README.md`, `.claude/rules/engram-*.md` — DONE
- `eslint.config.js` + `prettier` config — TODO
- `vitest.config.ts` — TODO
- `.github/workflows/ci.yml` (typecheck + test on push) — TODO
- `npm install` + commit lockfile — TODO

## Verified Evidence

- `aeffa3d` — initial scaffold commit; `git remote` → `github.com/Truncuso/engram`.

## Implementation Steps

| Step | File | State |
|------|------|-------|
| ESLint flat config + prettier | `eslint.config.js`, `.prettierrc` | TODO |
| Vitest config (unit/integration/e2e roots) | `vitest.config.ts` | TODO |
| `npm install`, commit `package-lock.json` | lockfile | TODO |
| CI: typecheck + test | `.github/workflows/ci.yml` | TODO |

## Verification

### Build Gates

| Gate | Command | Expected |
|------|---------|----------|
| typecheck | `npm run typecheck` | exit 0 (empty src ok) |
| test | `npm test` | exit 0 (no tests yet) |
| lint | `npm run lint` | exit 0 |
| CI | push → Actions | green |

### Specific Tests

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| W00-1 | fresh clone builds | `npm ci && npm run typecheck` exit 0 | manual/CI |

## Recommended Agents

| Phase | Agent | Rationale |
|-------|-------|-----------|
| tooling | `devops-engineer` | CI/lint baseline |

## Review

| Reviewer | Verdict | Notes |
|-----------|---------|-------|
| — | — | — |

## Follow-up Issues

| Issue | Severity | Status |
|-------|----------|--------|
| — | — | — |
