# WP-D: Legacy migration from `~/.claude/projects/<slug>/memory/`

**Status**: TODO (lazy per-repo execution)
**Severity**: MEDIUM
**Created**: 2026-05-17
**Implemented**: —
**Depends on**: WP-C5
**Relevant Sources:** [SRC-04]

---

## Problem

The existing global per-project memory tree at `~/.claude/projects/<slug>/memory/` holds real content for two projects:
- `-home-cunger-10-Projects-01-fsl-cleaningapplication/` — 3 files, 82 lines (small, easy split)
- `-home-cunger-10-Projects-11-private-OSRS-glite-glite-private/` — 7,660-line `MEMORY.md` (counter-example for typed-file rationale)
- `-home-christoph-dotfiles/` — 0 files (empty, no migration needed)

Migration must:
1. Preserve originals (copy, don't move) until user confirms migration clean.
2. Heuristic-split legacy files into typed subdirs per the new schema.
3. Handle the 7,660-line MEMORY.md as a corpus, not file-by-file.

---

## Target Files

- `~/.claude/skills/memory-init/SKILL.md` (Phase 4 MIRROR & MIGRATE — implemented as part of WP-A, exercised here)
- `~/.claude/skills/memory-init/references/migration-heuristics.md` — heuristic rules
- (per repo, runtime) `<repo>/.memory/{user,feedback,project,reference}/*.md` — migrated files
- (per repo, runtime) `~/.claude/projects/<slug>/memory/` — preserved as read-only legacy

---

## Verified Evidence

- Survey finding: existing project memory locations enumerated above
- Plan source §8 — full migration strategy including 7,660-line special case
- `~/.claude/projects/-home-cunger-10-Projects-01-fsl-cleaningapplication/memory/` listing: `MEMORY.md` (62 lines), `feedback_*.md` (11 lines), `*_wp_*_complete.md` (9 lines)

---

## Implementation Steps

| Step | File | State |
|------|------|-------|
| 1. Author `references/migration-heuristics.md` with file-name → type-subdir rules | `memory-init/references/` | TODO |
| 2. Heuristic rules: `feedback_*.md` → `feedback/`; `*_wp_*_complete.md` → `project/`; `user_*.md` → global `user/`; legacy `MEMORY.md` → section-split | references/migration-heuristics.md | TODO |
| 3. Implement section-split logic for legacy multi-section `MEMORY.md` | SKILL.md Phase 4 | TODO |
| 4. Implement interactive prompt fallback for ambiguous files | SKILL.md Phase 4 | TODO |
| 5. Implement copy-not-move semantics (preserve originals) | SKILL.md Phase 4 | TODO |
| 6. Special-case: detect MEMORY.md > 1,000 lines → switch to corpus mode | SKILL.md Phase 4 | TODO |
| 7. Corpus mode: archive original at `_archive/legacy_MEMORY_<date>.md`; invoke `grill-with-docs` to extract top-20 facts | SKILL.md Phase 4 | TODO |
| 8. Test against fsl-cleaningapplication legacy memory | live | TODO |
| 9. Test against Glite 7,660-line MEMORY.md (corpus mode) | live | TODO |
| 10. Test against empty -home-christoph-dotfiles/ (no-op) | live | TODO |

---

## Verification

### Build Gates

| Gate | Command | Expected |
|------|---------|----------|
| Migration heuristics doc complete | `wc -l ~/.claude/skills/memory-init/references/migration-heuristics.md` | > 30 lines |

### Specific Tests

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| WP-D-T01 | Migrate fsl-cleaningapplication legacy (3 files) | All 3 files placed in correct typed subdirs; originals preserved | `ls <repo>/.memory/`, `ls ~/.claude/projects/<slug>/memory/` |
| WP-D-T02 | Legacy MEMORY.md with `## Build commands` section | Section split into `project/build_commands.md` | File exists + content match |
| WP-D-T03 | Migrate Glite 7,660-line MEMORY.md | Corpus mode triggered; archived at `_archive/legacy_MEMORY_2026-MM-DD.md`; ~20 typed entries extracted | `ls _archive/`, `ls project/`, count entries |
| WP-D-T04 | Migrate empty -home-christoph-dotfiles/ legacy | No-op; warning logged "no legacy memory found" | Output inspect |
| WP-D-T05 | Ambiguous file (e.g., `notes.md` not matching any heuristic) | Interactive prompt asks user for target subdir | Manual confirmation |
| WP-D-T06 | Originals preserved | Legacy dir contents bit-identical to pre-migration | `diff` of legacy dir |
| WP-D-T07 | QMD indexes migrated entries | `qmd:search "<known term from migrated content>"` returns hit | QMD search |

---

## Recommended Agents

| Phase | Agent | Rationale |
|-------|-------|-----------|
| Authoring heuristics doc | Claude main loop | Spec writing |
| Corpus extraction (Glite) | `grill-with-docs` corpus pass | Reuse existing knowledge-extraction skill |
| Test fixtures | Subagent in worktree | Isolated test environment |

---

## Review

| Reviewer | Verdict | Notes |
|-----------|---------|-------|
| (pending) | — | — |

---

## Follow-up Issues

| Issue | Severity | Status |
|-------|----------|--------|
| If migration fails partway, recovery path unclear | MEDIUM | Add `--dry-run` mode for migration preview |
| 7,660-line Glite corpus may extract noisy facts | LOW | Manual review post-migration; reject low-quality entries |
