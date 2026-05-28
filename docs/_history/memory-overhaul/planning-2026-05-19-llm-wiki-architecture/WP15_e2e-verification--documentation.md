# WP-15: E2E Verification + Documentation

**Severity**: HIGH
**Status**: Phase 2b — PLANNING
**Depends on**: All preceding WPs (WP0 through WP14) — this is the final
integration, validation, and documentation phase.
Execution: Phase 6 in OVERVIEW.md — runs in parallel with WP14 where possible,
but full E2E flow (V15.1) requires WP14 daemons to be operational.

## Problem

At this point, all 15 work packages have been implemented individually. Each WP
had its own verification scripts and passed them. But no end-to-end integration
test has been run:

- Has the full pipeline (ingest a source, query it, lint it, cross-link it,
  synthesize themes, generate digest) been tested as a single flow?
- Does `make verify` pass — every verification script under `scripts/verify/` green?
- Has `skill-stocktake` been run on all ~24 skills to catch broken triggers or
  missing references?
- Does the Phase 1 → Phase 2 migration guide exist for users upgrading from the
  old `~/.claude/.memory/` system?
- Are all plan documents (OVERVIEW.md, TODO.md, VERIFICATION.md, OPEN_QUESTIONS.md,
  FINDINGS.md) internally consistent and finalized?

This WP is the **integration gate**. It ensures the system works as a whole, not
just as pieces.

## Target Files

- `<framework-repo>/scripts/verify/verify-e2e.sh` — new: full pipeline E2E test
  (or extend existing verify-memory.cjs with memory-specific test cases)
- `<framework-repo>/scripts/verify/verify-all.sh` — new: discovers and runs every
  `scripts/verify/verify-*.{sh,cjs}` and reports aggregate pass/fail (the runner
  used by `make verify` — no hard-coded script list)
- `<framework-repo>/docs/migration-guide.md` — new: Phase 1 → Phase 2 migration
  guide for users with existing `~/.claude/.memory/` setups
- `OVERVIEW.md` — final consistency pass (no stale references, statuses accurate)
- `TODO.md` — final status pass (all WPs reflect completed/production status)
- `VERIFICATION.md` — final check: all Vxx.y tests accounted for; all manual tests
  either converted to scripts or documented as intentional
- `OPEN_QUESTIONS.md` — final pass: OQ-8/9/11 status reflects reality
- `FINDINGS.md` — final pass: all architectural findings captured

## Implementation Steps

### Step 1: Extend verify-memory.cjs for full memory pipeline

The existing `verify-memory.cjs` from Phase 1 covers the typed-file memory store.
Extend it (or create a companion `verify-e2e.sh`) with memory-specific integration
tests:

```
verify-e2e.sh test plan:
1. SETUP: Create a clean test vault at /tmp/llm-memory-e2e-test/
   - Run memory-init --global (targeting the test vault)
   - Verify vault structure created (all category dirs + special files)
2. INGEST: Place a known test source in _raw/ and run memory-ingest
   - Assert: page created in correct category
   - Assert: full frontmatter with all required fields
   - Assert: >= 2 wikilinks in the page
   - Assert: .manifest.json updated with two hashes
3. QUERY: Run memory-query for a fact from the ingested page
   - Assert: tiered pipeline returns correct answer
   - Assert: answer cites the source page
4. HOUSEKEEPING: Run memory-lint, memory-status, cross-linker
   - Assert: memory-lint detects known issues in the test fixture
   - Assert: memory-status computes correct delta
   - Assert: cross-linker discovers and inserts at least 1 wikilink
5. SYNTHESIS: Run memory-synthesize
   - Assert: at least 1 co-occurrence candidate identified
6. DIGEST: Run memory-digest daily
   - Assert: digest generated and saved to correct path
   - Assert: content references existing pages
7. IDEMPOTENCY: Re-run steps 2–6
   - Assert: no duplicate pages, no redundant operations
8. DAEMON (optional — skip if WP14 not operational):
   - Place new file in _raw/, wait for timer tick (or fast-forward clock)
   - Assert: file auto-ingested by ingestion agent
9. CLEANUP: Remove /tmp/llm-memory-e2e-test/
```

The script exits 0 on all-pass, non-zero on first failure with a diagnostic
message. It is committed alongside the other verify scripts and run by
`make verify`.

### Step 2: Create verify-all.sh (aggregate runner)

