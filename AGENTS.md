# engram — Agent Catalog & Orchestration

Catalog/orchestration only. **Behavioral policy lives in `CLAUDE.md`** (and the
global `~/.claude/CLAUDE.md`). This file is also the canonical entry point for
non-Claude agents (Codex, Gemini CLI, etc.) working in this repo.

## Orientation

1. Read [`docs/engram-SPEC.md`](docs/engram-SPEC.md) (v2.1) — the authoritative
   design. §2 architecture, §13 phases.
2. Read [`docs/adr/`](docs/adr/) — why the key decisions were made.
3. Read the project rules in [`.claude/rules/`](.claude/rules/): architecture
   invariants + TS conventions.
4. The implementation plan (once written) sequences the bottom-up build.

## Recommended agents (global, reuse)

| Task | Agent |
|------|-------|
| Architecture / seams / scalability decisions | `architect`, `architect-reviewer` |
| TypeScript implementation | `typescript-pro` |
| TypeScript / async / security review | `typescript-reviewer` |
| General code review (every change) | `code-reviewer` |
| Security-sensitive code (capture filter, access control, MCP auth) | `security-reviewer` |
| SQLite schema / query review | `database-reviewer` |
| Bugs / failing tests | `debugger`, `diagnose` |
| Build / type errors | `build-error-resolver` |
| Docs & ADRs | `documentation-and-adrs` skill |
| Commit hygiene | `commit-work` skill |

## Module map (single package, `src/` by module)

| Path | Module | Build phase |
|------|--------|-------------|
| `src/core/` | CoreService, Store, Scoring, AppLog, AccessControl, OCC, Orchestrator, Plugin Host | 1, 8a/8c |
| `src/schemas/` | Zod schemas + JSON-schema exports (frontmatter, manifest, dream-output) | 1 |
| `src/plugins/llm/` | LlmPlugin (Vercel AI SDK; Claude/OpenAI/Ollama) | 2 |
| `src/plugins/retrieval/` | QMD RetrievalPlugin (in-process) | 3 |
| `src/mcp/` | Streamable HTTP MCP server, bearer auth, 16 verbs + status resource | 5 |
| `src/capture/` | Claude Code capture hooks + CaptureIntake wiring | 6 |
| `src/plugins/capture/` | CapturePlugin (install/normalise) | 6 |
| `src/plugins/graph/` (+ `py/`) | graphify GraphPlugin (subprocess) | 7 |
| `src/worker/` | detached dreaming/ingest worker | 8b |
| `src/cli/` | `engram` CLI | across phases |

## Verification

Each phase has a gate (tests in `tests/{unit,integration,e2e}/`). The 18 success
criteria (SPEC §12.3) are the final E2E suite. Build order and gates: see the
implementation plan.
