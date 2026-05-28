# Verification Matrix — Memory System Overhaul: LLM-Memory Architecture

**Date**: 2026-05-19

---

## Script-First Principle

Every work package ships with a deterministic verification script that produces pass/fail. No manual "looks good" verification. Scripts are committed alongside skills they test and runnable via `make verify`.

---

## Verification Script Inventory

| Script | Tests | Used By |
|--------|-------|---------|
| `scripts/verify/verify-symlinks.sh` | All symlinks resolve, vault structure complete, no nested .obsidian/ | WP0, WP1 |
| `scripts/verify/verify-qmd.sh` | Per-vault collections registered, multi-collection search returns hits | WP1 |
| `scripts/verify/verify-skill-frontmatter.sh` | YAML frontmatter valid, required fields present | WP2 through WP11, WP13 |
| `scripts/verify/verify-memory-init.cjs` | Idempotent scaffold, hook registration, token budget ≤ 2,500 | WP3 |
| `scripts/verify/verify-ingest.sh` | Known source → page with correct frontmatter, manifest updated, idempotent re-ingest | WP4, WP6 |
| `scripts/verify/verify-query.sh` | Known fact retrievable via tiered pipeline | WP4 |
| `scripts/verify/verify-daily-update.sh` | hot.md refreshed, content ≤ 500 words | WP4 |
| `scripts/verify/verify-lint.sh` | Test vault with known issues → all 13 types detected | WP5 |
| `scripts/verify/verify-status.sh` | Delta computed, token footprint, insights mode | WP5 |
| `scripts/verify/verify-cross-link.sh` | Unlinked mentions detected and linked | WP5 |
| `scripts/verify/verify-graph.cjs` | Kuzu sync, 5 Cypher query patterns, centrality | WP4 |
| `scripts/verify/verify-capture.sh` | CAPTURE mode: conversation → memory note | WP6 |
| `scripts/verify/verify-update.sh` | UPDATE mode: git-delta → project overview updated | WP6 |
| `scripts/verify/verify-rebuild.sh` | REBUILD: archive → rebuild → restore | WP7 |
| `scripts/verify/verify-dedup.sh` | DEDUP: near-duplicate merged, wikilinks rewritten | WP7 |
| `scripts/verify/verify-stage.sh` | STAGE: promote from _staging/, manifest updated | WP7 |
| `scripts/verify/verify-synthesize.sh` | Co-occurrence detected, synthesis page created | WP8 |
| `scripts/verify/verify-digest.sh` | Correct output for daily/weekly/monthly periods | WP8 |
| `scripts/verify/verify-history-ingest.sh` | Agent history → pages with correct category + sources | WP9 |
| `scripts/verify/verify-graph-colorize.sh` | graph.json colorGroups updated, backup created | WP10 |
| `scripts/verify/verify-export.sh` | Valid JSON/GraphML/HTML export | WP10 |
| `scripts/verify/verify-repo-governance.sh` | Stale wikilinks in .memory/ flagged | WP11 |
| `scripts/verify/verify-cron.sh` | CronList shows 3 jobs, staleness detection triggers | WP12 |
| `scripts/verify/verify-setup.sh` | setup.sh idempotent, second run clean | WP13 |
| `scripts/verify/verify-skill-yaml.sh` | All 24 skills: description+when_to_use split, disable-model-invocation on destructive, paths on vault-scoped, allowed-tools on operational, argument-hint on parameterized, ${CLAUDE_SKILL_DIR} usage, agent:memory on forked | WP13 |
| `scripts/verify/verify-daemon.sh` | systemd timers active, scripts executable, config valid, ingestion E2E, daemon+hook interaction | WP14 |
| `scripts/verify/verify-e2e.sh` | Full pipeline: ingest → query → lint → cross-link → synthesize → digest | WP15 |
| `scripts/verify/verify-memory-agent.sh` | Agent definition, SessionEnd spawn, housekeeping, self-improvement | WP16 |

---

## Improvement Loop Pattern

Each verification script follows:

```
1. BASELINE: measure before change (lint score, coverage %, token count)
2. CHANGE: apply the skill/workflow change
3. VERIFY: run verification script → pass/fail
4. COMPARE: diff baseline vs new measurement → regression or improvement?
5. REPORT: write results to _meta/improvement-log.json
```

---

