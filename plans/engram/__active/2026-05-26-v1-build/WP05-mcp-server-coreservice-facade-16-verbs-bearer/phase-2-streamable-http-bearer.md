---
name: phase-2-streamable-http-bearer
title: StreamableHTTPServerTransport on 127.0.0.1 + requireBearerAuth + token issuance/storage + multi-session
type: phase
phase_status: pending
wp: wp05-mcp-server-coreservice-facade-16-verbs-bearer
goal: The MCP server binds to 127.0.0.1 (DNS-rebinding protection) with StreamableHTTPServerTransport and requireBearerAuth middleware; token issuance stores per-agent files at 0600; multi-session is enabled via sessionIdGenerator; agent identity resolves to (agentId, capabilities, scope).
verify: "npm test tests/integration/mcp-auth — missing/bad token → HTTP 401; valid token → 200 + mcp-session-id header; two simultaneous sessions maintain independent state; token revocation causes subsequent calls with that token to 401."
---
<!-- Template: WP-folder PHASE v2 (frontmatter-first) -->

# Phase 2: StreamableHTTPServerTransport + requireBearerAuth + token issuance/storage + multi-session

**Goal:** The MCP HTTP server binds to `127.0.0.1` only (§6.3 DNS-rebinding
protection). Uses `@modelcontextprotocol/sdk@1.29` `StreamableHTTPServerTransport`
with `sessionIdGenerator` for multi-session support (Spike 2 CONFIRMED). Bearer
auth uses the SDK's `requireBearerAuth` middleware — NOT DIY. Per-agent token
files live at `~/.engram/agent-secrets/<agent-id>.token` at `0600` permissions,
daemon-owned. The middleware resolves a valid token to `AgentContext`
`{agentId, capabilities, scope}`.

**Verify:** `npm test tests/integration/mcp-auth` — absent/invalid token → HTTP
401; valid token → HTTP 200 + `mcp-session-id` header; two concurrent sessions
maintain independent context; revoking a token causes the next call to return 401.

## Steps

| Step | File | State |
|------|------|-------|
| Express app + `StreamableHTTPServerTransport` with `sessionIdGenerator` (Spike 2 pattern) | `src/mcp/server.ts` | TODO |
| Bind to `127.0.0.1` only; retry 3× on port-in-use (§9.9 step 5) | `src/mcp/server.ts` | TODO |
| `requireBearerAuth` middleware wired; resolves token → `AgentContext` | `src/mcp/server.ts` | TODO |
| Token issuance: mint opaque token, write `~/.engram/agent-secrets/<id>.token` at `0600` | `src/mcp/auth.ts` | TODO |
| Token lookup: read token file, verify, return `AgentContext`; cache in memory for active sessions | `src/mcp/auth.ts` | TODO |
| Token revocation: `engram agent revoke <id>` removes file + invalidates cached sessions | `src/mcp/auth.ts` | TODO |
| Integration tests: absent/bad/valid token; multi-session; revocation | `tests/integration/mcp-auth.test.ts` | TODO |

## Notes

Spike 2 live evidence: `127.0.0.1:8765`, bad token → **401**; valid token →
**200 + mcp-session-id**. The SDK's `requireBearerAuth` is in
`server/auth/middleware/bearerAuth.js` — use it, do not reimplement. Token files
use `fs.writeFileSync` with `mode: 0o600`; on Linux confirm with `stat`. Default
session agent capability set: `{read, write, dream}` scoped to itself (§6.3).
`govern` capability is granted only to `memory.init` / `engram agent add` flows.
