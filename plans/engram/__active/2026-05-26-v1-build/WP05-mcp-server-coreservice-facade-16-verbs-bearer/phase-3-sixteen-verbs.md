---
name: phase-3-sixteen-verbs
title: All 16 memory.*/dream.*/system.* verbs; dream.* stubbed not-implemented; outputSchema per verb; C11 cross-agent reject
type: phase
phase_status: pending
wp: wp05-mcp-server-coreservice-facade-16-verbs-bearer
goal: All 16 verbs from §6.3 are registered via registerTool with correct outputSchema; memory.* and system.status delegate to CoreService; dream.* return not-implemented stubs; dream.configure rejects config.mode=cross-agent with invalid-config (C11); dream.trigger enforces scope-denied for out-of-scope callers (S-13).
verify: "npm test tests/integration/mcp-verbs — all 16 tool names are present in server capabilities; memory.remember+recall round-trip; dream.configure{mode:cross-agent} → {isError:true, content:[{text:'invalid-config'}]}; dream.trigger from wrong agent → scope-denied."
---
<!-- Template: WP-folder PHASE v2 (frontmatter-first) -->

# Phase 3: All 16 verbs + outputSchema + C11 + scope-denied

**Goal:** All verbs from the §6.3 verb table are registered:
**11 `memory.*` + 5 `dream.*` + 1 `system.status` = 17 tool registrations**
(the spec's "16 verbs" counts the 11 memory + 5 dream; `system.status` is the
status verb/resource). `memory.*` (11 verbs: init, remember, update, recall, get,
list, forget, ingest, history, confirm, governance_delete) and `system.status`
delegate to `CoreService`. All 5 `dream.*` (`dream.list`, `dream.configure`,
`dream.trigger`, `dream.status`, `dream.result`) return `not-implemented` stubs
here (`dream.result` is naturally empty until WP08 produces job data).

Per §6.3 error semantics: protocol errors → JSON-RPC `error` field;
tool-execution errors → `result.isError: true + content`. Every verb has
`outputSchema` declared.

**C11:** `dream.configure` rejects `{config: {mode: "cross-agent"}}` with
`isError:true` + `invalid-config` (cross-agent is v2).

**S-13 / SC-10:** `dream.trigger` checks that the calling agent's `agentId` is
in the dreaming-memory's `scope`; if not → `scope-denied`.

**Verify:** `npm test tests/integration/mcp-verbs` — all 16 tool names in
capabilities; `memory.remember` + `memory.recall` round-trip; `dream.configure
{mode: "cross-agent"}` → `isError:true` with code `invalid-config`;
`dream.trigger` from an out-of-scope agent → `scope-denied`.

## Steps

| Step | File | State |
|------|------|-------|
| `memory.*` verb handlers (remember, update, recall, get, list, forget, ingest, history, confirm, governance_delete) — each calls `CoreService`, maps envelope to MCP result shape | `src/mcp/verbs/memory.ts` | TODO |
| `dream.*` verb handlers (list, configure, trigger, status, result) — stub not-implemented; C11 `dream.configure` cross-agent reject | `src/mcp/verbs/dream.ts` | TODO |
| `system.status` verb handler — calls `CoreService.systemStatus()` | `src/mcp/verbs/system.ts` | TODO |
| `outputSchema` Zod → JSON-schema per verb (§6.3 Returns column) | `src/mcp/verbs/memory.ts`, `src/mcp/verbs/dream.ts`, `src/mcp/verbs/system.ts` | TODO |
| `dream.trigger` scope check: `agentId ∈ dreamingMemory.scope` else `scope-denied` (S-13) | `src/mcp/verbs/dream.ts` | TODO |
| Router: `registerTool` for all 16 verbs + input schemas | `src/mcp/router.ts` | TODO |
| Integration tests: 16 verbs present; memory round-trip; C11; S-13 | `tests/integration/mcp-verbs.test.ts` | TODO |

## Notes

The verb table from §6.3 lists exactly 16 entries: `memory.init`,
`memory.remember`, `memory.update`, `memory.recall`, `memory.get`, `memory.list`,
`memory.forget`, `memory.ingest`, `memory.history`, `memory.confirm`,
`memory.governance_delete`, `dream.list`, `dream.configure`, `dream.trigger`,
`dream.status`, `dream.result`. All 16 registered here. `dream.*` stubs return
`{ok:false, error:{code:'not-implemented', message:'dream worker not available until WP08'}}`.
`AppLog` records `mcp_call` and `mcp_denied` events (C14) for every call.
