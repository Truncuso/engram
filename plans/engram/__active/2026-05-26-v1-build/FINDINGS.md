---
name: 2026-05-26-v1-build-findings
title: Findings — engram v1 — bottom-up build
type: findings
plan: 2026-05-26-v1-build
created: 2026-05-26
updated: 2026-05-26
sources: [SRC-01, SRC-02, SRC-03]
---
<!-- Template: FINDINGS v2 (frontmatter-first) -->

# Detailed Findings — engram v1 — bottom-up build

**Analysis Method**: Multi-agent parallel review (architect-reviewer ×2,
plan-reviewer ×2, gsd-plan-checker) synthesized against `docs/engram-SPEC.md`
v2.1 and `docs/research/spikes-2026-05-26.md`. Full per-dimension report in
`REVIEW_REPORT.md`.

> Severity: **BLOCK** = wrong/contradicts spec, untestable, or makes a goal
> unreachable → fix before execution. **WARN** = inaccuracy that will mislead the
> executor → should fix. **NOTE** = minor/clarity → optional, human-gated.
>
> Disposition: **[APPLIED]** fixed in this review pass · **[OQ]** opened as an
> open question (human must resolve) · **[PROPOSE]** NOTE-level, left for the human.
>
> Also the log for stage regressions: when a WP moves backward, record why here.

## Corrections Summary

| Original Claim | Corrected Finding |
|----------------|-------------------|
| "Spike-1b confirmed the structured-output path" (WP02/WP08b) | Spike-1b confirmed only the SDK **rejection** mechanism; **no model (local or cloud) was shown to *produce* schema-valid output** — every tested model failed (spikes §1b). Generation capability is unproven. |
| WP08/WP09 SC tags (SC-11/12/13) | Off-by-one vs §12.3: prompt-injection=SC-12, dream.trigger-scope=SC-13, active-pool-floor=SC-14. WP11 numbering is correct; WP08/WP09 drift. |
| WP05 OVERVIEW "dream.* (4 verbs)" | §6.3 has **5** dream verbs (`dream.result` omitted in the OVERVIEW target-file line). |
| `engram lifecycle revive` | Double-owned: implemented in both WP08/8d and WP10. One home only (WP10). |

---

## BLOCK (fix before execution)