Write `scripts/verify/verify-all.sh` — a simple script that runs every
verification script in `scripts/verify/` and reports a summary:

```bash
#!/bin/bash
# verify-all.sh — run all verification scripts and report aggregate pass/fail

PASS=0; FAIL=0; SKIP=0
for script in scripts/verify/*.sh scripts/verify/*.cjs; do
  [[ "$script" == "scripts/verify/verify-all.sh" ]] && continue
  echo "=== $script ==="
  if bash "$script" 2>/dev/null; then
    ((PASS++))
    echo "  PASS"
  else
    ((FAIL++))
    echo "  FAIL"
  fi
  echo ""
done
echo "Summary: $PASS passed, $FAIL failed, $SKIP skipped out of $((PASS+FAIL+SKIP))"
[[ $FAIL -eq 0 ]] && exit 0 || exit 1
```

Update `Makefile` (`make verify` target) to call `verify-all.sh` instead of
the current loop. This gives a clean aggregate report.

### Step 3: Write migration guide

Create `docs/migration-guide.md` for users with existing Phase 1 setups
(`~/.claude/.memory/` directory, hooks, QMD collection named `memory-default`):

```
docs/migration-guide.md structure:

1. PRE-MIGRATION CHECKLIST
   - Backup: tar czf ~/memory-backup-$(date +%Y%m%d).tar.gz ~/.claude/.memory/
   - Verify Phase 1 state: run existing verify-memory.cjs → must pass
   - Verify QMD is reachable: qmd search --name memory-default "test"

2. INSTALL LLM-MEMORY FRAMEWORK
   git clone https://github.com/cunger/llm-memory.git ~/...path...
   bash setup.sh

3. RUN memory-init --global
   - Creates ~/memory/ vault
   - Migrates Phase 1 content to memory vault categories:
     * ~/.claude/.memory/user/*.md → ~/memory/entities/
     * ~/.claude/.memory/feedback/*.md → ~/memory/references/
     * ~/.claude/.memory/reference/*.md → ~/memory/references/
     * ~/.claude/.memory/MEMORY.md → ~/memory/index.md
   - Creates ~/.claude/memory → ~/memory/ symlink (backward compat)
   - Re-registers QMD collection as memory-global

4. VERIFY MIGRATION
   - cd ~/memory && memory-status — should show migrated pages
   - memory-query "<known fact from Phase 1>" — should return result
   - memory-lint — should flag any frontmatter/tag incompatibilities
   - make verify (in framework repo) — all Phase 1+2 scripts pass

5. CLEANUP (optional)
   - Remove ~/.claude/.memory/ (only if symlink is working)
   - Remove old QMD collection: qmd index --remove memory-default

6. TROUBLESHOOTING
   - Symlink not working → check setup.sh output, re-run
   - QMD errors → qmd index --name memory-global ~/memory/
   - Missing pages → check migration mapping in memory-init Phase 5
   - Frontmatter errors → memory-lint --consolidate to auto-fix
```

### Step 4: Run skill-stocktake on all ~24 skills

Run the `skill-stocktake` skill in Quick Scan mode on the framework repo's
`.skills/` directory:

1. `cd <framework-repo> && /skill-stocktake --quick`
2. Verify: no broken triggers (skills that should fire but don't match expected
   trigger phrases).
