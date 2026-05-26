---
name: wp08-dreaming-worker-orchestrator
title: Dreaming worker + orchestrator
type: work-package
stage: spec
severity: HIGH
created: 2026-05-26
updated: 2026-05-26
plan: 2026-05-26-v1-build
tags: [dreaming, worker, orchestrator, jobs, merge]
relationships:
  - depends_on: [[wp06-capture-captureintake-staging]]
  - depends_on: [[wp07-ingest-worker-graphify-graphplugin-ollama]]
  - depends_on: [[wp02-plugin-host-llmplugin-vercel-ai-sdk]]
  - blocks: [[wp09-threat-model-hardening-planted-attack-tests]]
  - blocks: [[wp11-e2e-verification-18-success-criteria]]
sources: [SRC-01]
phases: [phase-8a-job-state-machine-spawning, phase-8b-worker-domain-logic, phase-8c-orchestrator-merge-classification, phase-8d-triggers-rollback-dream-verbs]
---
<!-- Template: WP-folder OVERVIEW v2 (frontmatter-first) -->

# WP08: Dreaming worker + orchestrator

> Folder work package. Phases live in `phase-8N-*.md`. `stage:` advances only
> when all phase `phase_status:` are `done`.

## Problem

Dreaming is the heart of engram: a **detached worker** distills staging
observations into typed memories, connects them, re-weights importance, and
learns from failures — while the **orchestrator** (in the core daemon) queues
jobs, classifies the worker's manifest deterministically, merges safe hunks, and
queues gated ones for review. A worker crash must NEVER touch the core daemon.

This was SPEC §13 phase 8 — a single phase spanning ~18 sub-systems (state
machine, heartbeat, atomic claim, distill/connect/re-weight/verify, manifest,
classification, three-way merge, merge validation, active-pool floor, rate
limits, review queue, idempotent resume, rollback, budget, contradiction
detection, counterfactual gate, episodic immutability). Unverifiable as one unit,
so it is **split into four phases**, each independently testable — 8a and 8c with
ZERO LLM calls.

SPEC refs: §5 (all), §5.4 (state machine), §5.5 (safe/gated), §7.2/§5.4 (merge
OCC, C12), §9.4 (merge validation), §9.5 (forgetting rails, rollback), §8.3
(S-05/S-11/S-12 — C6 enforcement).

## Target Files

- `src/worker/dream.ts` — detached worker entry (distill/connect/re-weight/verify)
- `src/worker/heartbeat.ts` — 30s heartbeat to jobs.db
- `src/core/orchestrator/{queue,claim,watchdog}.ts` — job lifecycle, atomic claim, reaper
- `src/core/orchestrator/classify.ts` — deterministic safe/gated predicate (§5.5)
- `src/core/orchestrator/merge.ts` — three-way merge + merge validation (§9.4)
- `src/core/orchestrator/review-queue.ts` — gated hunks
- `src/core/dreaming/{config,triggers,rollback}.ts` — `.dreaming/*.md`, triggers, `git revert`
- `src/mcp/verbs/dream.ts` — replaces WP05 stubs

## Phases

| Phase | Goal | Status |
|-------|------|--------|
| [8a](phase-8a-job-state-machine-spawning.md) | Job state machine + detached spawn + heartbeat + watchdog + idempotent resume + budget ceiling (zero LLM) | pending |
| [8b](phase-8b-worker-domain-logic.md) | Worker distill/connect/re-weight/verify → manifest; episodic immutability; counterfactual gate; contradiction detection; rate limits | pending |
| [8c](phase-8c-orchestrator-merge-classification.md) | Deterministic safe/gated; three-way merge; merge validation + active-pool floor; review queue (zero LLM, hand-written manifests) | pending |
| [8d](phase-8d-triggers-rollback-dream-verbs.md) | SessionEnd/cron/cumulative-importance triggers; `git revert` rollback; `dream.*` MCP verbs replace stubs | pending |

## Verification

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| W08-1 | worker crash → core healthy → TIMED_OUT → resume idempotent, no staging loss | per §5.4 / SC-11-crash | integration (8a) |
| W08-2 | dream distills staging → memories w/ derived_from; uncorroborated procedural at confidence 0.3/dormant; 2nd episode promotes | SC-5 | integration (8b) |
| W08-3 | two-layer contradiction → review queue, no auto-resolve, no `contradicts` edge (C5) | SC-6 | integration (8b) |
| W08-4 | safe/gated classification deterministic (LLM cannot self-classify, S-11) | per §5.5 | unit (8c) |
| W08-5 | active-pool floor blocks mass-archive regardless of merge_policy:always-auto | SC-13 | integration (8c) |
| W08-6 | manifest schema-validation rejects injected frontmatter → job FAILED (C6/S-05) | SC-11 | integration (8c) |
| W08-7 | forget → dormant after ≥2 consecutive sub-threshold runs; still searchable | SC-7 | integration (8c/8d) |
| W08-8 | `dream.trigger` out-of-scope → scope-denied (S-13) | SC-12 | integration (8d) |
| W08-9 | rollback reverts a merged dream (git revert) | per §9.5 | integration (8d) |

## Recommended Agents

| Phase | Agent | Rationale |
|-------|-------|-----------|
| 8a | `typescript-pro`, `database-reviewer` | state machine + jobs.db transactions |
| 8b | `typescript-pro`, `llm-architect` | worker LLM pipeline + structured output |
| 8c | `typescript-reviewer`, `security-reviewer` | merge correctness + S-05/S-11/S-12 |
| 8d | `typescript-pro` | triggers + verb wiring |
| all | `code-reviewer`, `tdd-guide` | per-phase gate |
