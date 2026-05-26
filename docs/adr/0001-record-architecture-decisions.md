# ADR-0001: Record architecture decisions

- **Status:** Accepted
- **Date:** 2026-05-26

## Context

engram is a non-trivial system (microkernel daemon + detached worker + 4 plugin
seams) built bottom-up over many phases by a human + AI agents. Decisions made
now (stack, boundaries, store layout) will be re-encountered by future sessions
and agents. Without recorded rationale, they get silently re-litigated or
violated.

## Decision

Use Architecture Decision Records. One Markdown file per significant decision in
`docs/adr/NNNN-title.md`, numbered sequentially, never deleted (superseded
decisions are marked, not removed). Format: Context / Decision / Consequences /
Alternatives considered. The authoritative living design stays in
`docs/engram-SPEC.md`; ADRs capture the *why* behind specific choices and the
trail of changes.

## Consequences

- Future agents read `docs/adr/` during orientation (see `AGENTS.md`).
- When implementation forces a new decision, an ADR is written and the spec
  updated in the same change — code and spec do not drift.

## Alternatives considered

- **Rationale only in the spec** — the spec shows the current state, not the
  trail of *why not X*; ADRs preserve the rejected alternatives.
- **Commit messages only** — not discoverable; lost in history.