3. Verify: no missing references (SKILL.md references a tool, script, or config
   file that doesn't exist).
4. Verify: all SKILL.md files have valid YAML frontmatter with `name` and
   `description` fields.
5. Fix any findings and re-run until clean.
6. Document the stocktake results in a brief note appended to `FINDINGS.md`.

### Step 5: Run full make verify

```
cd <framework-repo>
make verify
```

Assert: exit code 0. `make verify` runs every script under `scripts/verify/` via
`verify-all.sh` — that runner is the authority; the count grows with the skill
set, so it is **not** hard-coded here. The scripts, by WP:
- verify-symlinks.sh (WP0, WP1)
- verify-qmd.sh (WP1)
- verify-skill-frontmatter.sh (WP2 through WP11, WP13)
- verify-memory-init.cjs (WP3)
- verify-ingest.sh (WP4, WP6)
- verify-query.sh (WP4)
- verify-daily-update.sh (WP4)
- verify-lint.sh, verify-status.sh, verify-cross-link.sh (WP5)
- verify-capture.sh, verify-update.sh (WP6)
- verify-rebuild.sh, verify-dedup.sh, verify-stage.sh (WP7)
- verify-synthesize.sh, verify-digest.sh, verify-tag-taxonomy.sh,
  verify-ingest-url.sh, verify-data-ingest.sh, verify-research.sh (WP8)
- verify-history-ingest.sh (WP9)
- verify-graph-colorize.sh, verify-export.sh, verify-dashboard.sh,
  verify-memory-bridge.sh, verify-context-pack.sh (WP10)
- verify-repo-governance.sh (WP11)
- verify-cron.sh (WP12)
- verify-setup.sh (WP13)
- verify-daemon.sh (WP14)
- verify-e2e.sh (WP15)

If any script fails: isolate the root cause (code bug vs test bug), fix, re-run.
Log all failures and fixes to this WP's implementation notes.

### Step 6: Final documentation pass on all plan files

Read every plan file and check for:

1. **Stale references**: any mention of `obsidian-wiki` (the source repo is fine;
   stale is when a path/skill name/pattern should be `llm-memory` instead),
   `OBSIDIAN_VAULT_PATH` (should be `LLM_MEMORY_VAULT_PATH`), `~/.obsidian-wiki/`
   (should be `~/.llm-memory/`).
2. **Status accuracy**: all WP statuses reflect reality (completed → DONE, pending
   → PENDING, blocked → BLOCKED). OVERVIEW.md and TODO.md agree.
3. **Open questions**: confirm OPEN_QUESTIONS.md has no Active Questions — OQ-1
   through OQ-11 are all resolved (OQ-8/9/10/11 resolved 2026-05-19). Confirm
   WP14 reflects the OQ-8/9 resolutions (periodic scan, two cron jobs,
   queue-with-backoff, trust-API-estimate-local) and WP4 reflects OQ-10
   (append-mode default).
4. **Verification matrix**: VERIFICATION.md covers all 15 WPs. No Vxx.y test
   references a script that doesn't exist. All manual tests are either converted
   to scripts or documented as intentionally manual.
5. **Acceptance criteria**: OVERVIEW.md has checkboxes for all 16 ACs. Each AC that
   has been verified is marked `[x]`.
6. **Cross-references**: TODO.md implementation checklist items point to the correct
   WP files. OVERVIEW.md execution strategy phases match WP dependencies.
7. **WP-15 itself is the last incomplete item** — after this pass, the plan is
   fully specified and the system is production-ready.

### Step 7: Archive completed WPs

After all verification passes and documentation is finalized:

1. Mark all 15 WPs as complete in OVERVIEW.md and TODO.md.
2. Add a final entry to the Corrections Log if any last-minute fixes were made.
3. Commit all plan file updates with a summary of the E2E results.

## Recommended Agents

- `skill-stocktake` — Quick Scan on all ~24 skills
- `code-reviewer` — review verify-e2e.sh, verify-all.sh, migration-guide.md
- `plan-manager` — validate plan health after final documentation pass
- `superpowers:verification-before-completion` — final pre-completion checklist

## Verification

See VERIFICATION.md WP-15 section:
- V15.1: `verify-e2e.sh` — full pipeline passes: ingest, query, lint, cross-link,
  synthesize, digest. All stages green; idempotent re-run produces no changes.
- V15.2: `make verify` all green — exit code 0, every `scripts/verify/` script passes.
- V15.3: `skill-stocktake` Quick Scan — no broken triggers or missing references;
  all SKILL.md files valid.

Additional live checks (not automated):
- L15.4: Migration guide produces a working Phase 2 vault from a Phase 1 backup.
- L15.5: `~/memory/` opens in Obsidian with all pages visible in graph view.
- L15.6: `make install` in a fresh clone produces a working setup.sh topology for
  all 12 agent platforms (verified by verify-symlinks.sh).

## Complexity Delta

- **Added**: `verify-e2e.sh` (~150 lines of integration test), `verify-all.sh`
  (~30 lines of aggregate runner), `docs/migration-guide.md` (~100 lines of
  user documentation).
- **Removed**: None.
- **Justification**: E2E testing is the final correctness gate after 14 WPs of
  component-level work. Integration bugs (e.g., a query not finding a just-ingested
  page because the QMD refresh wasn't called) are invisible to unit tests. The
  migration guide reduces support burden for Phase 1 users. Both are minimal
  complexity adds for critical correctness and usability.
