---
name: 2026-05-26-v1-build-review-report
title: Review Report — engram v1 — bottom-up build
type: review-report
plan: 2026-05-26-v1-build
created: 2026-05-26
updated: 2026-05-26
sources: [SRC-01, SRC-02, SRC-03]
---

# Review Report — engram v1 bottom-up build

Multi-agent parallel critical review per `HANDOFF.md`. Five reviewers, one
mandate each, synthesized here. Ranked, severity-tagged findings live in
`FINDINGS.md`; genuine open questions in `OPEN_QUESTIONS.md`. This report is the
per-dimension narrative + the verdict.

**Reviewers dispatched (parallel):**
1. architect-reviewer — organization / structure / decomposition / dependency ordering
2. plan-reviewer — correctness (WP goals vs §12.3, dependency edges, spec/ADR contradictions)
3. plan-reviewer — hardening gaps (measurability, missing open questions, grilling)
4. architect-reviewer — risk (sequencing, integration, live-config, external prereqs)
5. gsd-plan-checker — goal-backward (do 13 WPs in this order deliver 18 SCs?)

Grounding: `docs/engram-SPEC.md` v2.1 (§3.6, §5.2/5.4/5.5, §6.1/6.3, §8.3/8.4,
§9.1/9.9, §12.3) and `docs/research/spikes-2026-05-26.md`.

---

## Verdict

**Ready after listed fixes — needs targeted re-hardening of WP06 + WP08 before
execution.**

The plan is structurally sound: WP01–WP11 map 1:1 to SPEC §13; WP00 (scaffold) and
WP12 (cutover) are justified additions; the WP08 8a–8d split is the right call;
goal-backward coverage is **18/18 SCs, 0 orphaned**; M0/M1/M2 are real automated
checkpoints. The schema score (100/100) reflects this — but schema-clean ≠ right,
which is what this review tested.

The blockers are not architectural — they are **unproven assumptions and
unresolved design decisions** that schema validation can't see:

- **B-1** the structured-output *generation* path was never proven (the spike
  proved rejection, not production; every model tested failed) — the cost-free
  local-dreaming goal is at risk and discovered late.
- **B-3/B-4/B-5** WP06 (drop-vs-strip) and WP08b (independent-episode,
  contradiction-layer-2) have indeterminate behavior — their SCs (4, 5, 6) are
  unverifiable as written.
- **B-2/B-6** process/traceability: SC-number drift in the security WPs, and 7
  WPs labeled `ready` without the grilling the lifecycle requires.

None require re-architecting. **Grill WP06, WP08, and WP12 before execution; run
a generation-quality spike before WP08.** The other WPs can proceed with the
applied fixes + standard review.

---

## Dimension 1 — Organization / structure / decomposition / ordering

**Sound.** WP decomposition is cohesive (one WP = one deliverable). WP01 is the
biggest and blocks everything, but its 4-phase split is the correct mitigation —
each phase is independently gated (phase-2 `tests/integration/applog`, phase-3
`tests/integration/occ`, phase-4 the M0 e2e). Do **not** promote SQLite/OCC to
sibling WPs — that adds a cross-WP edge for no verification benefit (N-5).

WP08 8a/8b/8c/8d independence holds: 8a (no-op worker) needs no LLM; 8c
(orchestrator) is a pure function of a hand-written manifest, schema-coupled to 8b
via the shared `dream-output.ts` (WP01), not runtime-coupled — genuinely
independently testable (N-6). WP12 is correctly a gated parallel track, not a
build-chain phase.

**Defects:** the OVERVIEW dependency graph drifts from the per-WP frontmatter —
missing WP02→WP07 and WP02→WP08 edges (W-1); `engram lifecycle revive`
double-owned by WP08/8d and WP10 (W-2); dream-verb count 4 vs 5 with
`dream.result` dropped in the WP05 OVERVIEW (W-3); WP04→WP02 spurious-looking edge
(W-4). All applied.

## Dimension 2 — Correctness vs spec

Subagent-authored WPs (WP02-04, 05-06, 07/09/10/11) got the hardest scrutiny.
**The math and mechanism claims hold** (m_v 0.42 example, base_m_v/default_conf
tables, deindex→deactivate, ingestEdges=batch-rebuild, scoring-never-imports-
retrieval — all confirmed against the spec). The §4.D invariant is preserved (the
reviewer confirmed recall-calls-retrieval ≠ scoring-imports-retrieval).

**The correctness defects are labeling/contract drift, not logic errors:** SC
off-by-one in WP08+WP09 (B-2, the most dangerous — corrupts the security-test
trail); `forbidden` not a spec'd error code (W-5); `dream.result` omission (W-3);
T-WP04-01 "clamped to 1.0" where the true value 0.971 isn't clamped (Confirmation
caveat → PROPOSE fix). One spec-internal tension: SC-11 crash → TIMED_OUT vs §5.4
dead-PID → FAILED (W-6 / OQ-08).

## Dimension 3 — Hardening gaps

The decisive dimension. **No WP was grilled; OPEN_QUESTIONS/VERIFICATION/TODO were
empty stubs; yet 7 WPs are `ready`** (B-6). Measurability is good for WP01/05 and
the zero-LLM phases (8a, 8c) but fails for WP08b (needs a live model), WP08d
(needs the assembled system), and all of WP12 (human/wall-clock observation) —
those can't be closed by an unattended loop (W-10).

