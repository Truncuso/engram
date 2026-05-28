# ADR-0010: v1 dashboard — read-only React SPA, sigma.js graph view, bearer-gated on loopback

- **Status:** Accepted
- **Date:** 2026-05-28
- **Related:** SPEC v2.3 §1.2 (capability table), §8 (threat model), §9 (degraded mode),
  §11 (technology decisions); ADR-0007 (graphify graph artifact); WP22
  (milestone v1.3-ingest-formats-and-dashboard)

## Context

The SPEC v2.2 position deferred all dashboard/GUI work to v2. The v2.3 amendment
(2026-05-28) moved a read-only dashboard into v1 based on the user requirement:
browse memories, run semantic search, and view the knowledge graph without
leaving the terminal-side engram workflow. The knowledge-graph view is the
primary stated value — it requires a browser renderer (sigma.js); a TUI cannot
satisfy it. Reference implementations `nashsu/llm_wiki` and `Pratiyush/llm-wiki`
use the same React + sigma.js pattern, confirming the stack is proven.

Any new network surface must satisfy the §8 threat model: loopback-only binding
and the same bearer-token gate already used by the MCP HTTP endpoint. Reusing
that auth infrastructure is a hard requirement — no new auth code.

Editing memory, changing tags inline, submitting new ingest, and visualising
dreaming in real time introduce state-machine and conflict-resolution complexity
that would block shipping. The navigation + search + graph value is complete
without them.

## Decision

The v1 engram dashboard is a **read-only React SPA** served by engramd on
`127.0.0.1:<dashboard_port>` (separate port from MCP), bearer-gated using the
same token as the MCP endpoint. Editing, ingest UI, dreaming visualisation, and
review queue are explicitly deferred to v2.

**Stack:**
- React 18 + TypeScript + Vite (build toolchain)
- sigma.js for the knowledge-graph view (renders the graphify-derived graph
  artifact from ADR-0007)
- Tanstack Query as the thin async-state layer over engram's MCP HTTP endpoint
- Built artifact baked into the engram npm package; engramd serves it via a
  static file route

**Auth:** The bearer token is delivered to the browser once via a one-shot
`engram dashboard login` CLI handshake that sets a `HttpOnly; SameSite=Strict`
cookie on `127.0.0.1`. Subsequent requests present the cookie; engramd validates
it against the same secret as MCP bearer. Zero new auth code paths.

**Pages (v1):**
1. Memory browser — list/filter by type, track, scope, confidence, recency
2. Semantic search — freetext query → recall results with score breakdown
3. Knowledge graph — sigma.js rendering the graphify graph; read-only pan/zoom

**Degraded mode (SPEC §9 extension, SC-33):** when engramd is down or returning
503, the dashboard renders a static "daemon unavailable" page with the
last-known status surfaced from local browser cache. It never silently breaks or
returns a blank page.

**Explicitly out of v1 scope:** inline editing, tag mutation, ingest submission,
dreaming visualisation, review queue.

## Consequences

- **+** User gets browse + search + graph value in v1 without waiting for v2.
- **+** Bearer reuse means zero auth code duplication; MCP and dashboard share
  one token lifecycle.
- **+** Read-only surface eliminates state-sync, OCC, and conflict-resolution
  complexity; no server-side write paths are added.
- **−** Serving a SPA from engramd adds a small attack surface — mitigated by
  loopback-only binding and bearer gate (SPEC §8).
- **−** Bundled React build inflates the npm package — mitigated by code-splitting
  and lazy-loading the sigma.js graph view (heaviest chunk).

## Alternatives considered

- **Defer dashboard to v2** (SPEC v2.2 position) — rejected; the user requirement
  explicitly moves it into v1 and the read-only scope is shippable without
  blocking other v1 work.
- **TUI instead of web** — lower deps, but the knowledge-graph view (the primary
  stated value) requires a browser renderer; sigma.js is not portable to a TUI.
- **Electron / Tauri desktop app** — reference `nashsu/llm_wiki` uses Electron;
  rejected as overkill for v1. Bundling a desktop runtime adds significant
  dep weight; engramd already ships as a daemon, the browser surface is
  sufficient for v1.
- **Read-only + editing in same v1 scope** — rejected; editing doubles the attack
  surface, introduces write-path state machines, and would delay shipping the
  navigation + search + graph value the user actually needs now.
