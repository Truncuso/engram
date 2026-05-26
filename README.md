# engram

A standalone, principled, self-organizing **memory for AI coding agents** — with
a decoupled *dreaming* process that consolidates and learns from it.

> Status: **pre-implementation.** Design is complete (`docs/engram-SPEC.md` v2.1)
> and de-risked (`docs/research/spikes-2026-05-26.md`). Bottom-up build starts
> from the core kernel.

## What it is

engram is a background application — an always-up daemon (`engramd`) + MCP server
+ CLI + a detached worker — that:

- stores agent memory as **Markdown files** (the source of truth), typed as
  `semantic` / `episodic` / `procedural` / `contextual`;
- retrieves it with **hybrid scored search** — `(Importance × Relevance ×
  Recency) × confidence`;
- runs a decoupled **dreaming** process that distills captured sessions into
  typed memories, connects them, re-weights importance, and learns from failures;
- **forgets by lifecycle transition** (`active → dormant → archived`), never by
  silent deletion.

It is **not** a wiki, not Obsidian-specific, and not an agent config — it is a
product you install. See `docs/engram-SPEC.md` for the full design.

## Architecture (one line)

A **fixed microkernel** (store, scoring, access control, orchestrator, capture
intake, plugin host) with **four plugin seams** — retrieval (QMD), graph
(graphify), LLM (Claude/OpenAI/Ollama), capture (Claude Code) — and a **detached
worker** for all LLM-heavy dreaming/ingest. A worker crash never touches the core.

## Stack

TypeScript / Node ≥22 (ESM, strict) · `@tobilu/qmd` (in-process retrieval) ·
`@modelcontextprotocol/sdk` (Streamable HTTP, bearer-gated) · Vercel AI SDK
(`ai`) behind `LlmPlugin` · graphify (`graphifyy`, Python subprocess) ·
`better-sqlite3` (app-log / jobs / stats). One Python seam (the graphify adapter);
everything else is TypeScript. Rationale: `docs/adr/`.

## Layout

```
src/
  core/        CoreService, Store, Scoring, AppLog, AccessControl, OCC, Orchestrator, Plugin Host
  schemas/     Zod schemas + JSON-schema exports (frontmatter, manifest, dream-output)
  plugins/     retrieval (QMD) · graph (graphify + py/) · llm · capture
  mcp/         Streamable HTTP MCP server (16 verbs, bearer auth)
  capture/     Claude Code capture hooks + CaptureIntake wiring
  worker/      detached dreaming / ingest worker
  cli/         the `engram` CLI
docs/          engram-SPEC.md · adr/ · review/ · research/
tests/         unit · integration · e2e (the SPEC §12.3 success criteria)
```

## Development

```sh
nvm use            # Node 22
npm install
npm run typecheck
npm test
```

Build bottom-up, one phase at a time, each behind a verification gate. The
earliest working remember→recall loop is reachable at the core phase (grep
fallback) before any plugin is wired. See the implementation plan and `AGENTS.md`.

## Security note

An engram store holds memory content and may hold secrets. The store directory
(`~/.engram/` or `<repo>/.engram/`) is git-ignored and **must not be pushed to a
public remote**. This repository contains only engram's *source*, never a store.

## License

MIT.
