# WP3: memory-init Overhaul (Fuse with memory-setup)

**Severity**: HIGH
**Status**: Pending
**Depends on**: WP1 (vault migration), WP2 (llm-memory skill)

## Problem

The current `memory-init` scaffolds a basic typed-file directory with hooks. The overhauled version must create a full memory vault (`~/memory/` or `<repo>/.memory/`), install hook scripts, register QMD collections, create CronCreate jobs, set up per-project skill symlinks for multi-agent access, install systemd timers for the upkeep daemon, and offer `memory-onboard`. It must be fully idempotent.

## Target Files

- `<framework-repo>/.skills/memory-init/SKILL.md` — complete rewrite
- `<framework-repo>/templates/` — index.md, log.md, hot.md, manifest.json, USER.md, GLOSSARY.md, PROJECT.md
- `<framework-repo>/scripts/hooks/` — session-start-memory.cjs, session-end-memory.cjs, qmd-refresh.cjs
- `~/.claude/CLAUDE.md` — update memory override block
- `~/.llm-memory/config` — create with vault path + framework repo path

## Implementation Steps

### Phase 1: Verify prerequisites
- QMD MCP reachable
- Detect scope: `--global` (creates `~/memory/`) or `--project` (creates `<repo>/.memory/` from CWD)

### Phase 2: Create vault structure
**Global**: `~/memory/` with concepts/, entities/, skills/, references/, synthesis/, journal/, projects/, _raw/, _meta/, _archive/, .obsidian/
**Project**: `<repo>/.memory/` with same structure + `_raw/plans/` → `<repo>/plans/`

### Phase 3: Create special files
- index.md, log.md, hot.md, .manifest.json (empty)
- USER.md, GLOSSARY.md (global only)
- PROJECT.md (project only)

### Phase 4: Symlinks
- `~/.claude/memory` → `~/memory/` (global mode)
- `~/memory/_raw/plans/` → `~/.claude/plans/` (global mode)
- `<repo>/.memory/_raw/plans/` → `<repo>/plans/` (project mode)

### Phase 5: Config files
- Write `~/.llm-memory/config`: `LLM_MEMORY_VAULT_PATH=~/memory`, `LLM_MEMORY_REPO=<framework-repo-path>`
- Project mode: write `<repo>/.env` with vault path

### Phase 6: CLAUDE.md override
- Global mode: update `~/.claude/CLAUDE.md` memory override block → canonical path `~/memory/`
- Project mode: add memory section to `<repo>/CLAUDE.md` or `AGENTS.md`
- Note: framework repo has its own `CLAUDE.md` (memory bootstrap with routing table). User's `~/.claude/CLAUDE.md` is a separate file with the memory override block. Both coexist.

### Phase 7: Install hook scripts with memory-aware load list
- Copy `session-start-memory.cjs`, `session-end-memory.cjs`, `qmd-refresh.cjs` from framework repo to `~/.claude/scripts/hooks/`
- Register in `~/.claude/hooks/hooks.json` if not already present
- **Updated session-start-memory.cjs load list** (replaces MEMORY.md pointer-index pattern):
  - **Global**: Load `~/memory/index.md` (page catalog with title + one-line summary per page) + `~/memory/hot.md` (~500-word semantic snapshot of recent activity). The old `MEMORY.md` (manual pointer-index) is superseded by auto-generated `index.md`; `USER.md` content is absorbed into `hot.md` for identity context.
  - **Project** (when CWD inside a git repo with `.memory/`): Load `<repo>/.memory/index.md` + `<repo>/.memory/hot.md`. Same `findProjectMemory()` CWD-walk logic preserved from current hook.
  - **Token budget**: ≤2,500 tokens combined, wrapped in `<memory-context>` framing. Project scope overrides global on conflict.
  - **Fallback**: If `hot.md` is missing or stale, load `USER.md` (global) or `PROJECT.md` (project) as one-page identity snapshots. If `index.md` is missing, skip (vault not yet initialized — no error).
  - **QMD staleness gate**: Check `_meta/.qmd-last-refresh` timestamp before loading. If >10 min stale, trigger `qmd-refresh.cjs` before loading. If the systemd QMD daemon is running, skip (daemon refreshed recently).

### Phase 8: QMD registration
- Global: `qmd index --name memory-global ~/memory/`
- Project: `qmd index --name memory-<project> <repo>/.memory/`
- Write `_meta/qmd-collections.json` listing all registered collections

### Phase 9: CronCreate jobs
- Daily: `57 9 * * *` → `/daily-update`
- Weekly: `7 10 * * 6` → `/memory-lint --consolidate`
- Monthly: `17 9 1 * *` → `/memory-digest monthly save`

### Phase 10: Install systemd timers (Linux) or launchd plists (macOS)
- QMD index daemon: every 10 min
- Ingestion agent: daily
- Curation agent: weekly

### Phase 11: Per-project skill symlinks (project mode)
- Create `.claude/skills/*` → relative symlinks to `<framework-repo>/.skills/*` (all 35 skills)
- Create `.cursor/skills/*` → relative symlinks to `<framework-repo>/.skills/*`
- Create `.windsurf/skills/*` → relative symlinks
- Create `.agents/skills/*` → relative symlinks
- Create `.kiro/skills/*` → relative symlinks
- This ensures ALL memory skills are available when working inside a project with `.memory/`

### Phase 12: Offer memory-onboard
- After scaffold, suggest `/memory-onboard` to populate initial content

### Idempotency
Every phase checks if the target already exists before acting. Re-running produces no changes.

### Phase 13: Fold in deferred Phase 1 code-review findings (G4)

WP-3 rewrites the hook scripts and the verification script, so it is the right
place to clear the two latent items Phase 1's code-review deferred (the third —
the `startsWith` path-boundary check — is handled in WP-1 Step 4):

- `verify-memory.cjs` (→ becomes `verify-memory-init.cjs`): the gitignore checks
  call `ignored()` twice per check, spawning `git check-ignore` 8× instead of 4×.
  Compute each result once and reuse.
- `qmdAvailable()` is duplicated byte-for-byte between `qmd-refresh.cjs` and the
  verification script. Extract a shared `.cjs` helper in `scripts/hooks/` (or
  `scripts/verify/`) and require it from both.

## Verification

See VERIFICATION.md WP3 section: 7 tests covering global/project scaffold, idempotency, hooks, token budget, cron, and per-project skill symlinks.
