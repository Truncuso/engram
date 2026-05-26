---
name: 2026-05-26-v1-build-open-questions
title: Open Questions — engram v1 — bottom-up build
type: open-questions
plan: 2026-05-26-v1-build
updated: 2026-05-26
---
<!-- Template: OPEN_QUESTIONS v2 (frontmatter-first) -->

# Open Questions — engram v1 — bottom-up build

> A WP cannot reach `stage: hardened` while any question that blocks it has
> `status: open`. Resolving a question appends a Resolution Log row.
> Surfaced by the 2026-05-26 multi-agent plan review (see FINDINGS.md /
> REVIEW_REPORT.md). Grilled 2026-05-26 (grill-with-memory): OQ-01/02/03/05/06/07/
> 08/09/10 resolved (see Resolution Log); **OQ-04 remains open** (λ_base value).

## Active Questions

| ID | Question | Context | Blocks | Owner | Status |
|----|----------|---------|--------|-------|--------|
| OQ-01 | Does a successful privacy-filter **match** (e.g. regex hits an AWS key) (A) strip the matched string and write the remaining observation to staging, or (B) drop the whole observation? §6.1 "fail-closed ⇒ drop" plausibly covers filter *errors*; SC-4 requires the secret *stripped* and the rest to *reach staging*. | WP06 says both "strips" and "dropped"; only strip-and-pass satisfies SC-4 (B-3). | WP06 | human | resolved |
| OQ-02 | What makes an episode "independent" for the counterfactual gate (§5.2 step 4)? Candidates: different `session_id` / different `agent_id` / different calendar day / not `derived_from` the same episodic source. Must be a computable predicate. | WP08b can't implement the gate, and SC-5's "2nd episode promotes" has no oracle, without this (B-4). | WP08 | human | resolved |
| OQ-03 | The two-layer contradiction detector's **second** (graph-traversal) layer is undefined. §5.2: "semantic (`sim>0.85 ∧ |Δimportance|>0.3`) + graph traversal." What graph relation/structure constitutes a contradiction? What similarity metric (cosine of embeddings? BM25)? | WP08b `connect.ts` has no complete spec; SC-6 has no deterministic oracle (B-5). | WP08 | human | resolved |
| OQ-04 | What is the default `λ_base` for recency decay (`recency_norm = exp(−λ_eff·hours_since_last_used)`, `λ_eff = λ_base·(1 − importance·0.7)`)? FadeMem is cited but no number appears in spec, plan, or spikes. | WP04 `engine.ts` cannot calibrate the recency component without it (N-2). | WP04 | human | open |
| OQ-05 | The entropy layer default threshold is 40 chars. What is the false-positive rate on normal agent payloads (base64 blobs, stack traces, SQL, minified JS)? Has it been measured against a representative sample? High FP ⇒ silent suppression of legitimate observations. | Affects WP06 default config and SC-4 test design (N-1). | WP06 | human | resolved |
| OQ-06 | Which model produces reliably schema-valid structured output, and is any **local Ollama** model among them? Spike-1b proved only the SDK *rejection* path; every tested model **failed** generation. Name a minimum model (+ measured pass rate) per provider, or decide: cloud-only dreaming (documented cost) vs constrained-decoding. Also specify the CI test path (real model vs mock/fixture). | Blocks WP02 evidence claim, WP08b verify gate, WP07, and SC-3/SC-5/SC-6; reconciles with WP11 "0 skips" gate (B-1, W-9, W-10). | WP02, WP07, WP08, WP11 | human | resolved |
| OQ-07 | Access-control capability failures: emit the spec'd `scope-denied` for both scope and capability violations, or introduce a new `forbidden` code (requires a §6.3/§7.1 spec amendment)? | WP05/phase-1 uses `forbidden`, which is not in §6.3/§7.1 (W-5). | WP05 | human | resolved |
| OQ-08 | A SIGKILL'd worker has a dead PID. §5.4 routes dead-PID → `FAILED` (crash-recovery scan) and stale-heartbeat (>5min) → `TIMED_OUT`; SC-11 mandates `TIMED_OUT` for a crash. Which transition wins for a killed (vs hung) worker, and is the dead-PID → `FAILED` path separately tested? | Spec-internal tension; WP11 SC-11 + WP08-8a watchdog/crash-recovery (W-6). | WP08, WP11 | human | resolved |
| OQ-09 | For a store with `N < 100` total memories, the active-pool floor `min(100, 20%·total)` yields `0.2·N` (e.g. 16 for 80; ~1 for 5). Intended, or should there be a bootstrap minimum (e.g. `≥ max(10, 20%·total)`)? | Affects WP08c merge-validation and the SC-14 test (N-3). | WP08 | human | resolved |
| OQ-10 | In WP12 Stage 7, "no session in 14 days relied on `~/.claude/.memory/`" is to be confirmed via "engram AppLog + daemon logs" — but engram's AppLog never records old-system QMD reads. What is the real evidence source: filesystem `atime`, absence of the QMD collection after re-pointing, or treating the Stage-5 hook swap as the evidence (calendar-only wait)? | Makes the WP12 teardown gate measurable (N-4). | WP12 | human | resolved |

