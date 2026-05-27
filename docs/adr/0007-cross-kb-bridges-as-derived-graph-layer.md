# ADR-0007: Cross-KB bridges are a derived graph layer, not live edges

- **Status:** Accepted
- **Date:** 2026-05-27
- **Related:** SPEC v2.2 §15.4; ADR-0004 (plugins are rebuildable derived indexes); ADR-0005 (KbPlugin); ADR-0006 (lifecycle workers)

## Context

ADR-0005 lets engram host multiple KBs. The user-facing value of multi-KB is
*connection*: a project memory that says "we picked SQLite for `jobs`" should
surface the matching wiki entry "SQLite WAL mode notes"; an agent-self
contextual memory about an in-progress task should link to the procedural
memory in a different KB that documents the workflow.

Two ways to model the connection:

1. **Live edges across KBs.** Each cross-KB link is a real edge written when
   the link is detected, removed when either side is archived, mutated when
   either side changes. Requires per-KB graphify graphs to support edge writes
   and a coordinator to keep them consistent.

2. **A derived bridge layer.** A separate index — `bridges.json` (per pair, or
   a global file) — is built from the per-KB graphify graphs by a scheduled
   job. Bridges are recomputed; they are not mutated in place.

ADR-0004 already commits engram to option 2 for the *intra-KB* graph (graphify
is `extract → graph.json → query`; no live edge mutation). Doing live edges
across KBs would contradict that, force a custom edge store, and pull live-
mutation semantics back into the graph layer.

## Decision

Cross-KB connections are a **derived bridge layer**:

- A new job kind `kb.connect.bridge` (ADR-0006) runs on a schedule per KB
  *pair* (or for the full set on weekly cadence by default).
- Inputs: the per-KB graphify graphs (already derived; ADR-0004).
- Output: a `bridges.json` artifact under `.engram/bridges/` containing edges
  of the form `{from: kb_id/node_id, to: kb_id/node_id, kind, score, why}`
  with bounded `kind` values: `entity-match`, `title-match`, `embedding-near`,
  `derived-from-citation`, `contradicts`.
- Recall does **not** fuse bridges into the scoring formula. Bridges are
  opt-in expansion: a caller passes `expand_via_bridges: true` and the recall
  pipeline does a second pass that walks bridge edges from the top-K
  per-KB results, returning an expanded set tagged with `via_bridge`.
- The scoring engine still owns the rank; bridges add candidates but do not
  reshape the formula. (Preserves SPEC §3.6 — no RRF, no competing rank.)
- `engram bridges show <id>` walks a bridge and explains *why* it exists
  (which `kind`, which score, which inputs).

Bridge build is **idempotent and rebuildable**. Deleting `bridges/` is safe;
`engram kb run bridge` rebuilds. A KB unregister removes that KB's bridges
without touching memory files.

## Consequences

- No live edge store. No coordinator. The bridge layer is the same shape as
  every other engram index: a derived, rebuildable artifact.
- Stale bridges are expected and accepted (last-run-wins). Latency is bounded
  by the schedule and the user can force-refresh.
- Bridge expansion is opt-in per query. Default recall stays unchanged — the
  multi-KB feature does not slow down or change the rank of single-KB recall.
- Bridge `kind`s are an enum the kernel owns. A KbPlugin cannot invent new
  bridge kinds; this keeps `engram bridges show` semantics stable.

## Alternatives considered

- **Live cross-KB edges.** Tightest freshness; rejected per ADR-0004
  consistency.
- **Fuse bridges into the scoring formula (RRF-style).** Cleanest end-user
  story — bridge-connected memories rank higher automatically. Rejected:
  SPEC §3.6 explicitly forbids RRF, and the bridge `score` is not directly
  comparable to `(I × Relevance × Recency) × m_v`.
- **Bridge generation inside the dreaming worker.** Coupling is wrong — a
  worker run on KB-A shouldn't have to rebuild bridges to KB-Z. Keep it as
  its own scheduled job kind.
