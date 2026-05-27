---
name: 2026-05-27-v1.2-multi-kb-and-skills-handoff
title: Handoff — engram v1.2 multi-KB + skill subsystem plan
plan: 2026-05-27-v1.2-multi-kb-and-skills
updated: 2026-05-27
type: plan-handoff
session_id: "(pending)"
created: "2026-05-27"
project: "/media/christoph/Samsung_Evo990/Projects/00_AI/01_Projects/engram"
branch: "master"
---

# Handoff: engram v1.2 — multi-KB + skill subsystem

> **This is a BUILD handoff, gated on v1 milestone M2.** Do not start v1.2
> work-packages until WP05 (MCP server + CoreService facade, 16 verbs +
> bearer) is `verified` in the v1 plan.

## What this plan does

v1.2 extends engram v2.1 (the bottom-up build now in
`plans/engram/__active/2026-05-26-v1-build/`) with:

1. A 5th plugin seam — `KbPlugin` — so KB *type* is a plugin and KB
   *instance* is a registry row (ADR-0005).
2. Per-KB lifecycle workers reusing the existing dream-job state machine
   (ADR-0006).
3. Cross-KB bridges as a derived graph layer (ADR-0007).
4. An agent-installable skill subsystem under `~/.claude/skills/engram/` and
   equivalents (ADR-0008).
5. `derived_from` mandatory for dreaming products + counterfactual gate
   regression suite + `engram memory why <id>` for chain walks across KBs.

## What the next agent should do first

1. **Verify v1 M2.** Confirm `plans/engram/__active/2026-05-26-v1-build/WP05`
   is `verified`. If not, stop and finish v1 first.
2. **Read in order:** `docs/engram-SPEC.md` §15 (v2.2 amendment),
   `docs/adr/0005…0008-*.md`, then `OVERVIEW.md` in this plan dir.
3. **Pick up WP13.** It is the foundation — every other WP depends on it.
   Drive it from `spec` → `ready` by addressing OQ-2.2-A and OQ-2.2-E first.

## Dependency graph (gates)

```
v1 M2 (WP05 verified)
   |
   v
WP13 ---+----> WP14 ----+
        |               +----> WP15 -+
        |               |            |
        +--> WP17 ------+----> WP16 -+----> v1.2 ready to install
```

## Suspected soft spots (confirm or refute — don't trust blindly)

- **WP13 is the foundation; touching every recall path.** It must not slow
  single-KB recall. Verify the fan-out path has a fast path when only one KB
  is connected (the v2.1-equivalent case).
- **WP16's installer touches user `~/.claude/`.** Idempotence + clean
  uninstall are not nice-to-haves; they are SC-22.
- **OQ-2.2-E (Dataview retrieval) could violate the scoring engine purity
  invariant** (§3.6 says scoring engine owns the formula; QMD = relevance
  only). If Dataview's results bypass QMD, that's a contradiction — needs
  explicit resolution before WP13 reaches `ready`.
- **Cross-KB `derived_from` chains depend on bridges** (`derived-from-citation`
  kind). WP17 must not start before WP15 emits that bridge kind, or define a
  stub kind that WP15 fills in.

## Where to record findings

- Adopt the same template as `plans/engram/__active/2026-05-26-v1-build/`.
- Add rows to `OPEN_QUESTIONS.md` here; never silently resolve.
- Add to `docs/_history/README.md` crosswalk only if a v1.2 decision changes
  a v2.1 commitment (which should not happen — v2.2 is additive).
