# ADR 0001 — Microkernel architecture with a detached dreaming worker

- **Status:** Accepted
- **Date:** 2026-05-22
- **Deciders:** cunger, Claude

## Context

engram needs a standalone application that stores agent memory, retrieves it,
and runs an LLM-powered *dreaming* process to consolidate it. Three forces
shape the architecture:

1. Several boundaries genuinely have **multiple implementations** — the LLM
   provider (Claude / OpenAI / Ollama), the capture adapter (Claude Code now,
   Codex / Copilot later), and the retrieval and graph engines (QMD, graphify
   today — with QMD at v0.9, an unproven version).
2. The design notes are explicit that dreaming must be a **decoupled,
   independent process** that does **not affect the main agent** — a heavy LLM
   dreaming run must never stall a latency-sensitive `memory.recall`.
3. The prior plan failed partly by **locking an archived dependency** (Kuzu).
   The architecture must keep swappable parts swappable.

## Decision

Adopt a **microkernel** with a **detached dreaming worker process**.

- **Fixed core** (one always-up daemon process — not swappable, the system's
  identity): Markdown store, scoring engine, versioning, access control,
  dreaming orchestrator, plugin host, MCP + dashboard API.
- **Four plugin seams** (each a contract with one v1 implementation):
  retrieval (QMD), graph (graphify), capture (Claude Code hooks), LLM /
  embedding (Claude / OpenAI / Ollama).
- **Dreaming runs as a separate OS process**, spawned per job, communicating
  with the core via the shared store and a job queue, writing results to a git
  branch.

## Alternatives considered

- **Monolithic daemon** — everything in one process. Rejected: collapses the
  dreaming/main-agent decoupling the design requires; a heavy dream stalls
  recalls.
- **Thin-kernel pure microkernel** — store, scoring, versioning themselves
  plugins. Rejected: over-abstraction; the system would have no fixed
  identity, every core concept indirected for no concrete second
  implementation.
- **In-process dreaming plugin** — dreaming isolated by module boundary only.
  Rejected: shares the event loop with the latency-sensitive MCP server.

## Consequences

- **Positive:** dreaming cannot stall or crash the core; swappable engines
  guard against the Kuzu mistake; the LLM-provider seam delivers the
  Claude/OpenAI/Ollama support required on day one; clean test seams.
- **Negative:** one IPC boundary (job queue + shared store) to design and
  test; a per-job process spawn has startup cost (acceptable — dreaming is not
  latency-sensitive).
- **Follow-up:** OQ-A — decide in-process vs out-of-process plugin loading
  (leaning: in-process for retrieval/LLM, subprocess for the Python graphify
  plugin).

## References

- `docs/spec/SPEC.md` §2 (Architecture)
- Graph-store review handoff, 2026-05-21 (the Kuzu-archived finding)
