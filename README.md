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

## Prerequisites

> Pre-implementation: the commands below are engram's intended operator
> interface (SPEC §10), not yet shippable. Install/usage will work once the
> bottom-up build reaches the MCP milestone (M2).

- **Node ≥ 22** (ESM, `strict`).
- **Disk:** on first indexing, the retrieval engine (QMD) downloads a one-time
  **~1.28 GB GGUF embedding model** to `~/.cache/qmd/models`. BM25-only recall
  works without it; full hybrid recall needs it.
- **Ollama** — required for `memory.ingest` and the graph-extraction path
  (graphify uses an Ollama backend). Run `ollama serve` with a
  **structured-output-capable** model available. Dreaming/ingest are unavailable
  (daemon stays healthy, degraded) until a working model is reachable.

## Install

```sh
npm install -g engram        # (planned) global CLI + engramd daemon
engram init                  # scaffold a store: --global (~/.engram) or --project (<repo>/.engram)
engramd start                # start the always-up daemon (binds MCP on 127.0.0.1)
```

`engram init` mints a per-agent **bearer token** (stored `0600`, daemon-owned)
and prints the MCP endpoint + token to wire into your agent's MCP client config.
Point your MCP client at `http://127.0.0.1:<port>` with
`Authorization: Bearer <token>`.

## Quick-start (operator)

```sh
engram status                # daemon health + per-plugin (QMD / graphify / LLM) health
engram doctor                # integrity check; quarantines broken frontmatter, never crashes
engram doctor --fix          # repair what it safely can
```

Memory verbs (`memory.remember` / `recall` / `ingest` / `forget` / `history` /
`confirm` / `governance_delete`, plus `dream.*`) are issued by your agent over
MCP — see the 16-verb contract in `docs/engram-SPEC.md` §6.3. The full CLI
surface (`engram log`, `engram migrate`, `engram install`, `engram backup`,
`engram dream …`) is specified in SPEC §10.

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
  mcp/         Streamable HTTP MCP server (16 verbs + system/status resource, bearer auth)
  capture/     Claude Code capture hooks + CaptureIntake wiring
  worker/      detached dreaming / ingest worker
  cli/         the `engram` CLI
docs/          engram-SPEC.md · adr/ · review/ · research/
tests/         unit · integration · e2e (the SPEC §12.3 success criteria)
```

## Development (building from source)

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

## Acknowledgements / Inspirations

engram's design is informed by — and where called out, explicitly rejects ideas
from — a body of open-source agentic-memory work. The "studied / inspired by /
adopted from" list (with adopt-adapt-reject verdicts) is in
[`docs/research/agentic-memory-survey-2026-05-27.md`](docs/research/agentic-memory-survey-2026-05-27.md).
Short attribution:

- **[Pratiyush/llm-wiki](https://github.com/Pratiyush/llm-wiki)** (MIT) —
  session → static Markdown wiki with wikilinks; informs engram's `llm-wiki`
  and `obsidian-vault` KB types.
- **[nashsu/llm_wiki](https://github.com/nashsu/llm_wiki)** — the
  "compile knowledge once" LLM-wiki pattern (incremental page merge, LanceDB
  hybrid search, sigma.js graph viz, format-agnostic ingest); same family as
  Pratiyush/llm-wiki and a direct influence on engram's read-only dashboard
  graph view (ADR-0010) and multi-format ingest (ADR-0011/0012).
- **[rohitg00/agentmemory](https://github.com/rohitg00/agentmemory)**
  (Apache-2.0) — 4-tier memory + 12 capture hooks + 8 slash skills; informs
  engram's hook/skill surface and seed skill set.
- **[letta-ai/letta](https://github.com/letta-ai/letta)** (formerly MemGPT,
  Apache-2.0) — core/recall/archival hierarchy; informs the working/episodic
  framing while engram keeps episodic immutability.
- **[mem0ai/mem0](https://github.com/mem0ai/mem0)** (Apache-2.0) — two-phase
  extract-then-update consolidation; informs the dreaming-worker output
  contract.
- **[topoteretes/cognee](https://github.com/topoteretes/cognee)** (Apache-2.0)
  — multi-dataset scoping with isolation; informs the multi-KB registry +
  cross-KB bridge model.
- **[agno-agi/agno](https://github.com/agno-agi/agno)** (MPL-2.0) — multi-agent
  shared/team memory; cross-agent dreaming explicitly deferred to v2 (D-3).
- **[kingjulio8238/Memary](https://github.com/kingjulio8238/Memary)** (MIT) —
  entity-grounded recall over Neo4j; informs entity-match edges in cross-KB
  bridges.

Prior art also cited in SPEC v2.1: MemMachine, ElephantBroker, TierMem,
agentmemory's iii-engine backing.

## License

MIT.
