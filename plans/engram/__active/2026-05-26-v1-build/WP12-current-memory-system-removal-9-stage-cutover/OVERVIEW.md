---
name: wp12-current-memory-system-removal-9-stage-cutover
title: Current memory-system removal (9-stage cutover)
type: work-package
stage: spec
severity: MEDIUM
created: 2026-05-26
updated: 2026-05-26
plan: 2026-05-26-v1-build
tags: [cutover, migration, removal, dotfiles]
relationships:
  - depends_on: [[wp05-mcp-server-coreservice-facade-16-verbs-bearer]]
  - depends_on: [[wp08-dreaming-worker-orchestrator]]
sources: [SRC-01]
phases: [phase-1-build-and-coexist, phase-2-migrate-and-cutover, phase-3-teardown]
---
<!-- Template: WP-folder OVERVIEW v2 (frontmatter-first) -->

# WP12: Current memory-system removal (9-stage cutover)

> Folder work package. Phases group the 9 stages. **Invariant: at every step,
> either the old system OR engram serves memory — never neither.**

## Problem

engram replaces an existing Claude Code memory system (5 skills, 3 hooks, a
CLAUDE.md override block, data under `~/.claude/.memory/` symlinked to
`dotfiles/claude/.memory/`, QMD MCP). This WP removes it **without ever leaving
the user without memory**, migrating real content first, and preserving the
non-memory value of `grill-with-memory`. It runs as a **parallel cutover track**,
gated on engram milestones — not a phase in the build chain.

### Inventory to act on (gathered; do not re-explore)
- **Skills (5):** `memory-write`, `memory-curate`, `memory-init`,
  `memory-onboard`, `grill-with-memory` (⚠ last one ALSO does plan-grilling /
  ADR / glossary — decouple, don't delete).
- **Hooks (3 .cjs)** in `~/.claude/scripts/hooks/`: `session-start-memory.cjs`,
  `session-end-memory.cjs`, `qmd-refresh.cjs` (⚠ multiple skill callers —
  `capture-learning`, `handoff` — remove last).
- **CLAUDE.md** auto override block `<!-- memory-init:override:start -->…end`.
- **Data:** `dotfiles/claude/.memory/` (3 project facts, 2 daily logs, indexes);
  legacy `~/.claude/projects/*/memory/` (NOT git-tracked).
- **QMD MCP** in `settings.json` — **STAYS**; only re-point collections.

## Target Files (external to engram repo — the user's `~/.claude` + dotfiles)

- `~/.claude/hooks/hooks.json` (SessionStart/SessionEnd entries)
- `~/.claude/scripts/hooks/{session-start,session-end,qmd-refresh}-memory.cjs`
- `~/.claude/skills/{memory-write,memory-curate,memory-init,memory-onboard,grill-with-memory}/`
- `~/.claude/CLAUDE.md` (override block)
- `dotfiles/claude/.memory/**` (migrate → engram, then remove)

## Phases (the 9 stages)

| Phase | Stages | Goal | Status |
|-------|--------|------|--------|
| [1](phase-1-build-and-coexist.md) | 1–2 | engram built to WP05; WP06 capture hooks installed; coexist with old system | pending |
| [2](phase-2-migrate-and-cutover.md) | 3–6 | migrate content (git-commit backup) → re-point QMD → swap SessionStart hook → skills cutover (grill rewire) | pending |
| [3](phase-3-teardown.md) | 7–9 | remove data (after 14d no-access) → remove .cjs scripts → strip CLAUDE.md block | pending |

## Verification

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| W12-1 | at no stage is memory unavailable | new session always gets memory context | manual per-stage |
| W12-2 | content migrated, recallable in engram | 3 facts + 2 daily logs retrievable via engram recall | e2e |
| W12-3 | grill-with-memory still grills/ADRs after rewire | non-memory paths unchanged; memory write → engram remember | integration |
| W12-4 | QMD binary + MCP intact; only collections re-pointed | `qmd` still works; old collection removed | manual |
| W12-5 | rollback works | `git checkout <pre-migration> -- .memory/` restores | manual |
| W12-6 | no orphaned callers break | `capture-learning`/`handoff` don't call removed qmd-refresh | grep gate |

## Recommended Agents

| Phase | Agent | Rationale |
|-------|-------|-----------|
| all | `code-reviewer` | surgical edits to live config |
| migrate | `debugger` | verify no broken-state between stages |