## WP0: Framework Repo Scaffold

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| V0.1 | `git status` in framework repo | Clean working tree after scaffold | `verify-symlinks.sh` |
| V0.2 | `~/.claude/skills/memory-init` resolves | Symlink → framework repo `.skills/memory-init/` | `ls -l ~/.claude/skills/memory-init` |
| V0.3 | All existing 6 skills symlinked | 6 symlinks in `~/.claude/skills/` point to framework repo | `verify-symlinks.sh` |

## WP1: Directory Migration + Vault Setup

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| V1.1 | `~/memory/` exists with all category dirs | 10+ directories created | `verify-symlinks.sh` |
| V1.2 | `~/.claude/memory` → `~/memory/` symlink | `readlink ~/.claude/memory` = `~/memory/` | `verify-symlinks.sh` |
| V1.3 | `_raw/plans/` → `~/.claude/plans/` symlink | Plans accessible from vault | `verify-symlinks.sh` |
| V1.4 | QMD collection registered | `qmd search --name memory-global "test"` returns without error | `verify-qmd.sh` |
| V1.5 | `~/.llm-memory/config` exists | Contains `LLM_MEMORY_VAULT_PATH=~/memory` | `grep` |
| V1.6 | CLAUDE.md override references `~/memory/` | No reference to `~/.claude/.memory/` as canonical | `grep ~/.claude/CLAUDE.md` |
| V1.7 | Pre-flight: verify-memory.cjs 37/37 before migration | All 37 tests pass | `verify-memory.cjs` |
| V1.8 | Backup exists before migration | `~/memory-backup-<date>/` exists with complete content | `test -d` |
| V1.9 | Post-migration: verify-memory.cjs 37/37 re-run | All 37 tests pass after migration | `verify-memory.cjs` |
| V1.10 | Rollback rehearsal: restore from backup → verify | Restored paths match pre-migration state | `verify-memory.cjs` |

## WP2: llm-memory Core Architecture Skill

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| V2.1 | SKILL.md frontmatter valid | name, description fields present and non-empty | `verify-skill-frontmatter.sh` |
| V2.2 | Skill triggers on `/llm-memory` | Agent routes to this skill | `verify-skill-frontmatter.sh` |
| V2.3 | References `~/memory/` and `<repo>/.memory/` | Path conventions documented in skill body | `grep` |

## WP3: memory-init Overhaul

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| V3.1 | `--global` creates all dirs + special files | index.md, log.md, hot.md, .manifest.json, USER.md, GLOSSARY.md exist | `verify-memory-init.cjs` |
| V3.2 | `--global` is idempotent on re-run | Second run makes no changes | `verify-memory-init.cjs` |
| V3.3 | `--project` creates `<repo>/.memory/` with `.obsidian/` | Full project vault structure | `verify-memory-init.cjs` |
| V3.4 | `--project` is idempotent | Second run makes no changes | `verify-memory-init.cjs` |
| V3.5 | Hooks registered in hooks.json | SessionStart + SessionEnd entries exist | `verify-memory-init.cjs` |
| V3.6 | Token budget ≤ 2,500 | `session-start-memory.cjs` output under cap | `verify-memory-init.cjs` |
| V3.7 | CronCreate jobs created | 3 jobs visible in CronList | `verify-cron.sh` |

## WP4: Core Ingest + Query Skills

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| V4.1 | memory-ingest: source → page with full frontmatter | All required + recommended fields present | `verify-ingest.sh` |
| V4.2 | memory-ingest: .manifest.json updated | Source hash + timestamp recorded | `verify-ingest.sh` |
| V4.3 | memory-ingest: >=2 wikilinks in new page | Cross-references inserted | `verify-ingest.sh` |
| V4.4 | memory-ingest: re-ingest same source (idempotency) | No duplicate pages, manifest unchanged | `verify-ingest.sh` |
| V4.5 | memory-query: known fact retrievable | QMD search returns hit with correct content | `verify-query.sh` |
| V4.6 | memory-query: tiered pipeline works | index → semantic → grep → read escalation | `verify-query.sh` |
| V4.7 | daily-update: hot.md refreshed | hot.md timestamp updates, content ≤ 500 words | `verify-daily-update.sh` |
| V4.8 | Kuzu sync: fixture vault → node + edge counts match | Kuzu DB reflects all pages and wikilinks | `verify-graph.cjs` |
| V4.9 | Kuzu multi-hop: returns correct path | Seeded multi-hop chain traversed correctly | `verify-graph.cjs` |
| V4.10 | Kuzu reverse-lookup: all inbound pages | Pages linking to target returned with rel_type | `verify-graph.cjs` |
| V4.11 | Kuzu neighborhood: depth-limited results | Pages within specified hop distance returned | `verify-graph.cjs` |
| V4.12 | Kuzu path existence: true/false correctly | Connected paths detected, disconnected returns false | `verify-graph.cjs` |
| V4.13 | Kuzu bridge detection: pages connecting categories | Pages with edges to both categories returned | `verify-graph.cjs` |

