# Related Work — Graphify Integration

**Added:** 2026-05-21 · One-way cross-reference. This memory-overhaul plan takes
**no dependency** on the work below; this note exists only to keep two
similar automation designs aligned.

## The shared pattern

The Graphify integration (`plans/graphify-integration/`) ships a SessionStart
hook, `~/.claude/hooks/graphify-session-refresh.sh`, that **deliberately reuses
this plan's WP12 staleness pattern**: at session start, compare an artifact's
freshness against its source; if stale, refresh in the background; if fresh,
fast no-op.

- **WP12 here** — `~/.claude/hooks/` SessionStart staleness check: compares
  `_meta/.last-daily-update` age; prompts if >25h stale.
- **Graphify** — SessionStart staleness check: compares `graphify-out/graph.json`
  mtime against the newest tracked source; rebuilds in the background if stale.

Same shape, different domains: WP12 keeps the **memory vault** fresh; graphify
keeps a **code knowledge graph** fresh. They do not share code and do not
conflict.

## Why this matters

If WP12's staleness mechanism is later redesigned (e.g. moved to a daemon, or
its timestamp format changed), check whether `graphify-session-refresh.sh`
should follow — keep the two convergent rather than letting them drift into two
unrelated implementations of the same idea.

Graphify is a **code** knowledge-graph tool; this plan's WP10 builds a
**memory-vault** knowledge graph. They are different graph domains — the memory
system does **not** adopt Graphify.

See `plans/graphify-integration/` (OVERVIEW.md Decision E, H).
