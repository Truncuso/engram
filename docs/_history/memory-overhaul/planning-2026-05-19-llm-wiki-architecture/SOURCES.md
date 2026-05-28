# Sources — Memory System Overhaul: LLM-Memory Architecture

**Date**: 2026-05-19
**Updated**: 2026-05-19

> Canonical bibliography for external references. Cite entries in other documents as `[SRC-XX]`.
> Load this file only when agents need to verify or explore cited sources.

## Source Catalog

| ID | Type | Title | Location | Retrieved | Relevance | Notes |
|----|------|-------|----------|-----------|-----------|-------|
| SRC-01 | GIT | obsidian-wiki: LLM-maintained Obsidian knowledge base | `/home/cunger/10_Projects/Agentic AI/Workflows/automatic-workflows/obsidian-wiki/` | 2026-05-19 | ALL WPs | MIT license, 35 skills, full memory lifecycle. Source of architecture patterns, page templates, frontmatter schema, config resolution protocol, setup.sh |
| SRC-02 | FILE | Phase 1 Memory Overhaul Plan | `~/.claude/plans/memory-overhaul/planning-2026-05-17-unified-memory/OVERVIEW.md` | 2026-05-19 | WP-0, WP-1, WP-3, WP-11 | Existing infrastructure (hooks, QMD, 6 skills, 37/37 verified). Phase 1 is 85% complete. |
| SRC-03 | FILE | QMD-vs-MemSearch Analysis | `/home/cunger/10_Projects/Agentic AI/Workflows/Notes/agentic-memory/QMD-vs-MemSearch-Analysis.md` | 2026-05-19 | Architecture | Definitive analysis rejecting memsearch in favor of QMD. 3 CRITICAL security blockers found in memsearch. |
| SRC-04 | URL | Karpathy LLM Memory Gist | `https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f` | 2026-05-19 | WP-2 | Conceptual foundation: compile-once memory vault pattern, three-layer architecture (raw → memory vault → schema), Obsidian as viewer |
| SRC-05 | FILE | Current CLAUDE.md Memory Override Block | `~/.claude/CLAUDE.md` (lines 175–189) | 2026-05-19 | WP-1, WP-3 | Current override block redirecting memory to `~/.claude/.memory/` — must update to `~/memory/` |
| SRC-06 | FILE | SessionStart Memory Hook | `~/.claude/scripts/hooks/session-start-memory.cjs` | 2026-05-19 | WP-1 | Must update `globalDir` path and add staleness detection |
| SRC-07 | FILE | QMD Refresh Script | `~/.claude/scripts/hooks/qmd-refresh.cjs` | 2026-05-19 | WP-1 | Must update collection paths and add per-project vault scanning |
| SRC-08 | FILE | Claude Code Memory Improvements — Implementation Guide | `/home/cunger/10_Projects/Agentic AI/Workflows/Notes/agentic-memory/Claude Code Memory Improvements - Implementation Guide.md` | 2026-05-19 | Historical | Original (rejected) memsearch-based plan. Reference for design evolution. |
| SRC-09 | FILE | Memory Init Skill (current) | `~/.claude/skills/memory-init/SKILL.md` | 2026-05-19 | WP-3 | Existing skill to overhaul — 7-phase scaffold with idempotent design |
| SRC-10 | FILE | Memory Write Skill (current) | `~/.claude/skills/memory-write/SKILL.md` | 2026-05-19 | WP-6 | Existing skill to overhaul — scope+type routing, dedup |
| SRC-11 | FILE | Memory Curate Skill (current) | `~/.claude/skills/memory-curate/SKILL.md` | 2026-05-19 | WP-7 | Existing skill to overhaul — DISTILL→PRUNE→MERGE→ARCHIVE→RE-INDEX→REPORT |
| SRC-12 | URL | QMD (tobi/qmd) | `https://github.com/tobi/qmd` | 2026-05-19 | WP-1 | Local markdown search engine, hybrid BM25+vector, MCP server. Chosen retrieval backend. |

## Source Types

| Type | Description | Location Format |
|------|-------------|-----------------|
| URL | Website, documentation | Full URL |
| GIT | Git repository | Absolute path or URL + `@branch` |
| FILE | Local file/directory | Absolute or repo-relative path |

## Unresolved Sources

Sources found but not yet integrated:
| Description | Location | Blocker |
|-------------|----------|---------|
| Stephen Chin — Context Graphs (Neo4j) talk | `https://www.youtube.com/watch?v=eW_vxrjvERk` | Low priority; context graph exploration is downstream of memory vault setup |
