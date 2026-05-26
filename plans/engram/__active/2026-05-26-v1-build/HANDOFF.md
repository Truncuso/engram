---
session_id: "492a2eeb-cf56-4e14-8d6f-7e885aed3757"
created: "2026-05-26"
project: "/home/cunger/10_Projects/Agentic AI/engram"
branch: "master"
type: review-handoff
---

# Handoff: Critical review of the engram v1 SDD plan

> **This is a REVIEW mandate, not a build handoff.** The next agent should
> critique this plan — organization, flaws, hardening gaps, risks — and record
> findings. It should NOT start implementing WPs.

## Goal

Fully and critically review the engram v1 build plan at
`plans/engram/__active/2026-05-26-v1-build/` for: **(1) organization/structure,
(2) correctness flaws, (3) hardening gaps, (4) risk.** Produce a findings report
(see "How to record findings"). Confirm or refute the suspected soft spots
below; find new ones. Goal-backward test: *would executing these 13 WPs in this
order actually deliver the 18 SPEC §12.3 success criteria, safely?*

## State of play

- **Plan is complete and passes `validate_plan.py` 100/100** — but a clean
  schema score says nothing about whether the plan is *right*. That is this
  review's job.
- 13 WPs operationalize SPEC §13. WP08 (dreaming) is split 8a–8d. WP12 is a
  9-stage removal cutover of the current memory system. M0/M1/M2 milestones.
- Authored partly by me (WP01, WP08, WP12, OVERVIEW) and partly by parallel
  `gsd-planner` (Sonnet) subagents (WP02-04, WP05-06, WP07/09/10/11) — **the
  subagent-authored WPs have had lighter human review and are the most likely to
  contain drift or over-confident claims.** Scrutinize them harder.
- Design + de-risking are done and trustworthy: `docs/engram-SPEC.md` (v2.1,
  post Round-3 review) and `docs/research/spikes-2026-05-26.md` (4 spikes). The
  spec is the source of truth — flag any WP that contradicts it.

## Review dimensions (cover all four)

1. **Organization / structure** — Is the WP decomposition right? Are boundaries
   clean (one WP = one cohesive deliverable)? Is anything mis-placed, missing, or
   redundant? Is the bottom-up ordering in OVERVIEW's dependency graph actually
   sound, or are there hidden cross-phase dependencies (e.g., a WP needing
   something from a later WP)? Are folder-WP phase splits (esp. WP08 8a–8d)
   genuinely independently verifiable?
2. **Flaws / correctness** — Do WP goals + verification rows actually prove the
   SPEC §12.3 criterion they claim? Are any verification rows untestable as
   written, or testable only after a much later WP (integration risk)? Do the
   dependency edges match reality? Any WP that claims `stage: ready` but isn't?
   Does any WP contradict the spec or an ADR?
3. **Hardening gaps** — Where is the plan thin? The SDD lifecycle is
   `draft→spec→hardened→ready`; most WPs are at `spec`/`ready` but **none have
   been through `grill-with-memory` hardening** (OPEN_QUESTIONS is empty/stub).
   What open questions SHOULD exist and don't? Which WPs need grilling before
   execution? Are the per-phase `goal:`/`verify:` measurable enough to loop on?
4. **Risk** — What is most likely to derail execution? Sequencing risk,
   integration risk (criteria testable only very late: SC-5/7/16 per the
   build-order review), the WP12 cutover touching live user config, the QMD
   embedding-model download, Ollama as a hard WP07 prereq, the
   structured-output-model requirement (WP02/08b). Rank by likelihood × blast
   radius.

## Suspected soft spots (confirm or refute — don't just trust these)

- **WP01 is huge.** 4 phases (schemas, SQLite×4, OCC+write-ordering, init+doctor+
  recall) in one WP. Is it too big even split? Should the SQLite stores or OCC be
  their own WP? It blocks everything — a flaw here is expensive.
- **TODO.md / OPEN_QUESTIONS.md / VERIFICATION.md at plan root are stubs.** The
  plan's cross-WP verification matrix and open-questions register were never
  filled. Is that acceptable for a "ready" plan, or a gap?
- **No WP has been grilled.** `hardened` stage requires `grill-with-memory` +
  zero open questions. Every WP skipped it. Which ones genuinely need it before
  execution (WP08, WP06 privacy filter, WP12 cutover are my guesses)?
- **WP08c claims "zero LLM, testable with hand-written manifests."** Verify that
  the orchestrator truly needs no worker run — if classification secretly needs
  worker output shape that only 8b produces, 8c isn't independently testable.
- **WP11 depends on 7 WPs.** Are its SC→WP mappings correct and complete? Does
  every one of the 18 SPEC §12.3 criteria map to exactly one delivering WP, with
  none orphaned?
- **WP12 edits live `~/.claude` config + dotfiles.** Is the 9-stage ordering
  truly non-breaking? Is the rollback story (git-commit-before-migrate) airtight?
  Does it correctly preserve grill-with-memory's non-memory value?
- **Subagent claims to spot-check:** WP04's m_v worked example (0.42), WP03's
  deindex→deactivate mapping, WP07's graphify `ingestEdges`=batch-rebuild, WP09's
  mitigation-home-WP table (does it correctly say what's implemented where?).

## How to record findings

Write findings to `FINDINGS.md` (and/or `REVIEW_REPORT.md`) in this plan dir.
Use severity tags **BLOCK / WARN / NOTE**. For each: the file/section, the issue
in one line, and a suggested fix. Populate `OPEN_QUESTIONS.md` with any genuine
open questions surfaced. Do NOT edit WP content directly — propose; the human
gates `spec→hardened` and `hardened→ready`.

## Skills / agents to use

- `plan-maintain` (the `--review` / hardening path of the SDD lifecycle) — the
  canonical tool for reviewing/hardening an existing plan.
- `plan-reviewer` agent — audits plan vs. spec/philosophy for gaps,
  contradictions, wrong statements.
- `architect-reviewer` — for the structure/ordering dimension.
- `grill-with-memory` — to actually harden the WPs that need it (after review
  identifies which).
- `gsd-plan-checker` — goal-backward "will this plan achieve the goal" check.

## Artifacts

- Plan: `plans/engram/__active/2026-05-26-v1-build/` (OVERVIEW.md + 13 WPs)
- Spec (source of truth): `docs/engram-SPEC.md` (v2.1)
- Spikes (de-risking evidence): `docs/research/spikes-2026-05-26.md`
- ADRs: `docs/adr/0001-0004*.md`
- Rules: `.claude/rules/engram-{architecture,typescript}.md`
- Commits: `aeffa3d` (scaffold), `4757622` (plan) on `master`,
  github.com/Truncuso/engram
- Validation baseline: `python3 ~/.claude/skills/plan-manager/scripts/validate_plan.py --plan-dir <this dir>` → 100/100 (schema only)
