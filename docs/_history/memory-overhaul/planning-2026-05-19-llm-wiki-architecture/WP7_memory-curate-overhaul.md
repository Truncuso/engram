# WP7: memory-curate Overhaul (Fuse memory-rebuild + memory-dedup + memory-stage-commit)

**Severity**: HIGH
**Status**: Phase 2b -- PLANNING
**Depends on**: WP6 (memory-write overhaul)

## Problem

Current `memory-curate` runs a 6-phase pipeline (DISTILL -> PRUNE -> MERGE -> ARCHIVE -> RE-INDEX -> REPORT) on the old typed-file store. The overhaul adds three new modes absorbed from obsidian-wiki: **REBUILD** (archive vault -> clear -> restore from all sources), **DEDUP** (identity resolution -> merge near-duplicate pages -> rewrite wikilinks), and **STAGE** (review staged pages in `_staging/` -> promote or reject). The existing DISTILL/PRUNE/MERGE/ARCHIVE/RE-INDEX/REPORT pipeline is also adapted to operate on memory pages with full frontmatter rather than typed memory files.

## Target Files

- `<framework-repo>/.skills/memory-curate/SKILL.md` -- rewrite adding REBUILD, DEDUP, STAGE modes
- `<framework-repo>/scripts/verify/verify-rebuild.sh` -- REBUILD mode verification
- `<framework-repo>/scripts/verify/verify-dedup.sh` -- DEDUP mode verification
- `<framework-repo>/scripts/verify/verify-stage.sh` -- STAGE mode verification

## Implementation Steps

### Phase 1: Rewrite SKILL.md -- add mode dispatch

Existing pipeline becomes CORE mode (default). Add three new modes:

| Mode | Trigger | Source skill |
|------|---------|-------------|
| **CORE** (default) | `"/memory-curate"` | existing DISTILL->PRUNE->MERGE->ARCHIVE->RE-INDEX->REPORT |
| **REBUILD** | `"rebuild memory vault"`, `"archive and rebuild"`, `"restore from archive"` | memory-rebuild |
| **DEDUP** | `"dedup my memory vault"`, `"find duplicates"`, `"merge duplicates"` | memory-dedup |
| **STAGE** | `"/memory-stage-commit"`, `"review staged pages"` | memory-stage-commit |

Scope defaults to `--both` (global + project). `--global` / `--project` restrict.

### Phase 2: Adapt CORE pipeline to memory format

Replace old typed-file operations with memory-page equivalents:
- **DISTILL**: read `journal/daily/*.md`, propose memory pages in correct categories
- **PRUNE**: surface pages with `lifecycle: draft` untouched > 90 days, or `lifecycle: archived` still in live dirs
- **MERGE**: use QMD similarity on `summary` + `tags` frontmatter fields (cheap pass); confirm before merging
- **ARCHIVE**: move pages with `lifecycle: archived` to `_archive/`, move journals older than 30 days
- **RE-INDEX**: run `qmd update && qmd embed` for all registered collections
- **REPORT**: pages before/after per category, journal entries distilled/archived, token-budget impact

### Phase 3: Implement REBUILD mode

Port memory-rebuild's 3 sub-modes. **Vault-structure note:** REBUILD snapshots live
under `_archive/snapshots/<YYYY-MM-DDTHH-MM-SSZ>/` — the canonical vault has a
single `_archive/` directory (defined by WP-1/WP-3); obsidian-wiki's separate
`_archives/` (plural) is NOT introduced. CORE-mode ARCHIVE (Phase 2) moves stale
pages to `_archive/` directly; REBUILD's timestamped snapshots are namespaced
under `_archive/snapshots/` so the two never collide.

- **Archive-only**: snapshot vault to `_archive/snapshots/<YYYY-MM-DDTHH-MM-SSZ>/`,
  write `archive-meta.json`, log
- **Archive+Rebuild**: archive, clear category dirs + projects/ (preserve
  `.obsidian/`, `_archive/`, `_raw/`), reset index.md/log.md, delete
  `.manifest.json`. Report: vault ready for re-ingest; user chooses what to re-ingest
- **Restore**: list snapshots under `_archive/snapshots/`, confirm selection,
  archive current state (pre-restore), clear live memory vault, copy snapshot content back,
  restore index/log/manifest

Safety rules: always archive before destructive ops, always confirm, never touch `.obsidian/`.

### Phase 4: Implement DEDUP mode

