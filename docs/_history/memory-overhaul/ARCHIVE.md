# memory-overhaul — archived planning trail

**Archived:** 2026-05-28
**Original location:** `/home/christoph/.claude/plans/memory-overhaul/` (dotfiles repo)
**Status:** Superseded by engram (this repo). Preserved for design-trail auditability.

## Why these are here

The two planning directories below are the immediate ancestors of engram. SPEC §0 records that "the prior effort (`plans/memory-overhaul/`) was scrapped — it was a config-dir transplant, not a standalone system, and it locked an archived dependency (Kuzu). engram is the redesign." The plans themselves are not engram — but the trail through them (what shipped, what was rejected, what was deferred) is load-bearing context for any future architectural change in this repo, and would be lost if they only lived in the dotfiles repo. They were moved here so engram is self-contained.

The dotfiles copy was deleted on 2026-05-28 after a clean `diff -r` round-trip verified an exact match against the copy in this directory; the deletion is recoverable from dotfiles git history if anyone wants it back.

## What the two directories contain

### `planning-2026-05-17-unified-memory/` (9 files)

**Status at archive time:** COMPLETE — Phase 1 shipped end-to-end (2026-05-19, 37/37 verified).

Goal: unify four uncoordinated memory mechanisms into one architecture — global `~/.claude/.memory/` + per-repo `<repo>/.memory/`, typed `.md` files with frontmatter, QMD retrieval as Tier 2, SessionStart memory-load hook, `memory-init` skill that scaffolds both scopes idempotently.

What shipped: the directory structure, the SessionStart + SessionEnd hooks, the QMD index refresh, three skills (`memory-init`, `memory-write`, `memory-curate`), edits to five existing skills (`capture-learning`, `handoff`, `grill-with-memory`, `setup-sdd-repo`, `repo-governance`), and the `~/.claude/CLAUDE.md` override block that redirects the harness's hard-coded auto-memory path to `~/.claude/.memory/`.

What was rejected and why: raw transcript capture via Stop hook + cron-based distillation + a separate `memsearch` index (rejected on security grounds — FINDINGS.md 3–6); the per-slug path `~/.claude/projects/<slug>/memory/` (deprecated because the harness still injects it via an uneditable system prompt — the override block in CLAUDE.md is the workaround).

What was deferred: every lifecycle operation past plain capture — synthesis, digest, ingest pipelines, cross-linking, visualisation. Those rolled into Phase 2 below.

### `planning-2026-05-19-llm-wiki-architecture/` (27 files)

**Status at archive time:** PARTIALLY SHIPPED — MVP milestone (WP0–WP5) approved and partially executed; WP6–WP16 remained in PLANNING.

Goal: layer the obsidian-wiki framework's operational model (~35 skills, autonomous upkeep agent, public framework repo, per-project Obsidian vault) on top of Phase 1's typed-file store.

MVP scope (WP0–WP5): framework repo scaffold, vault setup, `llm-memory` core architecture skill, `memory-init` overhaul (fused with memory-setup), core ingest + query skills, housekeeping skills (lint, status, cross-link).

Deferred (WP6–WP16): `memory-write` overhaul, `memory-curate` overhaul, synthesis/digest/tag/research/ingest, agent-history ingest, visualisation + export, existing skill edits, cron setup, framework repo finalisation, autonomous upkeep agent, E2E verification + documentation, the LLM-powered memory agent.

The full WP files (`WP0_*.md` through `WP16_*.md`) are preserved in the directory. Skill-creator review findings, architecture review (2026-05-19), open questions, sources, related-work analysis, and the overview document are all included.

## Why engram supersedes this

The two planning dirs describe a Claude-Code-internal memory mechanism — extending `~/.claude/` skills, hooks, and per-project `.memory/` directories. Engram inverts that: it is a standalone installable product (`engramd` daemon + MCP server + CLI + v1 read-only dashboard), agent-agnostic, with a microkernel + detached dreaming worker architecture (ADR-0001), a four-plugin seam (retrieval/graph/LLM/capture, ADR-0004) + KB plugin (ADR-0005), and a typed memory model (semantic/episodic/procedural/contextual). The Claude-Code skill suite from these plans is not abandoned — engram's skill subsystem (ADR-0008, WP16) re-implements the user-facing skills as thin clients of the engram MCP, so the surface the user already knew is preserved while the backend is principled.

Specifically: ideas from these plans that survived into engram include the typed `.md` schema with frontmatter, two scopes (global + per-project), QMD as the retrieval plugin (ADR-0004), the SessionStart memory-load hook (becomes capture hooks in engram), and the per-project Obsidian-vault treatment (becomes the wiki KbPlugin, ADR-0005). Ideas that were dropped: the four-mechanism unification was rendered moot when engram replaced them all; the override block in `~/.claude/CLAUDE.md` is no longer needed once engram is the canonical store.

## How to reference these from engram going forward

- SPEC §0 and the v2.3 changelog already cite this archive.
- ADR-0008 (engram skill subsystem + installer) is the bridge between this user-facing surface and engram's standalone-app architecture.
- The new milestone v1.3 (`plans/engram/__active/2026-05-28-v1.3-ingest-formats-and-dashboard/`) is descended directly from Phase 2's WP10 (visualisation + export) and WP4 (core ingest + query) — see those WP files for the original framing.

Read these directories the way you would read previous spec revisions: not as the current truth, but as the audit trail that produced it.
