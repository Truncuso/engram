---
name: phase-1-coreservice-facade-envelope
title: Transport-agnostic CoreService facade + {ok,data,error} envelope + access-control
type: phase
phase_status: pending
wp: wp05-mcp-server-coreservice-facade-16-verbs-bearer
goal: CoreService is the single application layer all adapters call; every method returns {ok,data,error}; access-control (scope/visibility/capability) is enforced on every call per §7.1 before touching any store primitive.
verify: "npm test tests/unit/coreservice-ac — a call with wrong scope returns {ok:false, error:{code:'scope-denied'}}; a call with insufficient capability returns {ok:false, error:{code:'forbidden'}}; a valid call with shared visibility returns data."
---
<!-- Template: WP-folder PHASE v2 (frontmatter-first) -->

# Phase 1: CoreService facade + {ok,data,error} envelope + access-control

**Goal:** `CoreService` is the transport-agnostic application layer. Every public
method returns the `{ok, data, error}` envelope (§6.3 error semantics). Access
control from §7.1 is enforced on every call — scope, visibility, and capability
checks fire before any store primitive is touched.

**Verify:** `npm test tests/unit/coreservice-ac` — a call with an out-of-scope
agent returns `{ok:false, error:{code:'scope-denied'}}`; a call missing the
required capability returns `{ok:false, error:{code:'forbidden'}}`; a valid call
with `visibility:shared` returns `{ok:true, data:{...}}`.

## Steps

| Step | File | State |
|------|------|-------|
| `ServiceEnvelope<T>` type: `{ok:true,data:T}` ∪ `{ok:false,error:{code,message}}` | `src/core/types/envelope.ts` | TODO |
| `AgentContext` type: `{agentId, capabilities, scope}` | `src/core/types/agent-context.ts` | TODO |
| `AccessControl.check(ctx, memory, requiredCap)` — scope+visibility+capability enforcement (§7.1) | `src/core/access-control.ts` | TODO |
| CoreService skeleton: constructor takes store + plugins; all verbs return `ServiceEnvelope<T>` | `src/core/coreservice.ts` | TODO |
| Wire access-control into `remember` / `get` / `list` / `recall` / `history` (delegates to WP01/WP04 primitives) | `src/core/coreservice.ts` | TODO |
| Unit tests: scope-denied, forbidden, shared-readable, private-invisible | `tests/unit/coreservice-ac.test.ts` | TODO |

## Notes

The `CoreService` is the only layer that enforces access control. MCP, CLI, and
the v2 dashboard are thin adapters that pass an `AgentContext` and unwrap the
envelope. This phase does NOT wire HTTP — that is phase 2. The facade calls into
the store/plugin primitives from WP01–WP04 but introduces no new store logic.
`hidden` memory lives in per-agent `0700` dirs (§7.1, S-14) — the filesystem layer
is set at write time (WP01); CoreService enforces at read time here.
