---
name: wp08-open-questions
title: Open Questions — Dreaming worker + orchestrator
type: wp-open-questions
wp: wp08-dreaming-worker-orchestrator
updated: 2026-05-26
---
<!-- Template: WP-folder OPEN_QUESTIONS v2 (frontmatter-first) -->

# Open Questions — WP08: Dreaming worker + orchestrator

> A WP cannot reach `stage: hardened` while any question has `status: open`.

> Canonical register is the plan-level `OPEN_QUESTIONS.md`; rows here mirror the
> ones that block this WP. Resolve there; this is the WP-local view.

## Active Questions

| ID | Question | Context | Blocks | Status |
|----|----------|---------|--------|--------|
| OQ-02 | What makes an episode "independent" for the counterfactual gate (§5.2 step 4)? session/agent/day/source? | gate + SC-5 promotion need a computable predicate | WP08 phase-8b | open |
| OQ-03 | Two-layer contradiction: the graph-traversal (2nd) layer is undefined (§5.2). What relation = contradiction? metric? | WP08b connect.ts + SC-6 oracle | WP08 phase-8b | open |
| OQ-06 | Which model reliably *produces* schema-valid structured output; is any local Ollama model among them? CI path? | spike proved rejection, not generation; SC-3/5/6 | WP08 phase-8b | open |
| OQ-08 | SIGKILL'd worker (dead PID) → FAILED (§5.4 crash-recovery) or TIMED_OUT (SC-11 text)? which fires for killed vs hung? | WP08-8a watchdog vs WP13-1 crash-recovery | WP08 phase-8a | open |
| OQ-09 | Active-pool floor `min(100,20%·total)` is tiny for small/new stores. Bootstrap minimum intended? | WP08c merge-validation + SC-14 | WP08 phase-8c | open |

## Resolution Log

| ID | Question | Resolution | Date |
|----|----------|------------|------|
|  |
