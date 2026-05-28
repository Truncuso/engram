---
name: wp22-read-only-dashboard-v1
title: Read-only dashboard v1 (React + sigma.js, loopback + bearer-gated)
type: work-package
created: 2026-05-28
updated: 2026-05-28
plan: 2026-05-28-v1.3-ingest-formats-and-dashboard
tags: [dashboard, ui, react, sigma, frontend, knowledge-graph]
relationships:
  - blocked_by: [[wp05-mcp-server-coreservice-facade-16-verbs-bearer]]
  - sequenced_after: [[wp18-pdf-book-ingest-pipeline]]  # soft: needs at least one ingest format for demo/fixtures, not a hard dep on PDF specifically
  - blocks: [[wp23-v1-3-e2e-acceptance-gate]]
sources: [SRC-ADR-0010, SRC-SPEC-v2.3-SC-27, SRC-SPEC-v2.3-SC-33]
stage: spec
status: TODO
---
<!-- Template: WP v2 (frontmatter-first) -->

# WP22: Read-only dashboard v1 (React + sigma.js)

## Why

SPEC v2.3 SC-27: a read-only dashboard serves on `127.0.0.1:<dashboard_port>` with
a bearer-token gate; it lists memories with filters, runs a semantic-search box
that returns recall results with score breakdown, and renders a knowledge-graph
view of the graphify graph via sigma.js. SC-33: when engramd is down the dashboard
shows a degraded page, never a blank one. The decision is recorded in ADR-0010 —
read-only only (no editing, ingest, or review-queue UI; those are v2). The dashboard
is a thin adapter over the existing MCP CoreService facade (WP05); it holds **no
engram logic** of its own.

## Deliverables

| Item | File | Status |
|------|------|--------|
| React 18 + Vite + TS SPA scaffold | `src/dashboard/` | TODO |
| Static file route mounted by engramd (separate port from MCP) | `src/dashboard-server/static-route.ts` | TODO |
| Auth handshake CLI command (`engram dashboard login`) | `src/cli/commands/dashboard-login.ts` | TODO |
| Memory browser page (filter type/track/scope/confidence/recency) | `src/dashboard/pages/Browser.tsx` | TODO |
| Semantic search page (recall + score breakdown) | `src/dashboard/pages/Search.tsx` | TODO |
| Knowledge-graph page (sigma.js render of graphify graph) | `src/dashboard/pages/Graph.tsx` | TODO |
| Degraded-mode page | `src/dashboard/pages/Degraded.tsx` | TODO |
| Data layer (Tanstack Query over MCP HTTP) | `src/dashboard/lib/api.ts` | TODO |
| Build pipeline (`npm run build:dashboard` → `dist/dashboard/`, baked into package `files`) | `package.json`, `vite.config.ts` | TODO |
| AppLog events for dashboard access audit | `src/dashboard-server/static-route.ts` | TODO |
| Unit + integration + degraded-mode tests | `tests/integration/dashboard/`, `tests/e2e/v1.3/sc27-dashboard.spec.ts` | TODO |

## Approach

1. Scaffold `src/dashboard/` with Vite + React 18 + TypeScript. Keep it isolated from the daemon's TS — it builds to a static bundle.
2. Add `src/dashboard-server/static-route.ts`: engramd mounts the built bundle at `/dashboard` on a **separate port** from MCP (ADR-0010). Port selection: first free in `7050…7060`, actual port logged + exposed on the daemon status endpoint (OQ-V1.3-4).
3. `engram dashboard login` CLI: performs a one-shot handshake that sets a short-lived browser cookie wrapping the MCP bearer token. No new auth code — reuses the MCP bearer (WP05).
4. Build the three pages, each backed by exactly one Tanstack Query hook per MCP verb in `lib/api.ts`. No engram logic leaks into the frontend — every data fetch is an MCP call.
5. `Graph.tsx`: lazy-load sigma.js (code-split) and render the graphify `graph.json` the daemon already produces (ADR-0007, derived artifact). Nodes = memories, edges = relations; click a node → its detail panel (read-only).
6. `Degraded.tsx`: when the MCP health probe returns 503 or times out, render a static "daemon unavailable" page with last-known status from `~/.engram/dashboard-cache.json` (refreshed every 60s while healthy; staleness shown, not hidden — OQ-V1.3-5).
7. Emit AppLog events `dashboard.login`, `dashboard.session_start`, `dashboard.search`, `dashboard.graph_view`, `dashboard.session_end` so access is auditable.
8. Build pipeline: `npm run build:dashboard` produces `dist/dashboard/`, added to `package.json` `files` so the npm package ships the bundle.
9. Tests: component snapshots + API-client unit tests; an integration test that boots a real engramd on a test port serving the real React build; a degraded-mode test that kills engramd and asserts the fallback page; SC-27 e2e in `tests/e2e/v1.3/`.

## Verified Evidence

_(filled at implementation time)_

## Quality Gates

| Gate | Command | Expected |
|------|---------|----------|
| typecheck | `npm run typecheck` | exit 0 |
| dashboard build | `npm run build:dashboard` | `dist/dashboard/index.html` produced |
| test | `npm test -- dashboard` | exit 0 |
| lint | `npm run lint` | exit 0 |
| SC-27 e2e | `npm run test:e2e -- sc27-dashboard` | login → search → graph all green |

## Verification Matrix

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| W22-1 | SC-27 full flow | fresh engramd + dashboard build → login → search → graph view renders, all bearer-authed | e2e |
| W22-2 | Auth gate | no/wrong bearer → 401, no data | integration |
| W22-3 | SC-33 degraded mode | kill engramd → reload → degraded page with last-known status, not blank | integration |
| W22-4 | Graph render | sigma.js renders graphify `graph.json`, nodes navigable | e2e |
| W22-5 | Filters | type/track/scope/confidence/recency combinations narrow the list correctly | integration |
| W22-6 | Read-only invariant | no editing UI present; any POST/PUT/DELETE to MCP from the dashboard is a defect | integration |
| W22-7 | Access audit | dashboard activity produces the AppLog events in order | integration |

## Risk & Mitigation

| Risk | Mitigation |
|------|------------|
| SPA build inflates npm package | code-splitting; sigma.js lazy-loaded only on `/graph` |
| New HTTP attack surface | loopback-only bind + bearer-gated; covered by SPEC §8 threat model |
| sigma.js learning curve | 1-2 day spike if needed; fallback is a plain node-list view |
| Frontend accreting engram logic | hard rule: one query hook per MCP verb; no logic in the SPA |
| Scope creep into editing/ingest UI | explicit OUT per ADR-0010; review gate rejects any mutating UI |

## Recommended Agents

| Task | Agent | Why |
|------|-------|-----|
| React + TS implementation | `typescript-pro` | SPA + typed API client |
| Component decomposition / layout | `frontend-design` | production-quality UI, not AI-generated feel |
| Auth handshake review | `security-reviewer` | bearer cookie handshake is security-sensitive |
| Final pass | `code-reviewer` | quality + invariant check |

## Review

| Reviewer | Verdict | Notes |
|----------|---------|-------|
| — | — | — |

---

## Follow-up Issues

| Issue | Severity | Status |
|-------|----------|--------|
| — | — | — |
