---
name: 2026-05-27-v1.2-multi-kb-and-skills
title: engram v1.2 — multi-KB orchestration + skill subsystem
status: active
feature: engram
created: 2026-05-27
updated: 2026-05-27
tags: [engram, memory, multi-kb, skills, sdd]
work_packages:
  - wp13-multi-kb-registry-and-kbplugin-seam
  - wp14-per-kb-lifecycle-workers
  - wp15-cross-kb-bridges-derived-graph
  - wp16-engram-skill-subsystem-and-installer
  - wp17-episodic-semantic-linkage-hardening
relationships:
  - extends: [[2026-05-26-v1-build]]
sources: [SRC-01, SRC-02, SRC-03]
---
<!-- Template: OVERVIEW v2 (frontmatter-first) -->

# engram v1.2 — multi-KB orchestration + skill subsystem

## Executive Summary

v1.2 extends engram with **multi-knowledge-base orchestration**, **per-KB
lifecycle workers**, **cross-KB bridges**, an **agent-installable skill
subsystem**, and **hardened episodic↔semantic linkage**. The authoritative
design is `docs/engram-SPEC.md` §15 (v2.2 amendment, 2026-05-27); this plan
operationalizes §15.1…§15.7 as five cohesive work-packages.

**v1.2 is additive and gated on v1 M2** (WP05 verified). No v1 work-package
is restructured; no v2.1 invariant changes. The microkernel gains exactly
**one** new plugin seam (`KbPlugin`, ADR-0005); the existing dream-job state
machine is reused for KB lifecycle work (ADR-0006); cross-KB connections are
a derived bridge layer (ADR-0007); the agent surface ships as a skill
subsystem (ADR-0008).

**Three milestones:**
- **M3** after WP13 — multiple KBs register, recall fans out, scoring is unified.
- **M4** after WP14 + WP15 — daily ingest + recall rollup + cross-KB bridges run
  through the same job state machine as dreaming, audited in AppLog.
- **M5** after WP16 + WP17 — `engram agent install` ships the skill subsystem
  to Claude Code / Codex / Gemini CLI / OpenCode / Cursor; `engram memory why`
  walks `derived_from` chains across KB instances.

## Active Work Packages

> Derived from WP frontmatter `stage:` by `update_plan.py --sync`. Do not edit
> the Stage column by hand.

| WP | Title | Severity | Stage | Impact |
|----|-------|----------|-------|--------|
| WP13 | Multi-KB registry + `KbPlugin` seam | HIGH | spec | **M3** multi-KB fan-out recall |
| WP14 | Per-KB lifecycle workers (`kb.daily.ingest`, `kb.recall.rollup`, `kb.connect.bridge`) | HIGH | spec | scheduled per-KB work via existing dream-job state machine |
| WP15 | Cross-KB bridges (derived graph layer) | MEDIUM | spec | **M4** opt-in bridge expansion in recall |
| WP16 | engram skill subsystem + `engram agent install` | HIGH | spec | **M5a** drop-in agent surface |
| WP17 | Episodic↔semantic linkage hardening (`derived_from` mandatory + counterfactual gate suite + `engram memory why`) | HIGH | spec | **M5b** auditable provenance chains |

## Execution Strategy

Strict bottom-up. An edge `A → B` means B depends on A (B cannot reach `ready`
until A is `verified`). All five WPs are gated on v1 milestone M2.

```
v1 M2 (WP05 verified)
   |
   v
WP13 ---+----> WP14 ----+
        |               +----> WP15 -+
        |               |            |
        +--> WP17 ------+----> WP16 -+----> M5
                        ^
                        M4 = after WP14+WP15
M3 = after WP13
```

- **WP13** introduces the `KbPlugin` seam, the registry, per-KB QMD + per-KB
  graphify provisioning. Unlocks M3 (multi-KB recall via existing scoring
  engine — no fusion, no RRF).
- **WP14** adds the new job kinds. Reuses SPEC §5.4 state machine.
- **WP15** depends on WP13 (per-KB graphs) + WP14 (the `kb.connect.bridge` job
  kind that builds the bridges). Adds opt-in `expand_via_bridges` recall path.
- **WP17** depends on WP13 (multi-KB context lets `derived_from` chains cross
  KB instances). Promotes `derived_from` to mandatory; adds counterfactual
  gate regression suite; adds `engram memory why` CLI/verb.
- **WP16** depends on WP13 + WP17 (the skill subsystem invokes
  `kb.register`, `expand_via_bridges`, and `memory why` via MCP). Adds the
  orchestrator + chain.yaml + child skills + installer.

## Out of scope (explicit)

- Cross-agent / federated dreaming (D-3 deferral stands).
- Live cross-KB edges (rejected per ADR-0007).
- A standalone KB orchestrator daemon (rejected per ADR-0006).
- Unifying graph + retrieval store (rejected per ADR-0004).
- Tool-mediated mutation of episodic memory (rejected per R-4).
- Restructuring any v1 WP (WP00…WP12) — v1.2 is additive only.