### B-1 — Structured-output *generation* never proven; "cost-free local dreaming" at risk
**Where:** WP02 (Verified Evidence lines 54–56), WP08/phase-8b (`verify:` "with a
structured-output model"), spikes §1b. Cascades to SC-3, SC-5, SC-6.
**Issue:** The plan treats "Spike-1b confirmed" as validating the structured-output
path. The spike confirms only that the Vercel AI SDK *rejects* bad output. Every
model actually tested (cloud-routed Ollama proxies `deepseek-v3.2:cloud` etc.)
**failed** schema compliance; OpenAI failed on billing; no Anthropic key was
present. **No model — local or cloud — was demonstrated producing schema-valid
structured output.** The spike's own operational note says engram must *pin a
structured-output-capable model per provider* — which no WP does. If no local
Ollama model clears the bar, the stated goal of cost-free local dreaming (SPEC
§6.2) collapses to a metered cloud dependency, discovered late at WP08b.
**Disposition:** [OQ] OQ-06 + risk note added to WP02/WP07/WP08. Fix = run a
generation-quality spike (pin a model, run `generateObject` against
`dream-output.schema.json` over N seeded inputs, require a measured pass-rate
threshold) **before WP08 starts**; record the exact model id in SPEC + WP02.

### B-2 — SC number drift in WP08 + WP09 (off-by-one) corrupts the security-test trail
**Where:** WP08/OVERVIEW (W08-5/6/8), WP08/phase-8c + phase-8d `verify:` fields,
WP09 specific-tests table. Confirmed independently by 3 reviewers.
**Issue:** §12.3 order is SC-11=worker-crash, SC-12=prompt-injection, SC-13=
dream.trigger-scope, SC-14=active-pool-floor. WP08/WP09 tag active-pool as SC-13,
prompt-injection as SC-11, scope-denied as SC-12 — each shifted one down. Affects
3 of the 6 CRITICAL-mitigation rows. An executor closing phase-8c/8d against the
wrong SC label produces a broken cross-reference with WP11 (whose numbering is
correct) and the §12.3 gate.
**Disposition:** [APPLIED] SC tags corrected in WP08 OVERVIEW, phase-8c, phase-8d,
and WP09.

### B-3 — WP06 privacy filter: drop-whole-observation vs strip-secret is undefined
**Where:** WP06/OVERVIEW (W06-1), WP06/phase-1 (`verify:`), §6.1, §12.3 SC-4.
**Issue:** WP06 says both "strips it" and "observation dropped (fail-closed)";
phase-1 verify says the payload is "fully dropped (not written to staging)". But
SC-4 requires the secret to be *stripped* and the surrounding observation to
*reach staging* (the test asserts the key is absent from a JSONL line that
exists). Drop-whole vs strip-and-pass are different behaviors with different SC-4
assertions; only strip-and-pass satisfies SC-4. "Fail-closed" plausibly governs
filter *errors* (exception/timeout → drop), not successful *matches* — but the
plan conflates them. Until resolved, SC-4 is unverifiable and WP06's test is
likely wrong-but-green.
**Disposition:** [OQ] OQ-01 opened; WP06 demoted `ready → spec` (see regression
log). Resolve before WP06/phase-1.

### B-4 — WP08b: "independent episode" (counterfactual gate) has no computable definition
**Where:** WP08/phase-8b; §5.2 step 4; SC-5.
**Issue:** The counterfactual gate promotes a procedural memory only if
"corroborated by ≥1 independent episode," but "independent" is never defined
algorithmically (different session? agent? day? not-same-source?). Without a
predicate the gate can't be implemented deterministically and SC-5's second-run
promotion has no pass/fail condition.
**Disposition:** [OQ] OQ-02 opened. Resolve before WP08/phase-8b.

### B-5 — WP08b: two-layer contradiction detection — second layer undefined
**Where:** WP08/phase-8b ("2-layer contradiction detect"); §5.2 step 2; SC-6.
**Issue:** §5.2 names "semantic (`sim>0.85 ∧ |Δimportance|>0.3`) + graph traversal."
The semantic layer is numerically specified; the graph-traversal layer is a
description with no computable definition (what graph relation constitutes a
contradiction?). `src/worker/connect.ts` has no complete spec and SC-6 has no
deterministic oracle.
**Disposition:** [OQ] OQ-03 opened. Resolve before WP08/phase-8b.

### B-6 — Plan-root stubs unfilled while 7 WPs are `stage: ready` (lifecycle violation)
**Where:** plan-root OPEN_QUESTIONS.md / VERIFICATION.md / TODO.md (all stubs).
**Issue:** `spec → hardened` requires a grilling pass and zero open questions; no
WP was grilled, yet WP02/03/04/05/06/09/11 carry `stage: ready`. Cross-WP
questions (OQ-01 spans WP06↔SC-4; OQ-06 spans WP02/07/08/11) have no home;
VERIFICATION.md defines no cross-WP build gate; TODO.md shows every WP as `draft`
(stale). `ready` is premature for any ungrilled WP and outright wrong for WP06.
**Disposition:** [APPLIED] OPEN_QUESTIONS.md populated (8 OQs); TODO.md re-synced;
VERIFICATION.md build-gate row added; WP06 demoted to `spec`. The remaining `ready`
labels are flagged for the human gate (see Verdict in the review summary) — this
review does not auto-promote to `hardened`.

---

## WARN (should fix)

### W-1 — OVERVIEW dependency graph omits WP02 → WP07 and WP02 → WP08 edges
**Where:** OVERVIEW.md graph (lines ~89–95) vs WP07/WP08 frontmatter.
**Issue:** Frontmatter correctly declares WP07 `depends_on: WP02, WP05` and WP08
`depends_on: WP06, WP07, WP02`, but the ASCII graph draws WP07/WP08 hanging off
WP05 only. An agent scheduling off the picture could start WP07/WP08 without the
LLM/plugin host. **Disposition:** [APPLIED] edges added to the graph + note.

### W-2 — `engram lifecycle revive` double-owned (WP08/8d and WP10)
**Where:** WP08/phase-8d step 4 (`src/cli/commands/lifecycle.ts`) and WP10 step 10
(`src/cli/commands/governance.ts`). **Issue:** Same CLI verb claimed by two WPs in
two files → merge collision / duplicate work. **Disposition:** [APPLIED] assigned
to WP10 (governance/lifecycle module); WP08/8d step changed to delegate to WP10.

### W-3 — dream-verb count inconsistent (4 vs 5); `dream.result` dropped in WP05 OVERVIEW
**Where:** WP05/OVERVIEW target-files line ("4 verbs"), WP05/phase-3 prose ("4
verbs … 5 total"), WP08/phase-8d ("five dream.* verbs"). **Issue:** §6.3 has 5
dream verbs; OVERVIEW omits `dream.result`, risking an incomplete `dream.ts`.
**Disposition:** [APPLIED] WP05 OVERVIEW + phase-3 corrected to 5 dream verbs;
the "16 verbs" label clarified (= 11 memory + 5 dream; `system.status` is the +1
tool/resource, per SC-17 "16 verbs + status resource").

### W-4 — WP04 declares `depends_on: WP02` but never calls the LLM plugin
**Where:** WP04 frontmatter. **Issue:** WP04 (scoring/recall) needs WP03 + WP01
only; the WP02 coupling is transitive via WP03 and reads as a scoring↔LLM
coupling that the §4.D invariant forbids. **Disposition:** [APPLIED] rationale
note added ("PluginLifecycle/RetrievalPlugin types originate in WP02; scoring does
not call LlmPlugin"); edge retained as type-only to avoid breaking the validator's
edge graph.

### W-5 — `forbidden` error code (WP05/phase-1) is not a spec'd MCP error
**Where:** WP05/phase-1 `verify:` + steps. **Issue:** §6.3/§7.1 define
`scope-denied` for capability/scope failures; `forbidden` is invented. Tests
asserting `forbidden` diverge from the spec'd surface. **Disposition:** [OQ] OQ-07
opened (use `scope-denied` for both, or amend spec to add `forbidden`).

### W-6 — SC-11 crash → `TIMED_OUT` vs `FAILED`: spec-internal tension
**Where:** WP11 step-14 (SC-11), §5.4 rules 1–2, §12.3 SC-11 text. **Issue:** SC-11
text mandates `TIMED_OUT` for a worker crash and WP11 follows it — correct against
the SC. But §5.4 routes dead-PID → `FAILED` (crash-recovery scan) and stale-
heartbeat → `TIMED_OUT`; a SIGKILL'd worker has a dead PID, so which transition
wins is ambiguous, and the `FAILED` (dead-PID) path has no dedicated test.
**Disposition:** [OQ] OQ-08 opened (spec-boundary); [PROPOSE] add a distinct
WP08-8a test for the dead-PID → FAILED crash-recovery path.

### W-7 — SC-18 attributed to WP05 alone, but graphify health isn't live until WP07
**Where:** WP11 SC-18 row (Delivering WP = WP05). **Issue:** SC-18 asserts
`plugins[{qmd,graphify,llm}].health.ok===true`; graphify plugin ships in WP07.
WP11 already `depends_on: WP07` so the final gate is safe, but the column should
read WP05+WP07 so no one tries to green SC-18 at M2. **Disposition:** [APPLIED]
WP11 SC-18 delivering-WP corrected to WP05/WP07; M2 note clarified ("16 verbs
reachable", not "SC-18 green").

### W-8 — WP12 cutover: non-git-tracked legacy store has no verified rollback
**Where:** WP12/phase-3 (`~/.claude/projects/*/memory/` "no git rollback — export
first"). **Issue:** "export first" is a bare TODO with no verification the export
is restorable before the irreversible Stage-7 delete; blast radius = the user's
live cross-project agent memory. **Disposition:** [PROPOSE] make the export a
*verified* gate (export → assert restorable via dry-run/checksum → only then
delete); treat Stage-5 shared-hook-array edit as a code-reviewer BLOCK gate.

### W-9 — Ollama hard-prereq vs WP11 "0 skips" gate (CI un-runnable)
**Where:** WP07 ("Ollama hard prerequisite"), WP11 build-gate ("0 skips"), WP02
T-WP02-06 ("skip if no Ollama"). **Issue:** WP07/WP08b/WP10 + SC-3/5/6/16 need a
running Ollama + capable model, but WP11 forbids skips — a bare CI runner cannot
pass the final gate. **Disposition:** [OQ] OQ-06 (model) + [PROPOSE] reconcile
WP11's no-skip rule with an explicit CI strategy (self-hosted Ollama runner, or
recorded `graph.json` fixtures + cloud-model fallback for LLM tests).

### W-10 — WP08b/8d `verify:` not autonomously loopable; WP12 verify is human-only
**Where:** WP08/phase-8b (needs live model), phase-8d (needs full 8a+8b+8c system),
WP12 all phases ("session loads context", "14-day no-access"). **Issue:** Several
`verify:` fields require a live model, the assembled system, or human/wall-clock
observation — they cannot be closed by an unattended loop. **Disposition:**
[PROPOSE] mark WP12 explicitly manual-execution + human-sign-off; add to 8d verify
"requires 8a+8b+8c passed"; resolve 8b's model via OQ-06.

---

## NOTE (optional / human-gated)

- **N-1** WP06/phase-1 entropy threshold (40 chars) has no false-positive analysis
  on normal payloads (base64, stack traces, SQL, minified JS) → [OQ] OQ-05.
- **N-2** WP04 `λ_base` recency-decay constant is never given a value anywhere
  (spec, plan, spikes) → [OQ] OQ-04. Required before WP04 `engine.ts`.
- **N-3** Active-pool floor `min(100, 20% total)` yields a tiny floor for small/new
  stores (e.g. 16 for an 80-memory store; ~1 for a 5-memory store) → [OQ] OQ-09:
  is a bootstrap minimum intended?
- **N-4** WP12/phase-3 "14-day no-access" gate is measured against engram's AppLog,
  which never logs old-system QMD reads → [OQ] OQ-10: define the real evidence
  source (filesystem atime / qmd active-collections / hook-swap-as-evidence).
- **N-5** WP01 is large but the 4-phase split is the correct mitigation (each phase
  independently gated); **do not** promote SQLite/OCC to sibling WPs — it adds a
  cross-WP edge for no verification benefit. Add a one-line note that `jobs.db`/
  `manifest`/`dream-output` schemas are produced-here, consumed-later (WP07/WP08).
- **N-6** WP08 8a/8c independence holds; 8c is schema-coupled (not runtime-coupled)
  to 8b via `src/schemas/dream-output.ts` (WP01). Add a WP08-OVERVIEW note that 8b
  and 8c both bind to the shared schema and neither redefines the manifest shape.
- **N-7** Critical path WP01→02→03→04→05 is long/serial before M2; WP03 over-depends
  on WP02 (needs only the interface types). Optional: split the plugin-host
  *contract* out of WP02 so WP03 can build against it in parallel. Schedule-only.
- **N-8** graphify `traverse`/`removeNode` round-trip was *mapped, not executed*
  (spikes §3). Add a thin WP07-early probe (one live `get_neighbors` round-trip +
  one extract→filter→rebuild cycle) before building the worker on it.
- **N-9** §6.3 spec header says "16 verbs" but the table lists 17 rows (incl.
  `system.status`). Spec-level inconsistency the plan inherits consistently;
  recommend a one-word spec fix ("16 verbs + 1 status resource/tool") via ADR or
  spec append. Not a plan defect.

---

## Confirmations (verified correct — do not change)

- m_v worked example `self-authored, conf 0.3 → 0.7·(0.3/0.5)=0.42` ✓ (§3.6).
- WP04 base_m_v table (human 1.0 / ingested 0.9 / agent-session 0.85 /
  self-authored 0.7) and default_confidence (0.90/0.80/0.70/0.50) ✓ (§3.6).
  Caveat W-3-adjacent: T-WP04-01 says agent-session conf 0.8 → m_v "clamped to
  1.0" but `0.85·(0.8/0.7)=0.971 ≤ 1.0` → **no clamp**; assertion should be ~0.971,
  not 1.0. → [PROPOSE] fix the T-WP04-01 expected value.
- WP03 `deindex → store.deactivate` (soft-delete) ✓ (§2.3 non-fatal, Spike 1).
- WP07 `ingestEdges = batch graphify update` (derived index, not live write) ✓
  (§2.3, §6.2, ADR-0004, Spike 3).
- "Scoring never imports retrieval plugin": recall (`degradation.ts`) calls
  `RetrievalPlugin.search`; scoring (`engine.ts`) receives pre-fetched hits — the
  §4.D invariant is preserved (not a contradiction) ✓.
- WP09 S-10 budget test asserts `ABORTED` (budget) — correctly distinct from SC-11
  `TIMED_OUT` (heartbeat) ✓ (§5.4); see W-6 for the residual dead-PID nuance.
- Goal-backward coverage: **18/18 SCs owned, 0 orphaned**; all multi-owner SCs
  (2,7,9,10,12,13) have a clear primary home + a genuine contributor ✓.
- Milestones M0 (WP01 grep-recall) / M1 (WP04 scored recall) / M2 (WP05 MCP) are
  real, automated-test-gated, independently demonstrable checkpoints ✓.
- Spikes 1 (QMD in-proc), 2 (MCP HTTP+bearer+subscriptions live), 3 (graphify
  mapping) are decisive for *transport/mechanism* — the de-risking is genuinely
  strong there. The unproven leg is LLM *generation* quality (B-1).
</content>
</invoke>
