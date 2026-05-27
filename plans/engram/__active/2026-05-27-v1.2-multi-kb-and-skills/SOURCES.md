---
name: 2026-05-27-v1.2-multi-kb-and-skills-sources
title: Sources — engram v1.2 — multi-KB + skill subsystem
type: sources
plan: 2026-05-27-v1.2-multi-kb-and-skills
updated: 2026-05-27
---
<!-- Template: SOURCES v2 (frontmatter-first) -->

# Sources — engram v1.2 — multi-KB + skill subsystem

> Canonical bibliography. Cite entries elsewhere as `[SRC-XX]`. Load this file
> only when agents need to verify or explore cited sources.

## Source Catalog

| ID | Type | Title | Location | Retrieved | Relevance | Notes |
|----|------|-------|----------|-----------|-----------|-------|
| SRC-01 | FILE | engram SPEC v2.2 §15 — Multi-KB Orchestration & Agent Skills | docs/engram-SPEC.md | 2026-05-27 | authoritative | The contract every v1.2 WP implements. |
| SRC-02 | FILE | Agentic-memory survey (light pass on 7 OSS repos) | docs/research/agentic-memory-survey-2026-05-27.md | 2026-05-27 | inspiration ledger | Adopt/adapt/reject verdicts tied to WPs. |
| SRC-03 | FILE | ADRs 0005–0008 (KbPlugin, lifecycle workers, bridges, skill subsystem) | docs/adr/0005…0008-*.md | 2026-05-27 | architecture decisions | Bound the WP scopes. |
| SRC-04 | FILE | Crosswalk for archived round-1/2 review findings | docs/_history/README.md | 2026-05-27 | audit trail | Where each D-/R-/OQ- landed in v2.1+. |
| SRC-05 | URL | Pratiyush/llm-wiki | https://github.com/Pratiyush/llm-wiki | 2026-05-27 | KB type model | Informs `llm-wiki` KbPlugin (WP13). |
| SRC-06 | URL | rohitg00/agentmemory | https://github.com/rohitg00/agentmemory | 2026-05-27 | hooks + seed skill set | Informs WP16 capture-hook surface and seed children. |
| SRC-07 | URL | letta-ai/letta (MemGPT) | https://github.com/letta-ai/letta | 2026-05-27 | memory hierarchy | Working/episodic framing; episodic-mutation pattern rejected. |
| SRC-08 | URL | mem0ai/mem0 | https://github.com/mem0ai/mem0 | 2026-05-27 | extract-then-update | Informs dreaming-worker output contract. |
| SRC-09 | URL | topoteretes/cognee | https://github.com/topoteretes/cognee | 2026-05-27 | multi-dataset scoping | Informs registry + bridges. |
| SRC-10 | URL | agno-agi/agno | https://github.com/agno-agi/agno | 2026-05-27 | multi-agent shared memory | Cross-agent dreaming explicitly deferred (D-3). |
| SRC-11 | URL | kingjulio8238/Memary | https://github.com/kingjulio8238/Memary | 2026-05-27 | entity-grounded recall | Informs entity-match bridge `kind`. |

## Source Types

| Type | Description | Location Format |
|------|-------------|-----------------|
| URL | Website, documentation | Full URL |
| ARXIV | arXiv paper | arXiv ID (e.g., 2301.12345) |
| GIT | Git repo | URL + optional `@branch` or `@commit` |
| PDF | Local PDF file | Absolute or repo-relative path |
| OBS | Obsidian vault note | `vault:path/to/note.md#heading` |
| FILE | Local file/directory | Absolute or repo-relative path |

## Unresolved Sources

| Description | Location | Blocker |
|-------------|----------|---------|
| — | — | — |
