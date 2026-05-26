# ADR-0007: Distinct `forbidden` (capability) vs `scope-denied` (scope) access errors

- **Status:** Accepted
- **Date:** 2026-05-26
- **Related:** SPEC §6.3, §7.1, plan OQ-07 (2026-05-26 grilling), WP05

## Context

engram's access control fails for two structurally different reasons:

1. **Scope/visibility** — the caller is trying to act on a *target* it may not
   touch (a memory outside its `scope`, a `private` memory of another agent, a
   dreaming-memory whose `scope` excludes it).
2. **Capability** — the caller's bearer token lacks the *capability* the verb
   requires (e.g. calling `memory.remember` without `write`, or
   `governance_delete` without `govern`).

SPEC §6.3's verb table named only `scope-denied`, and WP05's facade used an
unspecified `forbidden` code for the capability case. The 2026-05-26 review
flagged the divergence (OQ-07). The choice: collapse both into `scope-denied`
(minimal surface), or ratify `forbidden` as a distinct code.

The two cases are genuinely different information to a client: "you asked for the
wrong thing" vs "you asked the right thing but lack permission." A client *can*
act differently — a missing capability is a provisioning problem (request a
broader token / `engram agent add`), whereas a scope denial is a targeting
problem (you reached for another agent's data). Conflating them hides that.

## Decision

- engram exposes **two distinct access-denial error codes**:
  - **`scope-denied`** — the caller is outside the target's `scope`/`visibility`.
  - **`forbidden`** — the caller's token lacks the required capability for the verb.
- Both are enforced by `CoreService` before any store primitive is touched
  (§7.1), and both emit an AppLog `mcp_denied` event carrying the reason.
- §6.3 verb-table error columns and §7.1 are amended to include `forbidden`
  where a capability is the gate.

## Consequences

- Clients/agents can distinguish "wrong target" (adjust the request) from "wrong
  permission" (obtain the capability) — better diagnostics and self-correction.
- One more public error code in the MCP contract; verb-table error columns grow.
  Manageable: it is additive and orthogonal to the existing codes.
- The AppLog already records both under `mcp_denied`; no new event type.
- WP05 phase-1 implements both; the verb table must mark which verbs can return
  `forbidden` (capability-gated) vs `scope-denied` (target-gated) vs both.

## Alternatives considered

- **`scope-denied` for both (collapse)** — smaller wire surface; rejected because
  it erases an actionable distinction (capability vs target) the caller can use,
  and because the scope-vs-capability detail would have to be dug out of AppLog
  rather than seen at the call site.
- **HTTP-style 403 with a sub-reason field** — would carry the distinction in a
  payload rather than a code; rejected for consistency with engram's existing
  string-code error vocabulary (`version-conflict`, `not-found`, etc.) and the
  MCP `result.isError` + code convention.
</content>
