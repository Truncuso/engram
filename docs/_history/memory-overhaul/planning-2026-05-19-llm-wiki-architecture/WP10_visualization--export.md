# WP10: Visualization + Export

**Severity**: MEDIUM
**Status**: Phase 2b -- PLANNING
**Depends on**: WP0 (framework repo), WP2 (llm-memory), WP4 (ingest skills)

## Problem

The obsidian-wiki repo provides 5 visualization/export skills that must be adapted to the llm-memory framework: `graph-colorize` (rewrite `.obsidian/graph.json` colorGroups, 5 modes, always backs up), `memory-export` (JSON/GraphML/Neo4j/HTML export), `memory-dashboard` (Obsidian Bases `.base` files + Dataview queries), `memory-bridge` (browse/search/diff/map by tool origin), and `memory-context-pack` (token-bounded context snapshots). All require config path migration and vault convention alignment.

## Target Files

All under `<framework-repo>/.skills/`: `graph-colorize/`, `memory-export/`, `memory-dashboard/`, `memory-bridge/`, `memory-context-pack/`.

## Implementation Steps

### Step 1: Port `graph-colorize`
- Copy from obsidian-wiki; replace `$OBSIDIAN_VAULT_PATH` with `$LLM_MEMORY_VAULT_PATH`
- Retain all 5 modes: by-tag (top-10 tags), by-category (7 vault folders), by-visibility (pii/internal/public), combined, custom
- Keep 10-color colorblind-friendly palette, backup-before-write pattern, Obsidian-open-while-editing warnings
- **Keep the `_raw/` exclusion** — `_raw/` is a source-staging area in our vault;
  raw sources must not appear as colored knowledge nodes in the Obsidian graph

### Step 2: Port `memory-export`
- Adapt config and visibility filter; replace vault path references
- Retain: node/edge extraction, typed-edge enrichment from `relationships:` blocks, community detection, all 4 output formats
- Output to `memory-export/` at vault root; maintain: re-run safe, broken wikilink skipping, gitignore recommendation

### Step 3: Port `memory-dashboard`
- Adapt config; retain both tools: Obsidian Bases (canonical `.base` YAML schema) + Dataview (SQL-like queries)
- Keep filter syntax rules, property name conventions, groupBy handling
- Target `_meta/` for dashboard files

### Step 4: Port `memory-bridge`
- Adapt config and source_type mapping (same types in our system)
- Retain all 4 modes: browse, search, diff, map; keep tool names (deferred agents listed as unavailable)

### Step 5: Port `memory-context-pack`
- Adapt config, replace `hot.md`, `index.md` references to use `$LLM_MEMORY_VAULT_PATH`
- Retain: relevance scoring algorithm, tier-aware selection, compression/dedup, budget enforcement
- Keep invocation forms: `/memory-context-pack "<topic>" --budget N`, `--recent` mode
- Maintain 4 chars/token heuristic, max-budget warning

### Step 6: Create verification scripts
One verify script per skill — all 5 skills must be covered:
- `scripts/verify/verify-graph-colorize.sh`: by-tag mode produces valid colorGroups; backup exists
- `scripts/verify/verify-export.sh`: all 4 formats produced; JSON/GraphML parse correctly
- `scripts/verify/verify-dashboard.sh`: produces a `.base` file that parses as valid Obsidian Bases YAML
- `scripts/verify/verify-memory-bridge.sh`: browse mode lists all pages with a given `source_type`
- `scripts/verify/verify-context-pack.sh`: a snapshot for a known topic stays within the `--budget` token cap

## Recommended Agents

- `skill-creator` -- validate each skill's frontmatter and trigger accuracy
- `code-reviewer` -- review config path substitutions for completeness

## Verification

See VERIFICATION.md WP10 section — one check per skill:
- V10.1 / V10.2: `graph-colorize` — valid colorGroups, backup created
- V10.3 / V10.4: `memory-export` — all 4 formats, JSON/GraphML parse
- V10.5: `memory-dashboard` — emits a `.base` file parseable by Obsidian Bases
- V10.6: `memory-bridge` — browse mode lists all pages of a given `source_type`
- V10.7: `memory-context-pack` — snapshot ≤ the specified token budget

Scripts: `verify-graph-colorize.sh`, `verify-export.sh`, `verify-dashboard.sh`,
`verify-memory-bridge.sh`, `verify-context-pack.sh`.