## WP5: Housekeeping Skills

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| V5.1 | memory-lint: detects orphans | Orphaned page flagged | `verify-lint.sh` |
| V5.2 | memory-lint: detects broken wikilinks | Missing target flagged | `verify-lint.sh` |
| V5.3 | memory-lint: detects missing frontmatter | Pages without required fields flagged | `verify-lint.sh` |
| V5.4 | memory-lint: detects stale content | Pages untouched > 90 days flagged | `verify-lint.sh` |
| V5.5 | memory-lint: detects contradictions | Conflicting claims between pages flagged | `verify-lint.sh` |
| V5.6 | memory-lint: detects index inconsistency | index.md missing pages flagged | `verify-lint.sh` |
| V5.7 | memory-lint: detects provenance drift | Provenance marker issues flagged | `verify-lint.sh` |
| V5.8 | memory-lint: detects fragmented tag clusters | Low-cohesion tag groups flagged | `verify-lint.sh` |
| V5.9 | memory-lint: detects misc promotion candidates | misc/ pages with high affinity flagged | `verify-lint.sh` |
| V5.10 | memory-lint: detects synthesis gaps | Co-occurring concepts without synthesis page flagged | `verify-lint.sh` |
| V5.11 | memory-lint: detects confidence/lifecycle issues | Invalid lifecycle states flagged | `verify-lint.sh` |
| V5.12 | memory-lint: detects typed relationship errors | Invalid relationship types flagged | `verify-lint.sh` |
| V5.13 | memory-lint: detects visibility tag issues | Visibility tag inconsistencies flagged | `verify-lint.sh` |
| V5.14 | memory-lint: --consolidate auto-fixes | Broken links fixed, orphans cross-linked | `verify-lint.sh` |
| V5.15 | memory-status: delta computed | Shows ingested vs pending, token footprint | `verify-status.sh` |
| V5.16 | cross-linker: unlinked mentions detected and linked | [[wikilink]] inserted at first occurrence | `verify-cross-link.sh` |
| V5.17 | cross-linker `--deep`: emergent entities detected | Entities flagged with ≥3 occurrences, no single-page false positives | `verify-cross-link.sh` |
| V5.18 | cross-linker without `--deep`: entity pass skipped | No LLM call, no entity suggestions in output | `verify-cross-link.sh` |

## WP6: memory-write Overhaul

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| V6.1 | Write fact → full frontmatter | All memory fields populated | `verify-ingest.sh` |
| V6.2 | CAPTURE mode: conversation → memory note | Synthesis page with correct content type | `verify-capture.sh` |
| V6.3 | UPDATE mode: git-delta sync | Project overview page updated | `verify-update.sh` |

## WP7: memory-curate Overhaul

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| V7.1 | REBUILD: archive → rebuild → restore | Vault restored from sources, all pages intact | `verify-rebuild.sh` |
| V7.2 | DEDUP: near-duplicate pages merged | Redirect stub at secondary path, wikilinks rewritten | `verify-dedup.sh` |
| V7.3 | STAGE: review + promote from _staging/ | Staged page moved, manifest updated | `verify-stage.sh` |

## WP8: Synthesis + Digest + Tag + Research + Ingest

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| V8.1 | memory-synthesize: co-occurrence detected | Synthesis page created with Concept A x Concept B | `verify-synthesize.sh` |
| V8.2 | memory-digest daily: correct output | Themes, connections, open threads present | `verify-digest.sh` |
| V8.3 | memory-digest weekly: correct output | Weekly themes, notable connections | `verify-digest.sh` |
| V8.4 | memory-digest monthly: correct output | Monthly trends, long-running threads | `verify-digest.sh` |
| V8.5 | tag-taxonomy audit detects off-taxonomy tags | Unknown tags on a fixture vault flagged | `verify-tag-taxonomy.sh` |
| V8.6 | ingest-url: fetched URL becomes a page | Correct slug + confidence bucket in frontmatter | `verify-ingest-url.sh` |
| V8.7 | data-ingest: fixture CSV becomes a page | Correct category + frontmatter | `verify-data-ingest.sh` |
| V8.8 | memory-research: known topic researched | Reference + concept + synthesis pages created | `verify-research.sh` |

