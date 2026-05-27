# engram — Architecture Invariants (project rule)

The authoritative design is `docs/engram-SPEC.md` (v2.2; v1.2 multi-KB + skill subsystem amendment lives in §15 — see ADR-0005…0008). This rule states the
load-bearing invariants that must hold in every change. Violating one is a
review BLOCK, not a WARN.

## The four boundaries (microkernel)

engram is a **fixed core kernel** + **four plugin seams**. Code goes in exactly
one place:

- **FIXED CORE** (`src/core/`) — Markdown Store, Scoring Engine, GitVersioning,
  AppLog, AccessControl, Dreaming Orchestrator, CaptureIntake, Plugin Host,
  CoreService facade. This is engram's identity; it is NOT swappable. Security-
  critical logic (privacy filter, access control, safe/gated classification)
  lives here, never in a plugin.
- **PLUGIN SEAMS** (`src/plugins/{retrieval,graph,llm,capture}/`) — one v1
  implementation each, behind the `PluginManifest`/`PluginLifecycle` contract.
  Two transports only: in-process (TS) or subprocess (JSON-RPC/stdio). A new
  engine drops in behind the same interface — no FFI, no core change.

## Non-negotiable invariants

1. **Files are truth.** Markdown files in `memories/` are authoritative. QMD
   index, graphify graph, AppLog, and git are all **derived and rebuildable**.
   `index`/`update`/`deindex`/`ingestEdges` failures are non-fatal: log, enqueue
   retry, the write still succeeds. `rebuild()` always recovers correctness.
2. **The graph is a derived index, not a mutable DB** (Spike 3). graphify is
   `extract → graph.json → query`. `ingestEdges`/`removeNode` are batch
   rebuild/update operations, never live edge mutation.
3. **Scoring never imports the retrieval plugin.** QMD supplies *relevance only*;
   the core scoring engine fuses `(I × Relevance × Recency) × m_v` and owns the
   formula. Graph is opt-in expansion, not a competing rank (no RRF).
4. **The dreaming/ingest worker is a detached process.** All LLM work happens in
   the worker; the orchestrator owns no LLM. A worker crash/overrun NEVER touches
   the core daemon or the agent's session.
5. **No silent mutation, no silent capture.** Every memory change emits an
   AppLog record; every capture hook invocation is logged (including
   suppressions). Recall NEVER fails hard — it degrades (`partial`/`degraded`).
6. **Forgetting = lifecycle transition, never deletion.** `active → dormant →
   archived`. Hard-delete exists only as audited `governance_delete`. Respect the
   active-pool floor and per-run rate limits.
7. **Worker output is schema-validated** (`src/schemas/dream-output.schema.json`).
   Memory bodies are untrusted data; injected frontmatter fields are rejected at
   both worker and orchestrator stages. Validation failure → job FAILED.
8. **Capture is fire-and-forget; PreCompact is request/response.** Capture hooks
   post and exit 0 within budget. `PreCompact` is the one exception — it returns
   recall context to the host.

## Bottom-up build discipline

Build core-first, layer by layer, with a verification gate per phase (see the
implementation plan). Earliest working loop (remember → recall) is reachable at
the core phase via the grep fallback, before any plugin. Don't build a later
layer to make an earlier one "work."
