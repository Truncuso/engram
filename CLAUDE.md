# engram — Project Policy (Normative)

Project-scoped policy for the engram repo. **Global policy in `~/.claude/CLAUDE.md`
applies in full and takes precedence on conflict** unless overridden here. This
file adds only what is engram-specific. `AGENTS.md` is catalog/orchestration only.

## What engram is

A standalone, installable application giving AI coding agents a **persistent,
principled, self-organizing memory** plus a decoupled **dreaming** process that
consolidates and learns from it. TS/Node daemon (`engramd`) + MCP server +
detached worker; Markdown files = source of truth; QMD retrieval; graphify graph;
scored recall. **The authoritative design is [`docs/engram-SPEC.md`](docs/engram-SPEC.md)
(v2.1)** — read it before non-trivial work. Decisions are recorded as ADRs in
[`docs/adr/`](docs/adr/).

## Reused global rules (load on demand)

These global rules govern this repo — do not duplicate them here:

@~/.claude/rules/common/coding-style.md
@~/.claude/rules/common/code-review.md
@~/.claude/rules/common/testing.md
@~/.claude/rules/common/security.md
@~/.claude/rules/common/git-workflow.md
@~/.claude/rules/common/development-workflow.md

## engram-specific rules

@.claude/rules/engram-architecture.md
@.claude/rules/engram-typescript.md

## Stack (locked — see ADR-0002, ADR-0003)

- **Language:** TypeScript / Node ≥22, ESM, `strict`. One Python seam: the
  graphify adapter (`src/plugins/graph/py/`).
- **Retrieval:** `@tobilu/qmd` (in-process library — verified). **LLM substrate:**
  Vercel AI SDK (`ai`) behind `LlmPlugin`, raw `@anthropic-ai/sdk` for
  `cache_control`. **MCP:** `@modelcontextprotocol/sdk` Streamable HTTP on
  `127.0.0.1`, bearer-gated. **Graph:** graphify (`graphifyy`) subprocess.
  **State:** SQLite (`better-sqlite3`) for app-log / jobs / stats.

## Working agreements

- **Bottom-up, verify each phase.** Core first, then plugins, then MCP, then
  capture, then ingest, then dreaming (split 8a→8d). A phase is done only when
  its verification gate passes as automated tests. Earliest remember→recall loop
  works at the core phase (grep fallback) before any plugin.
- **The store is never committed.** `.engram/` and any memory store hold user
  memory and secrets — git-ignored, never pushed (the spec warns against public
  remotes for stores).
- **Spec is a living document.** When implementation forces a design decision,
  write an ADR and reflect it in `docs/engram-SPEC.md`; do not let code and spec
  drift.
- **Reuse before writing.** Prefer the chosen SDKs and existing global
  skills/agents (`typescript-pro`, `code-reviewer`, `documentation-and-adrs`)
  over new code.
