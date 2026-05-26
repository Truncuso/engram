# engram — TypeScript / Node Conventions (project rule)

Extends the global `coding-style` rule. Only engram-specific specifics here.

## Module system & runtime
- **ESM only.** `"type": "module"`; `NodeNext` resolution. Use `import`/`export`,
  never `require`. Relative imports carry the `.js` extension (NodeNext rule).
- **Node ≥22.** Target ES2023. No transpile-only polyfills.
- `verbatimModuleSyntax` is on — use `import type { … }` for type-only imports.

## Type safety (strict is non-negotiable)
- `strict`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`,
  `noImplicitOverride` are ON. Do not loosen `tsconfig` to make code compile —
  fix the code.
- No `any`. Use `unknown` at boundaries + a Zod parse. All external data
  (MCP params, LLM output, file frontmatter, plugin payloads) is validated with
  **Zod** at the boundary before it enters core types.
- Frontmatter and manifest shapes have a single source of truth in
  `src/schemas/` (Zod schemas + derived TS types + JSON-schema export for the
  worker). Never hand-redeclare these shapes elsewhere.

## Structure & dependencies
- Many small files (200–400 LOC typical, 800 max). One module = one clear
  purpose behind a thin interface (deep modules, per global coding-style).
- The plugin seam is the ONLY extension point — no speculative abstractions
  elsewhere (Simplicity-First).
- Pin direct deps; commit the lockfile. Prefer the SDKs already chosen
  (`@tobilu/qmd`, `@modelcontextprotocol/sdk`, `ai`, `@anthropic-ai/sdk`,
  `better-sqlite3`, `ulid`, `yaml`, `zod`) over hand-rolled equivalents.
- SQLite via `better-sqlite3` (synchronous, fits the daemon's single-writer
  model). All file writes are atomic: `.tmp` → `fsync` → `rename`.

## Errors & immutability
- Handle errors explicitly at boundaries; never swallow. Plugin errors use the
  `PluginError` kinds (`unavailable`/`invalid-input`/`transient`/`fatal`).
- Prefer immutable updates (return new objects); never mutate shared state.

## Testing
- **Vitest.** Co-locate unit tests in `tests/unit/`, integration in
  `tests/integration/`, success-criteria E2E in `tests/e2e/`.
- Each phase ships with its verification gate as automated tests before it is
  considered done (TDD per global `testing` rule; ≥80% coverage target).
- Phase 8c (orchestrator merge/classification) and 8a (job state machine) are
  testable with hand-written manifests / killed workers — **zero LLM calls**.

## Python seam (graphify adapter only)
- `src/plugins/graph/py/` is the ONLY Python. It adapts graphify's CLI + MCP
  stdio server to the `GraphPlugin` contract. PEP 8, 4-space indent, type hints.
  It is invoked as a subprocess — it shares no types with the TS core (the
  boundary is the JSON-RPC/stdio wire format).
