# ADR-0008: engram ships as a skill subsystem (orchestrator + chain.yaml + modular children)

- **Status:** Accepted
- **Date:** 2026-05-27
- **Related:** SPEC v2.2 §15.5; user's global "Skill Systems (Agentic-OS Layer)" policy in `~/.claude/CLAUDE.md`

## Context

engram's MCP surface (SPEC §7, 16 verbs) is the contract every agent client
talks to. Above it sits the agent-facing surface — slash skills like
`/remember`, `/recall`, `/handoff`, capture hooks, an installer. Two shapes
for that surface:

1. **A flat bundle of independent skills.** Each skill is a self-contained
   `SKILL.md` that calls one or more MCP verbs. Easy to author, easy to
   install. Hard to compose: `/handoff` that needs to call recall + remember
   + summarize ends up duplicating logic across files.

2. **A skill subsystem.** One orchestrator skill (`engram`) declares a
   deterministic chain of child skills via a sibling `chain.yaml`; child
   skills are small, focused, reusable. Matches the agentic-OS pattern the
   user already adopts in `~/.claude/CLAUDE.md` (`skills/systems/<system>/`).

The user explicitly asked for option 2 ("rich skill set with clean naming →
skill subsystems → skill chaining!").

## Decision

engram ships as a **skill subsystem** under `~/.claude/skills/engram/` (or
the target platform's equivalent — Codex, Gemini CLI, OpenCode, Cursor):

- `~/.claude/skills/engram/SKILL.md` — orchestrator, registers `/engram` and
  is the entry point used by sibling skills that need composition.
- `~/.claude/skills/engram/chain.yaml` — deterministic chain definition for
  the orchestrator. Determinism at runtime; learning (skill improvements) is
  out-of-band, propose-only.
- `~/.claude/skills/engram/<child>/SKILL.md` — modular children. Seed set:
  - `engram-recall` — search across KBs with optional bridge expansion.
  - `engram-remember` — write a memory to a chosen KB with type + frontmatter.
  - `engram-forget` — lifecycle transition (active → dormant → archived) or
    audited `governance_delete`.
  - `engram-recap` — produce a session recap; calls `recall` + `remember`.
  - `engram-handoff` — write a session handoff memory; calls `recap` +
    `remember`.
  - `engram-session-history` — list sessions and their captured episodics.
  - `engram-commit-context` — bundle current task context into a memory.
  - `engram-connect-kb` — register a new KB; calls discover + register +
    schedule jobs.
  - `engram-wiki-ingest` — pull session transcripts into an `llm-wiki` KB.
  - `engram-session-rollup` — run a `kb.recall.rollup` for the current KB.
  - `engram-grill-with-memory` — pressure-test a plan against an engram store
    (analog to the user's existing `grill-with-memory` skill, but engram-
    backed).

Children are **small, focused, reusable**. Each one does exactly one job and
calls MCP verbs. The orchestrator composes them per `chain.yaml`.

**Installer:** `engram agent install [--target claude-code|codex|gemini|...]`:

1. Verifies engramd is running and reachable.
2. Writes the skill files to the platform's skill directory.
3. Registers the MCP server entry in the platform's config.
4. Writes the 8 capture hooks (SPEC §6.1) to the platform's hook config.
5. Prints a one-line success summary with a list of installed skills/hooks.

Uninstall is `engram agent uninstall`. Both are idempotent.

**Self-improvement boundary:** The orchestrator does **not** rewrite child
skills at runtime. Improvements land via the user's existing
`/skill-improve` flow — propose-only, externally evaluated, diff-reviewed,
git-committed. No silent overwrites.

## Consequences

- engram conforms to the agentic-OS layer the user already runs globally; no
  new conventions to learn.
- Each child is independently testable (does it call the right MCP verbs with
  the right shape?) and independently usable (the user can invoke
  `/engram-recall` without going through the orchestrator).
- Chain composition is declarative, not code; reviewing what a chain does is
  reading one YAML file.
- The installer is the single integration point per agent platform. Adding
  support for a new platform means adding a target adapter in the installer,
  not editing every skill.

## Alternatives considered

- **Flat bundle of independent skills.** Smaller initial surface; rejected
  because composition forces duplication and the user explicitly chose the
  subsystem shape.
- **A `~/.claude/skills/systems/engram/` orchestrator-only path.** Cleaner
  per the user's contract, but children would still need a home. Decision:
  put children under `~/.claude/skills/engram/<child>/` (engram-owned
  namespace) and the orchestrator at `~/.claude/skills/engram/SKILL.md`.
  Sticks with one engram namespace.
- **A monolithic `/engram` mega-skill.** Smaller surface; rejected — the user
  policy in `~/.claude/CLAUDE.md` explicitly bans this ("Modular > mega").
