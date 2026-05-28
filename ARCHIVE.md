# engram — archived material index

This file indexes everything in `docs/_history/`. Each entry is design-trail material — superseded plans, prior SPEC revisions, prior review rounds — preserved here so the audit trail behind engram's current state is readable from one place.

The current truth is always:
- `docs/engram-SPEC.md` — the canonical SPEC (currently v2.3)
- `docs/adr/` — the architectural decision records (active set)
- `plans/engram/__active/` — the current planning packages

Anything in `docs/_history/` is **historical context only**. Do not act on it.

## Index

### `docs/_history/memory-overhaul/` — superseded planning trail (2026-05-17 → 2026-05-21)

The two immediate ancestor planning directories from before engram existed.
See `docs/_history/memory-overhaul/ARCHIVE.md` for the full per-directory rundown.

- `planning-2026-05-17-unified-memory/` (9 files) — Phase 1: unify four Claude-Code memory mechanisms into typed `.md` files with QMD retrieval. **COMPLETE** at archive time. Source-of-truth status: superseded; the user-facing skills from Phase 1 are wrapped by engram's skill subsystem (ADR-0008).
- `planning-2026-05-19-llm-wiki-architecture/` (27 files) — Phase 2: layer an obsidian-wiki–inspired operational model on Phase 1. **MVP shipped (WP0–WP5); WP6–WP16 deferred.** Source-of-truth status: superseded by engram; the visualisation, ingest, and agent-history ideas were redesigned into engram's microkernel architecture.

Origin: `/home/christoph/.claude/plans/memory-overhaul/` in the dotfiles repo. Moved here 2026-05-28 (diff-verified, then deleted from dotfiles; recoverable from dotfiles git history).

### `docs/_history/round-1-2/` — prior SPEC review rounds

Specialist reviews that produced SPEC v2:
- `security-review.md`
- `architecture-review.md`
- `failure-safety-review.md`
- `observability-review.md`
- `SYNTHESIS.md` — how the four reviews merged into v2 (decisions D-1…D-7, R-1…R-5)

These are the "why" trail behind a large block of SPEC v2.1 and v2.2 invariants. Cited from SPEC §0.
