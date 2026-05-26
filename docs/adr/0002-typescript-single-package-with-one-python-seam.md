# ADR-0002: TypeScript single package, with one Python seam (graphify)

- **Status:** Accepted
- **Date:** 2026-05-26
- **Related:** ADR-0003 (LLM substrate), SPEC §11.1, plan Part 1c

## Context

The SPEC defaulted to "all TypeScript" for velocity. The author has a second,
explicit goal: improve coding skill across languages, and wanted each module's
language chosen deliberately rather than by fiat. We examined every module
boundary for whether a second language adds learning value without becoming
bloat.

Findings: the **core daemon is effectively TS-locked** — the MCP SDK
(`@modelcontextprotocol/sdk`), QMD retrieval library (`@tobilu/qmd`, ESM), and
LLM substrate are all TS-first; a polyglot core would mean abandoning official
SDKs or building protocol bridges (pure cost). The genuine polyglot seam is the
**detached worker**, which is process-isolated (talks only via files + SQLite +
git) — but choosing TS there lets it share frontmatter/manifest types with the
core, eliminating cross-language type duplication. graphify is **already Python**
and consumed across a process boundary, so it is the one unavoidable non-TS seam.

## Decision

- **Single TypeScript package**, ESM, `strict`, Node ≥22. Layout: `src/` by
  module (`core`, `plugins/{retrieval,graph,llm,capture}`, `mcp`, `worker`,
  `cli`, `schemas`).
- **The dreaming/ingest worker is TypeScript**, sharing types with the core.
- **Exactly one Python seam:** the graphify adapter (`src/plugins/graph/py/`),
  consumed as a subprocess over a JSON-RPC/stdio wire boundary (no shared types).
- **Polyglot learning is deferred to a future agentic sub-project** (autonomous
  curation / dreaming-orchestration agent) built on the stable `LlmPlugin` seam,
  where a framework would earn its weight — not forced into the core now.

## Consequences

- Smallest repo, one primary toolchain, no TS↔X type duplication. Honors
  Simplicity-First and "non-bloating repo."
- The author's cross-language learning happens later, in a place where it is
  architecturally justified rather than incidental.
- graphify stays at arm's length; its Python is adapted, not authored.

## Alternatives considered

- **Rust worker** — real learning value, fast single binary; rejected for v1
  because it duplicates frontmatter/manifest types and an LLM client across the
  process boundary and slows the bottom-up build. Reconsider for a future
  perf-sensitive worker.
- **Python worker + LiteLLM** — closest to the LLM/agent ecosystem; rejected for
  v1 for the same type-duplication + second-runtime cost.
- **Monorepo workspaces** — cleaner module isolation; deferred as premature for a
  solo bottom-up build (can split later if needed).
