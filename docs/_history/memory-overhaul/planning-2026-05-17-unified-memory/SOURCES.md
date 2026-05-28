# Sources — Memory System Overhaul

**Date**: 2026-05-17
**Updated**: 2026-05-17

> Canonical bibliography for external references. Cite entries in other documents as `[SRC-XX]`.
> Load this file only when agents need to verify or explore cited sources.

## Source Catalog

| ID | Type | Title | Location | Retrieved | Relevance | Notes |
|----|------|-------|----------|-----------|-----------|-------|
| SRC-01 | FILE | Claude Code Memory Improvements - Implementation Guide | `/home/cunger/10_Projects/Agentic AI/Workflows/Notes/agentic-memory/Claude Code Memory Improvements - Implementation Guide.md` | 2026-05-17 | memory-init reference; superseded but archived in `memory-init/references/implementation-guide.md` | External implementation guide. Selectively adopted — see Corrections Log in OVERVIEW.md for what was rejected. |
| SRC-02 | INLINE | Security Review (this session) | `~/.claude/plans/eview-the-media-christoph-samsung-evo990-zesty-lovelace.md` §9 | 2026-05-17 | All WPs — risk mitigations | Run via `security-reviewer` agent on 2026-05-17. Identified 3 CRITICAL/HIGH blockers: prompt injection via MEMORY.md, daily/ not gitignored, memsearch supply chain + OpenAI default. All mitigated in this plan. |
| SRC-03 | INLINE | Architect Review (this session) | `~/.claude/plans/eview-the-media-christoph-samsung-evo990-zesty-lovelace.md` §2-§8 | 2026-05-17 | All WPs — unified architecture | Run via `architect` agent on 2026-05-17. Proposed typed-file + QMD-only architecture. Accepted by user via AskUserQuestion decisions. |
| SRC-04 | INLINE | Existing Memory Infrastructure Survey | (this session output) | 2026-05-17, re-verified 2026-05-18 | migration heuristics | Survey identified legacy memory locations: fsl-cleaningapplication (3 files, 82 lines), glite (1 file, **143 lines** — corrected 2026-05-18, earlier draft erroneously said 7,660). Also: auto-memory writes to `~/.claude/projects/<slug>/memory/` (live, dotfiles slug empty). |
| SRC-05 | FILE | Global CLAUDE.md tiering policy | `/home/cunger/.claude/CLAUDE.md` §"Code Intelligence (LSP-first retrieval)" | 2026-05-17 | WP-E retrieval policy | Mandates LSP → QMD → Grep → Read tiering. Memory retrieval slots into Tier 2 (QMD). |
| SRC-06 | FILE | `capture-learning` skill (existing) | `/home/cunger/.claude/skills/capture-learning/SKILL.md` | 2026-05-17 | WP-C1 reference | 203-line skill with classification state machine. Phase 3 routing extended in WP-C1. |
| SRC-07 | FILE | `handoff` skill (existing) | `/home/cunger/.claude/skills/handoff/SKILL.md` | 2026-05-17 | WP-C2 reference | 42-line skill. Frontmatter has `session_id` for "future `recall` integration" — exactly what WP-C2 implements. |
| SRC-08 | FILE | `grill-with-memory` skill (existing) | `/home/cunger/.claude/skills/grill-with-memory/SKILL.md` | 2026-05-17 | WP-C3 reference | 98-line skill updates CONTEXT.md + docs/adr/. WP-C3 adds memory-stub mirrors. |
| SRC-09 | FILE | `setup-sdd-repo` skill (existing) | `/home/cunger/.claude/skills/setup-sdd-repo/SKILL.md` | 2026-05-17 | WP-C4 reference | 144-line skill. Currently references missing template at `~/.claude/templates/sdd/MEMORY.md` → delegation to `memory-init` fixes the broken state. |
| SRC-10 | FILE | `repo-governance` skill (existing) | `/home/cunger/.claude/skills/repo-governance/SKILL.md` | 2026-05-17 | WP-C5 reference | 102-line skill scans for stale references. Extended in WP-C5 to walk typed subdirs. |
| SRC-11 | FILE | `skill-creator` official plugin | `/home/cunger/.claude/plugins/marketplaces/claude-plugins-official/plugins/skill-creator/skills/skill-creator/SKILL.md` | 2026-05-17 | WP-A, WP-B scaffolding | Official skill scaffolding tool. Used to create memory-init, memory-write, memory-curate. |
| SRC-12 | FILE | Real-world MEMORY.md example | `/home/cunger/.claude/projects/-home-cunger-10-Projects-11-private-OSRS-glite-glite-private/memory/MEMORY.md` | 2026-05-17, re-verified 2026-05-18 | migration test fixture | **143 lines** (corrected — earlier draft erroneously said 7,660). Section-structured single file; migrated by `memory-init` Phase 5. |

## Source Types

| Type | Description | Location Format |
|------|-------------|-----------------|
| URL | Website, documentation | Full URL |
| ARXIV | arXiv paper | arXiv ID (e.g., 2301.12345) |
| GIT | Git repository | URL + optional `@branch` or `@commit` |
| PDF | Local PDF file | Absolute or repo-relative path |
| OBS | Obsidian vault note | `vault:path/to/note.md#heading` |
| FILE | Local file/directory | Absolute or repo-relative path |
| INLINE | In-session agent output | Reference to plan source file + section |

## Unresolved Sources

| Description | Location | Blocker |
|-------------|----------|---------|
| Notion guide referenced by external implementation guide | Linked in SRC-01 header | Not fetched; SRC-01 already captures the key patterns |
| YouTube video referenced by external implementation guide | Linked in SRC-01 header | Not fetched; SRC-01 already captures the key patterns |