---

## Resolution Log

| ID | Question | Resolution | Resolved By | Date |
|----|----------|------------|-------------|------|
| OQ-01 | privacy filter strip vs drop | **Two paths by trigger:** a filter *match* → redact the matched span (`‹redacted:rule›`) and write the rest to staging (audit logs the rule); a filter *error/timeout* → fail-closed, drop the whole observation. Satisfies SC-4 + §6.1. SC-4 test asserts redaction-marker-present + observation-intact. | grill (human) | 2026-05-26 |
| OQ-02 | "independent episode" | **Independent = different `session_id`** (and not `derived_from` the same parent). Same-session repeats don't corroborate. Computable from capture metadata; SC-5's corroborating episode must be a different session. | grill (human) | 2026-05-26 |
| OQ-03 | contradiction layer 2 | **L1** = cosine sim > 0.85 (QMD vectors) ∧ \|Δimportance\| > 0.3; **L2** = pair shares an entity/subject node OR is ≤2-hop connected in the graphify graph. Both (AND) → `queue_review`, no edge in v1. Uses graphify `get_neighbors`/`shortest_path`. | grill (human) | 2026-05-26 |
| OQ-05 | entropy FP rate | **Configurable** (`entropy_min_len`/`threshold`) + **allowlist** of benign high-entropy shapes (git SHA, UUID, base64 data-URI, paths) + **calibrate** the default against a corpus of real captured payloads during WP06 impl (don't guess a constant). FP impact is bounded by OQ-01 redact-not-drop. | grill (human) | 2026-05-26 |
| OQ-06 | structured-output model | **Local-first, gated:** BLOCK WP08 on a new generation-quality spike that pins a specific local Ollama model and measures schema-valid pass rate (target ≥95% over N seeded `dream-output` generations). Cloud model is the documented fallback if it fails. CI uses the validated model or recorded fixtures. | grill (human) | 2026-05-26 |
| OQ-07 | `forbidden` vs `scope-denied` | **Add `forbidden`** as a distinct capability-failure code (separate from `scope-denied` for scope/visibility). **Requires a §6.3/§7.1 SPEC amendment + an ADR** (gated). §6.3 verb-table error columns updated where capability is the gate. | grill (human) | 2026-05-26 |
| OQ-08 | SIGKILL → FAILED vs TIMED_OUT | **Keep §5.4:** dead-PID → `FAILED` (PID-liveness check / startup crash-recovery scan), live-but-hung → `TIMED_OUT` (heartbeat stale >5min). **Broaden SC-11** to assert a terminal recoverable state (FAILED *or* TIMED_OUT) + `dream resume` works; add one test per path (kill -9 → FAILED; freeze → TIMED_OUT). No §5.4 change. | grill (human) | 2026-05-26 |
| OQ-09 | active-pool floor, small stores | **Keep `min(100, 20%·total)`.** The low floor for young stores is intended — the floor is a mass-archive brake for *mature* stores, not a young-store freeze; a bootstrap minimum would block early dreaming. Document the rationale in WP08c. | grill (human) | 2026-05-26 |
| OQ-10 | WP12 14-day no-access evidence | **Drop the 14-day soak — immediate cutover.** Once Stage-5 hook-swap is verified and engram demonstrably serves context (after the git-commit/export backup), remove the old implementation immediately. **The verified-restorable backup + an exhaustive static-caller grep are the safety net** (no soak). Strike engram AppLog from the gate. | grill (human) | 2026-05-26 |

---

## Escalation Protocol

If a question remains unresolved after 2 rounds of clarification:

1. Halt the affected WP at its current stage.
2. Escalate with: the question, impact on dependent WPs, and a proposed
   decision checkpoint (Option 1 / Option 2).
3. Do not advance the WP past `spec` until resolved.
</content>
