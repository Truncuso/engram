# WP9: Agent History Ingest

**Severity**: MEDIUM
**Status**: Phase 2b -- PLANNING
**Depends on**: WP0 (framework repo), WP2 (llm-memory), WP3 (memory-init — creates the
vault config + `.manifest.json` + QMD collections these skills read), WP4 (core ingest skills)

## Problem

The obsidian-wiki repo provides a unified history ingest pipeline: a thin router (`memory-history-ingest`) dispatching to per-agent handlers, a comprehensive Claude miner (`claude-history-ingest`) covering both CLI and desktop app sessions, and a query-driven cross-agent search tool (`memory-history-search`). These three skills must be adapted to the llm-memory framework, replacing obsidian-wiki paths and config conventions. The 4 other agent history variants (codex, hermes, openclaw, copilot) are explicitly deferred -- no data sources exist on this machine.

## Target Files

All under `<framework-repo>/.skills/`:

- `memory-history-ingest/SKILL.md` -- unified router dispatching to per-agent handlers
- `claude-history-ingest/SKILL.md` -- mine `~/.claude/projects/*/*.jsonl` + desktop `local-agent-mode-sessions/`
- `memory-history-search/SKILL.md` -- query-driven targeted ingest from specific agent history

## Implementation Steps

### Step 1: Port `memory-history-ingest` (router)
- Copy from obsidian-wiki, replace config references: `~/.obsidian-wiki/config` -> `~/.llm-memory/config`
- Drop codex, copilot, hermes, openclaw entries from routing table (deferred)
- Keep `auto` mode: infer from path heuristics
- Update invocation forms to `/memory-history-ingest claude` and `$memory-history-ingest claude`

### Step 2: Port `claude-history-ingest`
- Adapt config resolution: `OBSIDIAN_VAULT_PATH` -> `LLM_MEMORY_VAULT_PATH`; `CLAUDE_HISTORY_PATH` default stays `~/.claude`
- Keep: append mode (delta from `.manifest.json`), full mode, all 4 data sources (memory files, JSONL conversations, audit logs, session metadata)
- Replace `$OBSIDIAN_VAULT_PATH/...` with `$LLM_MEMORY_VAULT_PATH/...` throughout
- Replace the `_archives/` exclusion pattern with `_archive/` (our vault uses
  `_archive/`, singular). **Keep the `_raw/` exclusion** — `_raw/` is a live
  source-staging directory in our vault and must remain excluded from page scans.
- Update QMD references: drop the single-collection `QMD_WIKI_COLLECTION` env-var
  lookup; use the multi-collection model from AF-4 — read `_meta/qmd-collections.json`
  and iterate over registered collections (`memory-global`, `memory-<project>`).
- Maintain: step-by-step pipeline (survey delta, ingest memory first, parse JSONL, cluster by topic, distill into pages, update manifest/journal/hot.md)

### Step 3: Port `memory-history-search`
- Adapt config resolution and path conventions as in Step 2
- Drop codex, copilot, hermes, openclaw routing (deferred)
- Keep: Claude session inventory, scoring against query, extraction + distillation, synthesized answer format
- Retain cross-agent use patterns section (document future expansion)
- Update invocation forms: `/memory-claude [query]` remains

### Step 4: Create verification script
- `scripts/verify/verify-history-ingest.sh`: test router dispatch to claude handler, verify pages created with correct category + sources frontmatter

## Recommended Agents

- `skill-creator` -- validate each adapted skill's frontmatter and trigger accuracy
- `plan-reviewer` -- verify alignment with WP0/WP2 conventions

## Verification

See VERIFICATION.md WP9 section: V9.1 (router dispatch), V9.2 (extract session knowledge). Script: `scripts/verify/verify-history-ingest.sh`.