Eight genuine open questions were surfaced and opened (OQ-01..OQ-10, see that
file): drop-vs-strip, independent-episode, contradiction-layer-2, λ_base, entropy
FP rate, the structured-output model, the `forbidden` code, SIGKILL state,
active-pool floor for small stores, and the WP12 14-day evidence source.

**Grill ranking:** WP06 (Tier 1 — security + drop/strip ambiguity), WP08 (Tier 1
— most complex, 3 blocking OQs), WP12 (Tier 1 — live config, unmeasurable gates),
then WP04 (λ_base) and WP07 (Ollama model/adapter) at Tier 2. WP01/02/03/05/09/10/
11 are Tier 3 (execute-as-written with standard oversight).

## Dimension 4 — Risk (ranked by likelihood × blast radius)

1. **B-1 structured-output generation unproven (H×H).** The single
   highest-leverage unknown — invalidates WP07/WP08b + 3 SCs and contradicts the
   cost-free goal; spike proved the opposite (rejection, not production).
2. **W-8 WP12 live-config cutover (M×H).** Stage-5 hook swap is the
   memory-serving handoff; the non-git-tracked legacy store has no *verified*
   rollback; Stage-5 edits a shared hook array.
3. **W-9 Ollama hard-prereq vs WP11 0-skips (M×H).** ~40% of the plan + the final
   gate are un-runnable on a bare CI runner.
4. **late-testable SC-5/7/16 (M×M).** Concentrated discovery risk at the tail;
   graphify `removeNode`/`traverse` round-trip was mapped, not executed (N-8).
5. **WP01 SPOF (L×H).** Well-mitigated by the early M0 loop; residual = schema
   churn → freeze schema contracts as a versioned WP01 exit gate.
6. **serial critical path / WP03 over-depends on WP02 (M×L).** Schedule-only (N-7).

Cross-cutting SPOFs: (1) the structured-output assumption, (2) WP01 schema
contracts, (3) Ollama availability.

## Dimension 5 — Goal-backward (gsd-plan-checker)

**18/18 SCs owned, 0 orphaned, no ambiguous multi-owner.** Every multi-owner SC
(2, 7, 9, 10, 12, 13) has a clear primary home + a genuine contributor. WP11 sits
correctly at the graph end with all `depends_on` edges (04,05,06,07,08,09,10)
satisfied; the graph is acyclic. WP11's own SC→WP map is correct against §12.3
(the drift is confined to WP08/WP09 home tables). One attribution fix: SC-18 →
WP05+WP07 (W-7). Milestones M0/M1/M2 are real and independently demonstrable.

---

## WPs requiring `grill-with-memory` before execution

| WP | Tier | Why |
|----|------|-----|
| WP06 | 1 (must) | Drop-vs-strip (OQ-01) makes SC-4 unverifiable; entropy FP (OQ-05); security-critical kernel. **Demoted `ready → spec`.** |
| WP08 | 1 (must) | 3 blocking OQs (OQ-02 independent-episode, OQ-03 contradiction-layer-2, OQ-06 model); most complex WP; LLM-dependent. |
| WP12 | 1 (must) | Live `~/.claude` config; unmeasurable verify gates (mark manual + human sign-off); OQ-10 evidence source; verified-export rollback gate. |
| WP04 | 2 (should) | OQ-04 `λ_base` missing — required before `engine.ts`. Otherwise deterministic. |
| WP07 | 2 (should) | OQ-06 model; confirm `adapter.py` optional vs required; add the graphify round-trip probe (N-8). |
| WP02 | 3 | Re-state the Spike-1b claim (rejection, not generation); resolve via OQ-06. |
| WP01, WP03, WP05, WP09, WP10, WP11 | 3 | Execute as written with code-reviewer + tdd-guide. |

---

## Fixes applied in this review pass (see FINDINGS for the full list)

- **B-2** SC renumbering corrected in WP08 OVERVIEW + phase-8c + phase-8d + WP09.
- **B-6** OPEN_QUESTIONS populated (10 OQs); TODO re-synced; VERIFICATION build-gate
  row added; **WP06 demoted `ready → spec`** (regression logged below).
- **W-1** OVERVIEW graph: WP02→WP07 and WP02→WP08 edges + note.
- **W-2** `lifecycle revive` assigned to WP10; WP08/8d delegates.
- **W-3** WP05 OVERVIEW + phase-3: 5 dream verbs; "16 verbs" label clarified.
- **W-4** WP04→WP02 type-only rationale note.
- **W-7** WP11 SC-18 delivering-WP → WP05/WP07.
- B-1/B-3/B-4/B-5/W-5/W-6/W-9/W-10 + all NOTEs → opened as OQs / PROPOSE (human-gated).

## Stage regression log

| WP | From | To | Why |
|----|------|----|-----|
| WP06 | ready | spec | B-3/OQ-01: privacy-filter drop-vs-strip behavior undefined; SC-4 unverifiable until resolved. Cannot be `ready` (let alone `hardened`) with a blocking open question. |

## Skill-improvement proposals filed

None. The review tooling (architect-reviewer, plan-reviewer, gsd-plan-checker,
plan-maintain) triggered correctly and was sufficient for the task; no skill
mis-fired, was missing, or was too weak to warrant a `_proposals/` draft.
</content>
