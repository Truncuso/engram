---
name: wp05-mcp-server-coreservice-facade-16-verbs-bearer
title: MCP server + CoreService facade (16 verbs, bearer)
type: work-package
stage: ready
severity: HIGH
created: 2026-05-26
updated: 2026-05-26
plan: 2026-05-26-v1-build
tags: [mcp, api, auth]
relationships:
  - depends_on: [[wp04-scoring-engine-recall-degradation-chain]]
  - blocks: [[wp06-capture-captureintake-staging]]
  - blocks: [[wp07-ingest-worker-graphify-graphplugin-ollama]]
sources: [SRC-01]
phases:
  - phase-1-coreservice-facade-envelope
  - phase-2-streamable-http-bearer
  - phase-3-sixteen-verbs
  - phase-4-system-status-resource
---
<!-- Template: WP-folder OVERVIEW v2 (frontmatter-first) -->

# WP05: MCP server + CoreService facade (16 verbs, bearer)

> Folder work package. Phases live in `phase-N-<slug>.md`. `stage:` advances only
> when all phase `phase_status:` are `done`.

## Problem

The daemon has a working kernel (store, scoring, OCC, plugins) after WP01–WP04,
but no network surface. This WP wires the transport-agnostic **CoreService facade**
— the single application layer all adapters (MCP, CLI, v2 dashboard) call into —
and exposes it over **Streamable HTTP on `127.0.0.1`** with bearer-token gating
using `@modelcontextprotocol/sdk@1.29` (Spike 2 CONFIRMED: `StreamableHTTPServerTransport`,
`requireBearerAuth`, `sendResourceUpdated`). Delivers **Milestone M2**: agents can
connect and issue all 16 verbs; `dream.*` verbs return `not-implemented` stubs
until WP08d.

SPEC refs: §2.1 (CoreService facade), §6.3 (Streamable HTTP, bearer, 16 verbs,
error semantics, C11), §7.1 (access control), §8.2–8.3 (bearer token 0600,
S-03), §9.9 step 5 (bind MCP server), §10.3 (system.status resource).

## Target Files

- `src/core/coreservice.ts` — transport-agnostic facade; `{ok, data, error}` envelope; access-control enforcement on every call (§7.1)
- `src/core/access-control.ts` — bearer token lookup → `(agentId, capabilities, scope)`; scope/visibility enforcement
- `src/mcp/server.ts` — `StreamableHTTPServerTransport` on `127.0.0.1`; Express app; `requireBearerAuth` middleware
- `src/mcp/auth.ts` — token issuance (`memory.init` / `engram agent add`), 0600 storage, revocation; agent identity → caps/scope
- `src/mcp/router.ts` — tool/resource registration; maps 16 MCP tool names to `CoreService` calls
- `src/mcp/verbs/memory.ts` — `memory.*` verb handlers (10 verbs); `outputSchema` per verb
- `src/mcp/verbs/dream.ts` — `dream.*` verb handlers (4 verbs); `dream.*` stubbed `not-implemented` until WP08d
- `src/mcp/verbs/system.ts` — `system.status` verb handler
- `src/mcp/resources/status.ts` — `engram://system/status` resource + `sendResourceUpdated` subscription
- `tests/integration/mcp-auth.test.ts` — bearer accept/deny; 16-verb smoke; C11 cross-agent reject
- `tests/integration/system-status.test.ts` — resource subscribe; plugin health events

## Phases

| Phase | Goal | Status |
|-------|------|--------|
| [phase-1](phase-1-coreservice-facade-envelope.md) | Transport-agnostic CoreService facade + `{ok,data,error}` envelope + access-control on every call | pending |
| [phase-2](phase-2-streamable-http-bearer.md) | `StreamableHTTPServerTransport` on `127.0.0.1`, `requireBearerAuth`, token issuance/storage 0600, multi-session | pending |
| [phase-3](phase-3-sixteen-verbs.md) | All 16 `memory.*`/`dream.*`/`system.*` verbs; `dream.*` stubbed; `outputSchema` per verb; C11 cross-agent reject | pending |
| [phase-4](phase-4-system-status-resource.md) | `engram://system/status` resource + `sendResourceUpdated` subscription; plugin health events | pending |

## Verification

> Required before `stage: done`. Aggregates per-phase `verify:` checks.

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| W05-1 | Bearer token absent/invalid | HTTP 401; no tool result | integration (SC-17) |
| W05-2 | Bearer token valid, `memory.recall` | HTTP 200 + mcp-session-id + tool result | integration (SC-17) |
| W05-3 | All 16 verbs registered and callable | Each returns a result (or `not-implemented` for `dream.*`) | integration (SC-17) |
| W05-4 | `dream.configure {mode: "cross-agent"}` | Returns `invalid-config` (C11) | integration (SC-10) |
| W05-5 | Agent B calls `dream.trigger` on dreaming-memory scoped to Agent A | Returns `scope-denied` (S-13) | integration (SC-10) |
| W05-6 | `system.status` resource subscribed; plugin health changes | `sendResourceUpdated` fires; subscriber receives updated health | integration (SC-18) |
| W05-7 | `engram status` shows plugin health via `system.status` | Healthy daemon reports plugin kind + health | e2e (SC-18) |

## Recommended Agents

| Phase | Agent | Rationale |
|-------|-------|-----------|
| phase-1 | `typescript-pro` | CoreService facade design; envelope type derivation |
| phase-2 | `security-reviewer` | Bearer auth implementation; 0600 token storage; S-03 |
| phase-3 | `typescript-pro` | 16-verb handler scaffolding; `outputSchema` per verb |
| phase-3 | `security-reviewer` | C11 cross-agent reject; S-13 scope-denied enforcement |
| phase-4 | `typescript-pro` | `sendResourceUpdated` subscription wiring |
| all | `code-reviewer`, `tdd-guide` | per-phase gate |
