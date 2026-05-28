# WP12: Cron Setup

**Severity**: MEDIUM
**Status**: Phase 2b -- PLANNING
**Depends on**: WP0 (framework repo), WP3 (memory-init), WP4 (daily-update), WP5 (memory-lint), WP8 (memory-digest)

## Problem

The llm-memory system requires autonomous scheduling for maintenance operations. The obsidian-wiki pattern uses Claude Code's CronCreate mechanism (session-based, no system crontab) for daily/weekly/monthly jobs. Three CronCreate jobs must be defined: daily vault refresh, weekly lint+consolidation, and monthly digest generation. Additionally, a SessionStart hook must detect staleness (last daily update >25h ago) and prompt the user.

## Target Files

- `~/.claude/hooks/hooks.json` -- register SessionStart staleness hook
- `~/.claude/scripts/hooks/session-start-memory.cjs` -- add staleness check
- `<framework-repo>/config/.env.example` -- add cron schedule documentation

## Implementation Steps

### Step 1: Create CronCreate daily update
`57 9 * * *` /daily-update -- refresh hot.md, update index.md if needed. Title: "Daily Memory Update".

### Step 2: Create CronCreate weekly lint
`7 10 * * 6` /memory-lint --consolidate -- full 13-point lint scan with auto-fix. Title: "Weekly Memory Lint + Consolidation".

### Step 3: Create CronCreate monthly digest
`17 9 1 * *` /memory-digest monthly save -- generate monthly summary of themes, connections, open threads. Title: "Monthly Memory Digest".

### Step 4: SessionStart staleness detection
Add check in session-start-memory.cjs: read _meta/.last-daily-update timestamp. If >25h stale, prompt user (accept/dismiss, no auto-run). Write timestamp on successful completion.

### Step 5: Verification script
verify-cron.sh: CronList shows 3 jobs with correct schedules; staleness triggers when >25h stale.

### Step 6: Cron management
Document schedules in config/.env.example. CronCreate is idempotent. Provide /memory-cron-status command.

### Relationship to WP-14's autonomous daemons
WP-12's three CronCreate jobs run **inside an interactive Claude session** —
they are the user-facing scheduled maintenance. WP-14's daemon scripts
(`ingest-agent.sh`, `curate-agent.sh`) run **autonomously, outside any session**,
as the unattended backstop. The two **coexist** and do not conflict:
- Where both touch weekly lint, the WP-12 CronCreate job runs `memory-lint
  --consolidate` (interactive, applies fixes with confirmation); the WP-14
  curation daemon runs `memory-lint` report-only (`auto_consolidate: false` by
  default) — the daemon never writes what the interactive job applies.
- WP-12 jobs are registered by `memory-init` (WP-3 Phase 9); WP-14 daemon timers
  are installed by `memory-init` (WP-3 Phase 10). Both are idempotent.
This split is also documented in WP-14 Step 4.

## Recommended Agents

- `skill-creator` -- validate skill triggers for daily-update, memory-lint, memory-digest
- `plan-reviewer` -- verify schedule alignment with WP3 CronCreate setup in memory-init

## Verification

See VERIFICATION.md WP12 section: V12.1 (3 CronCreate jobs registered), V12.2 (staleness detection triggers). Script: `verify-cron.sh`.
