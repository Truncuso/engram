# WP-16: Memory Agent — Autonomous Memory Worker

**Severity**: MEDIUM
**Status**: PLANNING (Phase 2b)
**Depends on**: WP2 (llm-memory schema), WP3 (memory-init hooks), WP4 (ingest/query + Kuzu), WP5 (cross-linker + lint), WP7 (memory-curate)
**Augments**: WP14 (autonomous upkeep daemons) with LLM judgment layer

## Problem

WP14's daemons handle deterministic work (QMD indexing, file watching) but cannot
make judgment calls: "is this page stale but high-value?", "should I merge these
near-duplicates or flag them?", "which cross-linking strategy worked best last time?"

The main agent burns context on memory operations that a specialized worker could
handle in parallel. A dedicated memory agent that:

- Knows the full vault structure, all memory skills, Kuzu, QMD, and verify scripts
- Runs in parallel without consuming the main agent's context
- Makes LLM-powered decisions about what to clean, link, synthesize, or archive
- Verifies its own work (runs verify scripts, checks pass/fail)
- Learns from results and improves its own strategies over time
- Reports structured results back to the main agent

## Design

### Agent Definition

**Location**: `~/.claude/agents/memory.md` (canonical); also stored in
`<framework-repo>/agents/memory.md` for distribution.

**Type**: Custom Claude Code subagent (`subagent_type: "memory"`), launchable via Agent tool.

**Core instructions:**

```
# Memory Agent

You maintain the llm-memory vault. Your job: retrieve knowledge, maintain
the vault, and improve the memory system itself.

## Skills
- memory-query — search (Kuzu graph + QMD semantic)
- memory-ingest — add new sources
- memory-write — save facts with full frontmatter
- memory-lint — audit structural issues
- cross-linker — insert missing wikilinks
- memory-curate — dedup, rebuild, stage
- memory-synthesize — discover co-occurrence patterns
- memory-digest — generate period summaries

## Scripts
- sync-graph.cjs — build/refresh Kuzu graph
- qmd-refresh.cjs — refresh QMD index
- verify-*.sh — deterministic pass/fail tests

## Vault Structure
Global: ~/memory/  |  Project: <repo>/.memory/
Categories: concepts/ entities/ skills/ references/ synthesis/ journal/ projects/
Special: index.md hot.md log.md .manifest.json
Kuzu DB: <vault>/.kuzu/ (derived, rebuildable)
QMD collections: memory-global, memory-<project>

## Work Protocol
1. Check _meta/agent-reports/ for pending tasks
2. Execute task using appropriate skill
3. Run corresponding verify-*.sh script
4. Record result to _meta/agent-reports/YYYY-MM-DD/THHMMSS-task.md
5. If verify fails: diagnose, retry (max 2), escalate if still failing
6. Report summary to main agent (1-2 lines)

## Report Format (~/memory/_meta/agent-reports/YYYY-MM-DD/THHMMSS-task.md)
---
task: <what was attempted>
skill: <skill used>
verify: pass|fail|partial
findings: N
fixed: N
needs_review: N
errors: <any>
---
<detailed findings>

## Self-Improvement
Weekly: review past 7 days of reports.
- Identify success/failure patterns
- Update this agent definition with improved heuristics
- Flag consistently underperforming skills for user review
- Track: tasks attempted, pass rate, avg duration, common failure modes
```

### Triggers

| Trigger | When | What | Budget |
|---------|------|------|--------|
| **SessionEnd hook** | After each session | Quick: `cross-linker` (no --deep), `memory-lint` report-only, `sync-graph.cjs --force` | 30s async |
| **CronCreate** | Daily 9:57 / Weekly Sat 10:07 / Monthly 1st 9:17 | Daily: `daily-update` + staleness. Weekly: `memory-lint --consolidate` + `cross-linker --deep`. Monthly: `memory-curate DEDUP` + `memory-synthesize` + `memory-digest monthly` | Sonnet weekly/monthly, Haiku daily |
| **On-demand** | User: "memory agent, do X" | Interprets request, runs skill(s), verifies, reports | Matches parent |

### Integration with WP14