Port memory-dedup's 3 modes (Audit default, Merge `--merge`, Auto-merge `--auto`):
1. Build page registry from all .md files (exclude `_archive/`, `_raw/`, special files, redirect stubs)
2. Detect candidate pairs: title similarity (Jaccard + edit distance + substring + alias cross-match) + semantic signals (category match, tag overlap). Threshold >= 0.75
3. Semantic verdict per pair: `merge`, `keep-separate`, `needs-review`
4. Present audit report with high/medium/needs-review sections
5. Merge: pick canonical by incoming wikilinks > content richness > sources > title length, merge frontmatter + body, write redirect stub at secondary path, rewrite wikilinks vault-wide
6. Update index.md, `.manifest.json`, hot.md, log.md

### Phase 5: Implement STAGE mode

Port memory-stage-commit's workflow:
1. Inventory `_staging/**/*.md` (new pages) and `_staging/**/*.patch.md` (updates)
2. Per-file interactive review: show title/tags/summary/confidence + preview; accept [a]/reject [r]/skip [s]
3. Accept new page: move to final category dir, update index.md. Accept patch: merge additions/deletions into target page, detect conflicts (target updated after staging)
4. Reject: move to `_raw/rejected-<path>.md`
5. Update hot.md, log.md, report

Only active when `WIKI_STAGED_WRITES=true`. `--all` accepts everything; `--reject-all` rejects everything; `--list` shows inventory only.

### Phase 6: Write verification scripts

- `verify-rebuild.sh`: Create test vault with known pages, run archive, clear, verify archive integrity, restore, assert pages intact
- `verify-dedup.sh`: Create vault with near-duplicate pages (e.g. "rsc" vs "react-server-components"), run DEDUP --auto, assert merge + redirect stub + wikilinks rewritten
- `verify-stage.sh`: Place test page in `_staging/`, run STAGE --all, assert page moved to final location + index updated

## Recommended Agents

| Agent | Use |
|-------|-----|
| `skill-creator` | Validate 4-mode dispatch triggers, run evals |
| `implement-and-verify` | Execute phases sequentially |
| `code-reviewer` | Review destructive ops safety (archive-before-clear, confirm gates) |

## Verification

See VERIFICATION.md WP7 section:
- V7.1: REBUILD -> archive created, vault cleared, restore returns pages intact
- V7.2: DEDUP -> near-duplicate merged, redirect stub at secondary path, wikilinks rewritten
- V7.3: STAGE -> staged page promoted, manifest updated, rejected page moved to `_raw/`

## Complexity Delta

- **Added**: 3 new modes (REBUILD/DEDUP/STAGE) with sub-mode dispatch, identity resolution algorithm, redirect stub handling, patch-merge with conflict detection
- **Removed**: old typed-file MERGE logic replaced by memory-page MERGE (operates on frontmatter not MEMORY.md index)
- **Justification**: Three standalone memory lifecycle skills fold into one, sharing scope resolution, confirm gates, archival conventions, and QMD refresh. The CORE pipeline already exists; new modes reuse its REPORT/RE-INDEX phases.

## Migration & Backout

This WP rewrites a working, Phase-1-verified skill — the same discipline as WP-6.

- **Phase-1 behavior that MUST be preserved.** CORE mode is the default and keeps
  every Phase-1 contract: the `/memory-curate` trigger, the six-phase pipeline
  (DISTILL → PRUNE → MERGE → ARCHIVE → RE-INDEX → REPORT), the interactive
  confirm-before-destructive-op gate, the `--global`/`--project`/`--both` scoping,
  and the rule that nothing destructive runs without explicit confirmation.
  Phase 2 of this WP *adapts* the CORE pipeline to operate on memory pages with full
  frontmatter instead of typed memory files — but the phase sequence, the user
  prompts, and the confirm gates are unchanged. REBUILD, DEDUP, and STAGE are
  purely *additive* modes reached by distinct triggers; they never alter CORE.
- **Backout.** The Phase-1 `memory-curate/SKILL.md` is git-tracked (dotfiles repo
  and, post-WP0, the framework repo). If the rewrite regresses, revert the
  SKILL.md to its pre-WP7 commit. No data migration is needed — CORE operated on
  the same `.memory/` tree before and after; the new modes only add capability.
- **Gate.** Before the old behavior is considered replaced, CORE mode is verified
  against the Phase-1 `memory-curate` scenarios (distill synthetic stale daily
  logs; archive logs > 30 days; prune superseded entries; the dry-run/confirm
  gate holds) — all must pass, in addition to V7.1–V7.3 for the new modes.
