# ADR-0005: MCP transport — Streamable HTTP on loopback with SDK bearer auth

- **Status:** Accepted
- **Date:** 2026-05-26
- **Related:** SPEC §6.3, §8.2, §8.3 (S-03), §11.1, §11.2, Spike 2 (MCP SDK), ADR-0002

## Context

engram exposes its memory verbs to agents over MCP. Three transport shapes were
on the table, and the SPEC initially left auth as a possible DIY concern. The
choice is load-bearing for security (every memory read/write crosses it) and for
the daemon's shape (always-up, multiple concurrent agent sessions). Spike 2
resolved the open questions against a live server.

- **Spike 2 (MCP TS SDK):** `@modelcontextprotocol/sdk@1.29` ships
  `StreamableHTTPServerTransport` with a `sessionIdGenerator` (multi-session built
  in), first-class bearer auth (`requireBearerAuth` in
  `server/auth/middleware/bearerAuth.js` — not DIY), and resource subscriptions
  (`sendResourceUpdated`). A live Express + StreamableHTTP server on
  `127.0.0.1:8765` confirmed: bad token → 401; valid token → 200 + `mcp-session-id`.
- engramd is a long-running daemon serving multiple agent sessions concurrently;
  stdio (one client, one process lifetime) does not fit.
- The trust boundary is the local machine (§8.1); network attackers and
  multi-machine are out of scope for v1 (§8.6).

## Decision

- **Transport: Streamable HTTP bound to `127.0.0.1` only** (DNS-rebinding
  protection per the MCP spec). Not stdio; not a Unix domain socket.
- **Auth: the SDK's `requireBearerAuth` middleware** — a per-agent opaque bearer
  token, `0600`, daemon-owned, minted by `memory.init` / `engram agent add`,
  mapped to `(agentId, capabilities, scope)`, revocable. Not a hand-rolled auth
  layer (S-03).
- **Multi-session via `sessionIdGenerator`**; `engram://system/status` exposed as
  a subscribable MCP resource (`sendResourceUpdated`), no polling.
- engram's internal `{ok, data, error}` envelope lives in CoreService; the MCP
  adapter translates to native MCP shapes (protocol errors → JSON-RPC `error`;
  tool errors → `result.isError`).

## Consequences

- Always-up daemon can serve many agent sessions with isolated state — the right
  fit for a background memory service.
- Bearer auth, token issuance/revocation, and session isolation come from the
  maintained SDK, not custom code — smaller attack surface, less to get wrong.
- Loopback binding keeps the v1 trust boundary at the local machine; remote/multi
  -machine access is a deliberate v2 concern, not an accident of transport.
- A small HTTP/Express dependency in the core daemon (acceptable; the SDK
  already pulls it).

## Alternatives considered

- **stdio MCP transport** — rejected: one-client/one-process model is wrong for an
  always-up daemon serving multiple concurrent sessions; would force a process per
  session.
- **Unix domain socket + `SO_PEERCRED`** — rejected: no SDK support (custom
  transport cost), and loopback HTTP + bearer already gives session identity and
  isolation with maintained code. Peer-credential auth is stronger in theory but
  not worth a bespoke transport for v1's local trust boundary.
- **DIY bearer middleware** — rejected: the SDK ships `requireBearerAuth`;
  re-implementing auth is exactly the kind of security code best not hand-rolled.
</content>