```
WP14 daemons (always on, no LLM)     Memory Agent (triggered, LLM-powered)
  │                                     │
  ├─ qmd-index-daemon (10 min)          ├─ "stale but central — keep"
  ├─ ingest-agent (10 min)              ├─ "near-duplicates — merge"
  ├─ curate-agent (weekly)              ├─ "cross-linker failed 3× — investigate"
  │                                     ├─ "contradiction found — resolve"
  │                                     └─ "verify-graph fails — rebuild Kuzu"
```

### Communication with Main Agent

**Short** (~50 tokens inline): "Memory agent: 3 cross-links, 0 lints, graph synced. 1 page flagged."

**Full** (read on demand): `~/memory/_meta/agent-reports/YYYY-MM-DD/THHMMSS-task.md`

At SessionStart, the most recent agent report summary is included in `hot.md`.

## Target Files

- `<framework-repo>/agents/memory.md` — canonical agent definition
- `~/.claude/agents/memory.md` — installed by memory-init WP3
- `~/memory/_meta/agent-reports/` — report directory (created by memory-init)
- `<framework-repo>/scripts/verify/verify-memory-agent.sh` — E2E verification

### Integration with `context: fork` skills

The Memory Agent serves dual roles:

1. **Autonomous scheduler**: invoked by SessionEnd hooks and CronCreate to dispatch
   housekeeping tasks.
2. **Agent type for forked skills**: 10 memory skills use `context: fork` +
   `agent: memory` in their SKILL.md frontmatter (see `SKILL_YAML_REVIEW.md` §1.2).
   This means forked subagents inherit vault-structure knowledge, tool access, and
   the skill catalog from `~/.claude/agents/memory.md` — they don't start cold.

**Forked execution model:**

```
Main Agent or CronCreate
  │
  ├─ Lightweight skills (in-process):
  │   memory-query, memory-status, daily-update, memory-write
  │
  └─ Heavy skills (forked via context: fork + agent: memory):
      memory-ingest (batch), memory-lint, cross-linker --deep,
      memory-curate (DEDUP/REBUILD), memory-synthesize, memory-digest,
      memory-research, claude-history-ingest, memory-export, memory-dashboard
```

The Memory Agent definition (`~/.claude/agents/memory.md`) must include:
- Vault structure reference (both global and project)
- Skill catalog summary (which skill for which task)
- Work protocol (execute → verify → report)
- `allowed-tools` covering all forked skill tool needs

## Implementation Steps

### Step 1: Agent Definition

1. Write `memory.md` with Work Protocol, report format, and self-improvement loop.
2. Install via memory-init WP3: symlink from framework repo to `~/.claude/agents/`.
3. Test: launch via Agent tool with `subagent_type: "memory"`.

### Step 2: SessionEnd Integration

1. Extend `session-end-memory.cjs` (WP3) to optionally spawn the memory agent.
2. Config flag: `MEMORY_AGENT_AUTO=true|false` in `~/.llm-memory/config`.
3. When enabled: after QMD refresh, spawn agent for post-session housekeeping.
4. Agent runs async (30s timeout) — main session does not wait.

### Step 3: CronCreate Jobs

Add to WP12 CronCreate setup:
- Daily 9:57: `daily-update` + `memory-lint` report
- Weekly Sat 10:07: `memory-lint --consolidate` + `cross-linker --deep` + `verify-graph.cjs`
- Monthly 1st 9:17: `memory-curate DEDUP` + `memory-synthesize` + `memory-digest monthly`
- Weekly Sun 8:07: self-improvement review

### Step 4: Verification

`verify-memory-agent.sh`:
- Fixture vault with known issues (broken link, orphan, unlinked mention)
- Launch agent: "run housekeeping on this vault"
- Assert: broken link fixed, orphan cross-linked, unlinked mention detected
- Assert: report written to `_meta/agent-reports/` with correct frontmatter
- Assert: `verify-cross-link.sh` passes after agent run

## Verification

See VERIFICATION.md WP-16 section:
- V16.1: Agent definition valid and loadable
- V16.2: SessionEnd spawns agent when `MEMORY_AGENT_AUTO=true`
- V16.3: Agent runs housekeeping on fixture vault, fixes known issues
- V16.4: Agent writes structured report with correct frontmatter
- V16.5: Agent self-improvement loop runs without errors
- V16.6: `verify-graph.cjs` passes after agent graph sync
- V16.7: `verify-cross-link.sh` passes after agent cross-link run
