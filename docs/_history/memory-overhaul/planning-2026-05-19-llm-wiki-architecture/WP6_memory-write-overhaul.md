# WP6: memory-write Overhaul (Fuse memory-capture + memory-update)

**Severity**: HIGH
**Status**: Phase 2b -- PLANNING
**Depends on**: WP0 (framework repo), WP1 (vault + `~/.llm-memory/config`), WP2 (llm-memory
schema + frontmatter spec), WP3 (memory-init + hook scripts), WP4 (ingest+query skills),
WP5 (housekeeping skills)

## Problem

Current `memory-write` writes simple typed `.md` files with minimal frontmatter (`name`, `description`, `type`, `scope`, `created`, `origin_session`). It lacks the memory lifecycle: no provenance/confidence/lifecycle/tier/summary fields, no conversation-to-note capture, no cross-project sync. The overhaul fuses two obsidian-wiki skills as modes: **memory-capture** (classify conversation content, rewrite as declarative knowledge, place in correct vault category) and **memory-update** (project-aware git-delta knowledge sync into vault). Result: a single portable `memory-write` capable of writing full memory pages from any project context.

## Target Files

- `<framework-repo>/.skills/memory-write/SKILL.md` -- complete rewrite adding CAPTURE + UPDATE mode dispatch
- `<framework-repo>/scripts/verify/verify-capture.sh` -- CAPTURE mode verification
- `<framework-repo>/scripts/verify/verify-update.sh` -- UPDATE mode verification

## Implementation Steps

### Phase 1: Rewrite SKILL.md with mode dispatch

Retain existing scope/type routing. Add mode dispatch:

| Mode | Trigger | Source skill |
|------|---------|-------------|
| **WRITE** (default) | `"remember this"`, `"/memory-write"` | existing + frontmatter upgrade |
| **CAPTURE** | `"save this"`, `"capture this"`, `"file this conversation"` | memory-capture |
| **UPDATE** | `"update memory vault"`, `"/memory-update"`, `"sync to memory vault"` | memory-update |

### Phase 2: Upgrade WRITE mode frontmatter

Extend typed-file frontmatter to full memory spec: `title`, `tags` (2-5 from taxonomy), `sources`, `created`, `updated`, `summary` (1-2 sentences, <=200 chars, `>-` folded scalar), `provenance` (extracted/inferred/ambiguous fractions), `base_confidence`, `lifecycle` (draft|compiled|reviewed), `lifecycle_changed`, `tier` (pillar|core|supporting|peripheral), `aliases`, `relationships`. Body follows concept/synthesis/reference structure from memory-capture templates.

### Phase 3: Implement CAPTURE mode

Port memory-capture's 7-step workflow:
1. Identify preservable content from current conversation; skip logistics
2. Classify as synthesis/concept/source/decision/session -> target folder
3. Rewrite as declarative present-tense knowledge (not transcript)
4. Slugify title (lowercase-hyphenated, max 50 chars)
5. Write page with full frontmatter + body template matching content type
6. Update index.md, log.md, hot.md
7. Confirm save path

Enforce >=2 wikilinks per note. Apply provenance markers (`^[inferred]`, `^[ambiguous]`).

### Phase 4: Implement UPDATE mode

Port memory-update's 7-step workflow:
1. Understand project (README, build files, git log)
2. Compute delta vs `.manifest.json`; skip if no meaningful changes
3. Distill: architecture decisions, patterns, trade-offs, key abstractions
4. Write to `projects/<name>/` with overview `<name>.md` + category subdirs; cross-project knowledge goes to global categories
5. Cross-link: new pages -> existing pages and back
6. Update `.manifest.json`, index.md, log.md, hot.md
7. Report

Karpathy heuristic: if codebase answers it, skip. If re-deriving requires 20 commits of git blame, memory it.

### Phase 5: Write verification scripts

- `verify-capture.sh`: Inject test transcript, run CAPTURE, assert output has all frontmatter fields, >=2 wikilinks, correct content type, tracking files updated.
- `verify-update.sh`: Create test project with git history, run UPDATE, assert `projects/<name>/<name>.md` created, frontmatter complete, manifest updated.

### Phase 6: QMD refresh integration

All modes append QMD refresh step (best-effort, no rollback on failure): `qmd update` then `qmd embed` if vectors stale.

## Recommended Agents

| Agent | Use |
|-------|-----|
| `skill-creator` | Validate SKILL.md triggers, run evals |
| `implement-and-verify` | Execute phases, write verification scripts |
| `code-reviewer` | Review frontmatter completeness, mode dispatch clarity |

## Verification

See VERIFICATION.md WP6 section:
- V6.1: WRITE mode -> full memory frontmatter (provenance, confidence, lifecycle, tier, summary, relationships)
- V6.2: CAPTURE mode -> conversation produces memory note with correct content type, >=2 wikilinks, tracking files updated
- V6.3: UPDATE mode -> git-delta sync produces project overview page, manifest recorded

## Complexity Delta

- **Added**: 3-mode dispatch, memory frontmatter schema, declarative knowledge rewriting heuristics, git-delta diff logic, project detection
- **Removed**: none (existing simple flow is retained as WRITE mode default)
- **Justification**: Two standalone skills (memory-capture, memory-update) fold into one skill as modes, sharing scope resolution, QMD refresh, and config protocol. Net reduction in skill count while adding full memory lifecycle.

## Migration & Backout

This WP rewrites a working, Phase-1-verified skill — the same discipline as WP-7.

- **Phase-1 behavior that MUST be preserved.** WRITE mode is the default mode and
  keeps every Phase-1 contract: the existing triggers (`"remember this"`, `"note
  that"`, `"save this"`, `"/remember"`, `"/forget"`), the scope decision
  (global vs project) and type routing (user/feedback/project/reference), the
  dedup check, the `MEMORY.md`/`index.md` pointer update, and the on-write
  `qmd-refresh.cjs --force` call. The memory frontmatter is a strict *superset* of
  the Phase-1 fields — a Phase-1 fact re-written by the new skill must still
  carry `name`, `description`, `type`, `scope`, `created`, `origin_session`.
  CAPTURE and UPDATE are *additive* modes; they never alter WRITE's behavior.
- **Backout.** The Phase-1 `memory-write/SKILL.md` is git-tracked in the dotfiles
  repo (and, post-WP0, in the framework repo). If the rewrite regresses, revert
  the SKILL.md to its pre-WP6 commit; no data migration is needed because the new
  frontmatter is a superset (old readers ignore the extra fields).
- **Gate.** Before the old behavior is considered replaced, the rewrite is
  verified against the Phase-1 `memory-write` test scenarios (user fact → global,
  project decision → project, glossary mirror, feedback correction, `/forget`,
  dedup hit) — all must pass, in addition to V6.1–V6.3.
