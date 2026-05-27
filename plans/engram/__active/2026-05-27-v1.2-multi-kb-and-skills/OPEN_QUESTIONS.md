---
name: 2026-05-27-v1.2-multi-kb-and-skills-open-questions
title: Open Questions — engram v1.2 — multi-KB + skill subsystem
plan: 2026-05-27-v1.2-multi-kb-and-skills
updated: 2026-05-27
---
<!-- Template: OPEN_QUESTIONS v2 (frontmatter-first) -->

# Open Questions — engram v1.2 — multi-KB + skill subsystem

> A WP cannot reach `stage: hardened` while any question that blocks it has
> `status: open`. Resolving a question appends a Resolution Log row.

## Active Questions

| ID | Question | Context | Blocks | Owner | Status |
|----|----------|---------|--------|-------|--------|
| OQ-2.2-A | Where does the `agent-self` KB instance live by default? Per-agent under `~/.engram/agents/<agent_id>/` so it survives session resets, or fully ephemeral under `/tmp/`? | §15.1, §15.3 SessionEnd archival semantics depend on this. | WP13, WP14 | — | open |
| OQ-2.2-B | Is `embedding-near` bridge `kind` computed against the per-KB QMD vector or against a shared cross-KB embedding pass? Latter is more accurate; former is cheaper and matches ADR-0004. | §15.4 bridge build cost vs quality. | WP15 | — | open |
| OQ-2.2-C | Does `engram agent install --target cursor` write skills as MDC files, or rely on a Cursor MCP-server entry only? Cursor's skill story is less standardized than Claude Code's. | §15.5 installer targets. | WP16 | — | open |
| OQ-2.2-D | Does `engram memory why <id>` walk into archived/dormant ancestors or stop at the active boundary? Archived may be more truthful but exposes legitimately-forgotten material. | §15.7. | WP17 | — | open |
| OQ-2.2-E | If a registered `obsidian-vault` enables Dataview-aware retrieval, does it bypass QMD entirely or feed Dataview results as candidates that QMD then re-ranks? Affects scoring engine purity. | §15.2, §3.6 invariant. | WP13 | — | open |

## Resolution Log

| ID | Question | Resolution | Resolved By | Date |
|----|----------|------------|-------------|------|
| — | — | — | — | — |