## WP9: Agent History Ingest

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| V9.1 | memory-history-ingest: routes to correct agent handler | claude-history-ingest invoked for Claude sessions | `verify-history-ingest.sh` |
| V9.2 | claude-history-ingest: extracts session knowledge | Pages created with correct category + sources | `verify-history-ingest.sh` |
| V9.3 | memory-history-search: cross-agent query | Returns results from indexed agent histories | `verify-history-ingest.sh` |

## WP10: Visualization + Export

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| V10.1 | graph-colorize: by-category mode | graph.json colorGroups updated, backup created | `verify-graph-colorize.sh` |
| V10.2 | memory-export: JSON export | Valid JSON with nodes + edges | `verify-export.sh` |
| V10.3 | memory-export: GraphML export | Valid GraphML parseable by Gephi | `verify-export.sh` |
| V10.4 | memory-export: HTML export | vis.js interactive graph loads in browser | `verify-export.sh` |
| V10.5 | memory-dashboard: emits a `.base` file | Parses as valid Obsidian Bases YAML | `verify-dashboard.sh` |
| V10.6 | memory-bridge: browse by source_type | Lists all pages of a given source_type | `verify-memory-bridge.sh` |
| V10.7 | memory-context-pack: budget respected | Snapshot ≤ the specified token budget | `verify-context-pack.sh` |

## WP11: Existing Skill Edits

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| V11.1 | capture-learning routes to correct scope | Global or project path based on CWD | `verify-skill-frontmatter.sh` |
| V11.2 | handoff writes to .memory/handoffs/ and daily/ | Both files created with correct content | `verify-skill-frontmatter.sh` |
| V11.3 | grill-with-memory writes with memory frontmatter | ADR/glossary pages have full frontmatter | `verify-skill-frontmatter.sh` |
| V11.4 | repo-governance scans for stale wikilinks | Broken [[links]] in .memory/ flagged | `verify-repo-governance.sh` |

## WP12: Cron Setup

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| V12.1 | CronList shows 3 jobs | daily-update, memory-lint, memory-digest registered | `verify-cron.sh` |
| V12.2 | SessionStart staleness detection | Prompt shown if daily update > 25h stale | `verify-cron.sh` |

## WP13: Framework Repo Finalization

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| V13.1 | setup.sh idempotent | Second run makes no changes to working tree | `verify-setup.sh` |
| V13.2 | All skills in `.skills/` have valid SKILL.md frontmatter (every skill goes through skill-creator review) | No missing or malformed frontmatter; skill-creator passes on all skills | `verify-skill-frontmatter.sh` |
| V13.2a | All skills have `description` + `when_to_use` split with >=3 trigger phrases each | Non-empty `when_to_use` on every skill | `verify-skill-yaml.sh` |
| V13.2b | Destructive skills have `disable-model-invocation: true` | memory-curate, memory-write, memory-init, memory-research, ingest-url | `verify-skill-yaml.sh` |
| V13.2c | Vault-scoped skills have `paths: "~/memory/**,**/.memory/**"` | 20 of 24 skills; portable 4 do NOT have paths | `verify-skill-yaml.sh` |
| V13.2d | Skills with `context: fork` have valid `agent: memory` | All forked skills use agent: memory (WP16) | `verify-skill-yaml.sh` |
| V13.2e | Operational skills have `allowed-tools` set | Non-empty, valid tool names matching skill body operations | `verify-skill-yaml.sh` |
| V13.2f | Parameterized skills have `argument-hint` + use `$ARGUMENTS` or `$N` in body | Argument declaration matches body usage | `verify-skill-yaml.sh` |
| V13.2g | Skills with bundled scripts use `${CLAUDE_SKILL_DIR}` | No hardcoded paths to skill directory | `verify-skill-yaml.sh` |
| V13.3 | README.md documents architecture + setup | All required sections present | `grep` |
| V13.4 | AGENTS.md routing table covers every shipped skill | No skill missing from the routing table | `verify-skill-frontmatter.sh` |
| V13.5 | Secrets-hygiene scan: no real tokens/config in committed files | Zero findings (G3 — public repo) | `verify-setup.sh` + `grep` |
| V13.6 | CI workflow file is valid | GitHub Actions YAML parses; runs `make verify` | YAML lint |
| V13.7 | v1.0.0 git tag exists and points at the release commit | `git tag --list` shows `v1.0.0` | `git tag --list` |

