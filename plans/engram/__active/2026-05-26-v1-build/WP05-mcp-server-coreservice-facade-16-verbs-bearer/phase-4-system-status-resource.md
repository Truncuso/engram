---
name: phase-4-system-status-resource
title: engram://system/status resource + sendResourceUpdated subscription + plugin health
type: phase
phase_status: pending
wp: wp05-mcp-server-coreservice-facade-16-verbs-bearer
goal: engram://system/status is registered as an MCP resource via registerResource; subscribers receive sendResourceUpdated notifications when plugin health changes; engram status CLI reads the same data and reports daemon healthy with per-plugin health.
verify: "npm test tests/integration/system-status — a client subscribes to engram://system/status; a simulated plugin health change triggers sendResourceUpdated with updated {plugins, worker, storePath, version}; engram status CLI exits 0 and prints plugin health lines."
---
<!-- Template: WP-folder PHASE v2 (frontmatter-first) -->

# Phase 4: engram://system/status resource + sendResourceUpdated + plugin health

**Goal:** `engram://system/status` is registered as an MCP resource (§6.3,
§10.3). Subscription via `sendResourceUpdated` lets monitoring clients learn of
plugin state changes without polling (Spike 2 CONFIRMED: `sendResourceUpdated`
works in `@modelcontextprotocol/sdk@1.29`). The `system.status` MCP verb and the
`engram status` CLI both read the same `SystemStatus` snapshot:
`{plugins:[{kind, health}], worker, storePath, version}`.

**Verify:** `npm test tests/integration/system-status` — a client subscribes;
simulating a plugin health change causes `sendResourceUpdated` to fire with the
updated status payload; `engram status` (CLI) exits 0 and prints one health line
per plugin (QMD, graphify, LLM).

## Steps

| Step | File | State |
|------|------|-------|
| `SystemStatus` type: `{plugins:[{kind, name, health:{ok,detail?}}], worker:{activeJobs}, storePath, version}` | `src/core/types/system-status.ts` | TODO |
| `CoreService.systemStatus()` — reads plugin health probes + jobs.db active count | `src/core/coreservice.ts` | TODO |
| `registerResource('engram://system/status', ...)` with snapshot read handler | `src/mcp/resources/status.ts` | TODO |
| Plugin health change hook: when any plugin health toggles, call `sendResourceUpdated('engram://system/status')` | `src/mcp/resources/status.ts` | TODO |
| `engram status` CLI command — calls `CoreService.systemStatus()`, prints formatted output | `src/cli/commands/status.ts` | TODO |
| Integration tests: subscribe; health-change event; CLI exit-0 + output | `tests/integration/system-status.test.ts` | TODO |

## Notes

`sendResourceUpdated` is called from the plugin host's health-change callback,
not on a poll timer — it fires exactly when a plugin transitions `ok↔degraded`.
The `system.status` MCP verb (registered in phase 3) calls `CoreService.systemStatus()`
synchronously; the resource subscription is the push path. Both share the same
snapshot function. This delivers SC-18: `engram status` reports daemon healthy
with plugin health for QMD, graphify, and LLM. This phase also completes **Milestone M2**:
daemon is MCP-accessible with bearer auth, 16 verbs, and observable status.
