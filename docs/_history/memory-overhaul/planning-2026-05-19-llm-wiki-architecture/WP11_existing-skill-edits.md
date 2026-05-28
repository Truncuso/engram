# WP11: Existing Skill Edits

**Severity**: MEDIUM
**Status**: Phase 2b -- PLANNING
**Depends on**: WP0 (framework repo), WP3 (memory-init overhaul), WP4 (ingest skills)

## Problem

Five existing skills in `~/.claude/skills/` must be updated for the new memory architecture. `capture-learning` and `handoff` route writes to the legacy `~/.claude/.memory/` + `tmp/handoffs/` paths; both need to use `~/memory/` or `<repo>/.memory/` with memory frontmatter. `grill-with-memory` writes memory files with scripts needing memory-compatible frontmatter. `setup-sdd-repo` delegates to `/memory-init --project` but lacks verification. `repo-governance` scans `.memory/` for dangling links but skips wikilinks in page bodies.

## Target Files

Post-WP0, the framework repo is the **canonical source** for all skills;
`~/.claude/skills/` entries are symlinks into it. All edits MUST be made in the
framework repo, not through the symlink.

- `<framework-repo>/.skills/capture-learning/SKILL.md`
- `<framework-repo>/.skills/handoff/SKILL.md`
- `<framework-repo>/.skills/grill-with-memory/SKILL.md` (+ `scripts/write_memory_file.py`)
- `<framework-repo>/.skills/setup-sdd-repo/SKILL.md`
- `<framework-repo>/.skills/repo-governance/SKILL.md`

## Implementation Steps

### Step 0: Ensure all five skills are in the framework repo

WP0 Step 4 copied only six skills into `<framework-repo>/.skills/` — `memory-init`,
`memory-write`, `memory-curate`, `memory-onboard`, `grill-with-memory`,
`capture-learning`. **`handoff`, `setup-sdd-repo`, and `repo-governance` were NOT
copied.** Before editing, copy any of the five that are still absent from
`<framework-repo>/.skills/` (`cp -r ~/.claude/skills/<name> <framework-repo>/.skills/`),
then let WP3's symlink farm replace the `~/.claude/skills/` originals. Verify each
of the five resolves through `<framework-repo>/.skills/` before Step 1.

### Step 1: Update `capture-learning`
- Phase 4: replace typed-file template with memory frontmatter (add `tags:`, `confidence:`, `lifecycle:`, `summary:`)
- Phase 3: replace `~/.claude/.memory/` with `~/memory/` as global scope path
- Phase 5: add wikilink insertion step for cross-references
- Update `memory-init` references to `llm-memory` conventions

### Step 2: Update `handoff`
- Route writes to `<repo>/.memory/handoffs/YYYY-MM-DD/<slug>.md` (was `tmp/handoffs/`)
- Add memory frontmatter to mirror files: `tags: [handoff]`, `lifecycle: stable`
- Update QMD refresh reference to framework repo script path

### Step 3: Update `grill-with-memory`
- Upgrade `write_memory_file.py` frontmatter output to memory format (add `tags:`, `summary:`, `confidence:`, `lifecycle:`)
- Path updates: `~/.claude/.memory/` -> `~/memory/`; scope rule stays (project only)

### Step 4: Update `setup-sdd-repo`
- Section D: add verification step confirming `.memory/` was created by delegation
- Note that `memory-init` now creates full memory vault structure

### Step 5: Update `repo-governance`
- Add wikilink scan: parse `[[wikilinks]]` in page bodies, validate targets, flag as HIGH severity
- Add MEMORY.md wikilink table validation; keep existing pointer-index and typed-file checks

### Step 6: Create verification script
- `scripts/verify/verify-repo-governance.sh`: validate stale wikilink detection on test vault

## Recommended Agents

- `skill-creator` -- validate updated frontmatter formats
- `plan-reviewer` -- verify alignment with WP0/WP3 conventions
- `code-reviewer` -- review path substitutions for completeness

## Verification

See VERIFICATION.md WP11 section: V11.1 (capture-learning routing), V11.2 (handoff paths), V11.3 (grill-with-memory frontmatter), V11.4 (repo-governance wikilink scan). Script: `verify-repo-governance.sh`.