## WP14: Autonomous Upkeep Agent

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| V14.1 | systemd timer active | `systemctl --user is-active qmd-index-daemon.timer` = active | `verify-daemon.sh` |
| V14.2 | ingest script executable and dry-run succeeds | `test -x scripts/daemons/ingest-agent.sh`, dry-run exits 0 | `verify-daemon.sh` |
| V14.3 | config.yaml valid | YAML parses without error, all required fields present | `verify-daemon.sh` |
| V14.4 | QMD index refreshes within 10 min | File mtime updated; timestamp in `_meta/.qmd-last-refresh` | `verify-daemon.sh` |
| V14.5 | Ingestion processes new _raw/ content | New file → page created + manifest updated within 15 min | `verify-daemon.sh` |
| V14.6 | Curation dry-run produces expected output | Lint findings + synthesis candidates listed without mutation | `verify-daemon.sh` |
| V14.7 | Daemon + SessionStart hook interaction | Hook skips refresh if daemon refreshed < 10 min ago | `verify-daemon.sh` |

## WP15: E2E Verification

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| V15.1 | Full pipeline: ingest → query → lint → cross-link → synthesize → digest | All stages pass | `verify-e2e.sh` |
| V15.2 | `make verify` all green | Exit code 0 | `make verify` |
| V15.3 | skill-stocktake Quick Scan | No broken triggers or missing references | skill-stocktake |

## WP16: Memory Agent

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| V16.1 | Agent definition valid and loadable | `~/.claude/agents/memory.md` parses without error | `verify-memory-agent.sh` |
| V16.2 | SessionEnd spawns agent when `MEMORY_AGENT_AUTO=true` | Agent launched with housekeeping prompt | `verify-memory-agent.sh` |
| V16.3 | Agent runs housekeeping on fixture vault | Known issues fixed (broken links, orphans, unlinked mentions) | `verify-memory-agent.sh` |
| V16.4 | Agent writes structured report | `_meta/agent-reports/YYYY-MM-DD/THHMMSS-task.md` with correct frontmatter | `verify-memory-agent.sh` |
| V16.5 | Agent self-improvement loop runs | Weekly review processes past 7 days of reports without error | `verify-memory-agent.sh` |
| V16.6 | verify-graph.cjs passes after agent sync | Kuzu DB reflects all pages after agent graph sync | `verify-memory-agent.sh` |
| V16.7 | verify-cross-link.sh passes after agent run | All unlinked mentions resolved after agent cross-link | `verify-memory-agent.sh` |

---

## Live / CLI Verification

| ID | Command / Action | Expected Observation |
|----|------------------|---------------------|
| L1 | `memory-init --global` on clean install | All infrastructure created, idempotent |
| L2 | `memory-ingest` with known source | Page written, manifest updated, QMD indexed |
| L3 | `memory-query "known topic"` | Correct answer with citations |
| L4 | `memory-lint` on fresh vault | No errors |
| L5 | `cross-linker` after ingest | New page linked from existing pages |
| L6 | `memory-synthesize` | Co-occurrence opportunities listed |
| L7 | `memory-digest daily` | Digest generated and saved |
| L8 | Open `~/memory/` in Obsidian | Graph view shows all pages with color-coded categories |
| L9 | Open `<repo>/.memory/` in Obsidian | Per-project graph view with independent colorGroups |
| L10 | Wait 10 min after new _raw/ content | Content ingested automatically by daemon |

---

## Verification Status

| WP | Verification Scripts | Live/CLI | Review | Overall |
|----|---------------------|----------|--------|---------|
| WP0 | TODO | TODO | TODO | TODO |
| WP1 | TODO | TODO | TODO | TODO |
| WP2 | TODO | TODO | TODO | TODO |
| WP3 | TODO | TODO | TODO | TODO |
| WP4 | TODO | TODO | TODO | TODO |
| WP5 | TODO | TODO | TODO | TODO |
| WP6 | TODO | TODO | TODO | TODO |
| WP7 | TODO | TODO | TODO | TODO |
| WP8 | TODO | TODO | TODO | TODO |
| WP9 | TODO | TODO | TODO | TODO |
| WP10 | TODO | TODO | TODO | TODO |
| WP11 | TODO | TODO | TODO | TODO |
| WP12 | TODO | TODO | TODO | TODO |
| WP13 | TODO | TODO | TODO | TODO |
| WP14 | TODO | TODO | TODO | TODO |
| WP15 | TODO | TODO | TODO | TODO |
| WP16 | TODO | TODO | TODO | TODO |
