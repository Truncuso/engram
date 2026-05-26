---
name: 2026-05-26-v1-build-gap-report
title: Gap Report — engram v1 — completeness, SPEC-coverage, docs, wiki integration
type: gap-report
plan: 2026-05-26-v1-build
created: 2026-05-26
updated: 2026-05-26
sources: [SRC-01, SRC-02, SRC-03]
---

# Gap Report — engram v1 build plan

Second-pass gap-scan after the 2026-05-26 critical review (`REVIEW_REPORT.md` /
`FINDINGS.md`). Four parallel subagents (Sonnet 4.6), one mandate each:
1. internal plan-WP completeness · 2. SPEC-vs-plan coverage holes · 3. whole-repo
docs/README/ADR · 4. external **LLM-Wiki v2** mining + classification.
Synthesized on Opus.

> Disposition: **[APPLIED]** done this pass · **[WP13]** folded into the new WP13 ·
> **[OQ]** opened in OPEN_QUESTIONS · **[PROPOSE]** left for the human gate ·
> **[NON-ISSUE]** subagent claim verified false.
>
> Output policy (per the human this session): scaffold a real WP only on a clear
> in-scope-for-v1 gap with no owner + a SPEC anchor; everything else proposed.
> SPEC edits require an ADR (CLAUDE.md "spec is living") → proposed, not applied.

---

## Headline outcome

**One genuinely-missing work package found and scaffolded: WP13 — Daemon process
envelope + operational layer.** Three of the four subagents independently
converged on it: the engram *components* are all owned, but the **`engramd`
process itself** — entrypoint, §9.9 startup/shutdown sequencer, PID lock, SIGTERM,
schema-migration guard, structured logger, unified retry, the S-07 store-open
symlink scan, and the operational CLI (`install`/`backup`/`log`/`dream review
--approve`/`agent rotate`) — had **no owning WP**. Every other WP and the WP11
E2E harness *assume* engramd runs; nothing built it. WP13 now owns it (gated on
WP05+WP01, blocks WP11). Plan re-validates **100/100**.

Everything else is fills (applied) or proposals (doc/ADR/SPEC/wiki — human-gated).
The wiki mining confirmed engram's design is a near-superset of LLM-Wiki v2: most
concepts are ALREADY-COVERED; a handful are richer-in-wiki proposals; two
(**RRF**, **last-write-wins**) directly **conflict with engram's locked ADRs** —
do not adopt.

---

## A. New work package (scaffolded)

### WP13 — Daemon process envelope + operational layer  [WP13, stage: spec]

Owns what the component-decomposition skipped. Phases:
- **8 §9.9 startup steps + graceful SIGTERM/SIGUSR2 + PID lock** (`src/daemon/`).
  Resolves the dead-PID→FAILED crash-recovery path (OQ-08 counterpart to WP08-8a).
- **Schema-version start guard + `engram migrate`** (§9.10).
- **Structured logger (§10.3) + unified retry (§9.2) + S-07 store-open symlink
  scan (§8.4)** — three cross-cutting primitives no WP owned.
- **Operational CLI:** `install`/`backup`/`restore`/`log`/`dream review
  --approve` (§5.4 REVIEW_PENDING→MERGED)/`agent rotate`.

Evidence it was a real gap (verified by grep, not just subagent claim):
`engram migrate`, `engram install`, `engram backup`, `dream review`, `engram log`
→ **zero matches** anywhere in the plan; `src/daemon`/`engramd` entrypoint →
**unowned** (only referenced as "engramd is up/down"); S-07 → only write-path
`O_NOFOLLOW` in WP01, no store-open scan.

Follow-ups noted in WP13: WP09's S-07 mitigation-home mapping should be updated
WP01→WP13 during grilling; `backup`/`restore` may be v2-trimmable.

---

## B. Applied fills (this pass)

| # | Fill | Where | Disposition |
|---|------|-------|-------------|
| F1 | SOURCES.md catalog populated (SRC-01/02/03 rows; 30+ live citations had a blank bibliography) | SOURCES.md | [APPLIED] |
| F2 | WP04 T-WP04-01 expected value corrected: `0.85×(0.8/0.7) ≈ 0.971` (≤1.0, no clamp), was wrongly "clamped to 1.0" | WP04 | [APPLIED] |
| F3 | SPEC `SPIKES.md` reference → correct path `docs/research/spikes-2026-05-26.md` | engram-SPEC.md | [APPLIED] |
| F4 | Per-WP OPEN_QUESTIONS seeded with the plan-level OQs that block them (was: empty stubs orphaning the OQs) | WP05/06/08/12 OPEN_QUESTIONS.md | [APPLIED] |
| F5 | WP11 `depends_on: WP13` added; OVERVIEW graph note for WP13 | WP11, OVERVIEW | [APPLIED] |

Subagent claims verified FALSE (not applied):
- **[NON-ISSUE]** "ADR README table missing header separator" — the `|---|` row is
  present; renders fine.
- **[NON-ISSUE]** "AGENTS.md says 'impl plan (once written)'" — actual text reads
  "see the implementation plan"; not stale. (Subagent read a truncated tail.)

---

## C. Proposed — SPEC / plan coverage (human-gated; mostly fold into WP13 or note)

Most are now owned by WP13. Remaining proposals:

| # | Gap | §ref | Proposal | Disposition |
|---|-----|------|----------|-------------|
| C1 | `engram status` full field set (recall p50/p99, staging backlog, dream cost MTD) — WP05-ph4 only does plugins/worker/store/version | §10.3 | extend WP05-ph4 `SystemStatus`; add latency-recording step to WP04; cost accumulator to WP08-8d | [PROPOSE] |
| C2 | Dream run audit JSON `agent-reports/drun_<id>.json` (20+ mandatory fields) + 80% budget WARN — distinct from the manifest | §10.4 | add `src/core/orchestrator/audit.ts` step to WP08-8c; 80% WARN to WP08-8a budget.ts | [PROPOSE] |
| C3 | `memory.ingest` `unsupported-type` error path (binary/zip/exe) | §6.2/§6.3 | add early MIME/ext check + `unsupported-type` to WP07 ingest.ts | [PROPOSE] |
| C4 | Weekly full `engram doctor` schedule (vs abbreviated-on-startup, owned) | §9.8 | add scheduler registration to WP13-ph1 (daemon owns the interval) or WP01-ph4 | [PROPOSE] |
| C5 | `λ_base` per-type decay constants must exist before WP04-ph3 compiles | §3.1/§3.6 | OQ-04 (already open); add constants to WP04 `engine.ts`/`constants.ts` on resolution | [OQ] |

The daemon-entrypoint, migrate, logger, retry, S-07 scan, `engram log`, `dream
review --approve`, `agent rotate`, `install`, `backup` items the SPEC-coverage
agent flagged (F-2,F-3,F-4,F-6,F-7,F-10 + W-C2,W-C3) → **[WP13]** (owned).

---

## D. Proposed — documentation / ADRs (human-gated)

| # | Gap | Where | Proposal | Disposition |
|---|-----|-------|----------|-------------|
| D1 | README has no user/operator install/usage — only a dev-build section, for "a standalone installable application" | README.md | add Prerequisites → Install → Quick-start (`engram init/status/doctor`, daemon start, bearer setup) → Security note | [PROPOSE] |
| D2 | ~1.28 GB QMD GGUF download not in README (WP03 promises "document in setup") | README.md | add to Prerequisites: first-index downloads ~1.28GB to `~/.cache/qmd/models`; BM25-only works without | [PROPOSE] |
| D3 | Ollama hard prereq (WP07+) not in README | README.md | add to Prerequisites: Ollama required for `memory.ingest`/graph extraction; `ollama serve` + a structured-output-capable model (see OQ-06) | [PROPOSE] |
| D4 | No ADR for MCP transport + bearer (Streamable HTTP/127.0.0.1/`requireBearerAuth`; rejected stdio + unix-socket) — a locked security decision | docs/adr/ | write **ADR-0005: MCP transport — Streamable HTTP on loopback with SDK bearer auth** | [PROPOSE] |
| D5 | No ADR for CaptureIntake-in-kernel / privacy-filter-must-not-be-a-plugin invariant (stated in SPEC §2.2 + architecture rule, no ADR) | docs/adr/ | write **ADR-0006: CaptureIntake is fixed core — privacy filter is not a plugin** | [PROPOSE] |
| D6 | SPEC frontmatter references 5 `docs/review/*` + 4 `docs/research/*` docs that don't exist (only spikes-2026-05-26.md present); §0 actively directs readers to missing `SYNTHESIS.md` | engram-SPEC.md | either backfill, or add a §0 note that these were internal/ephemeral review inputs not committed; summarize D-1..D-7/R-1..R-5 inline so §0 is self-contained | [PROPOSE] |
| D7 | README/AGENTS "16 verbs" vs §6.3 17-row table (the spec-header nit from FINDINGS N-9) | SPEC §6.3, README, AGENTS | patch §6.3 header to "16 verbs + 1 status resource"; update README/AGENTS to match | [PROPOSE] |
| D8 | SPEC §14 Round-3 verification recommendation never closed | engram-SPEC.md §14 | add a one-line note: satisfied by the 2026-05-26 multi-agent review + 4 spikes, or explicitly waived | [PROPOSE] |
| D9 | No CONTRIBUTING / dev-setup doc (expected by WP12 cutover / external contributors) | repo root | add CONTRIBUTING.md before WP12 go-live (env vars, test suite, plugin-add guide) | [PROPOSE, post-v1] |

---

## E. Wiki integration — LLM-Wiki v2 classification

engram's SPEC is a **near-superset** of the wiki. Full classification (22
concepts) below; only the non-COVERED rows need action.

### E.1 ADR-CONFLICTS — do NOT adopt (the wiki contradicts engram's locked design)

| Wiki claim | engram locked decision | Verdict |
|------------|------------------------|---------|
| "Fuse hybrid results with **reciprocal rank fusion (RRF)**" | §4.D + §11.2 + architecture rule **explicitly reject RRF**: single ranked source (QMD); graph is opt-in *expansion*, not a competing rank; scoring never imports the retrieval plugin | **ADR-CONFLICT — do not adopt.** RRF presupposes independent ranked lists engram intentionally collapsed; adopting it would break the §4.D invariant. |
| "**Last-write-wins** multi-agent merge, timestamp resolution" | §7.2 OCC + advisory lock + three-way field merge; §5.4 base-version mismatch → review queue (never LWW); multi-machine is §8.6 OUT-of-v1 | **ADR-CONFLICT + V2-DEFERRED — do not adopt.** LWW silently loses writes — incompatible with §10.1 "no silent mutation" + per-field AppLog provenance. |

### E.2 GENUINE-GAP-v1 (in scope, not covered) — narrow

| Wiki concept | Proposal | Disposition |
|--------------|----------|-------------|
| Output formats beyond Markdown / structured **export** | narrow; add a read-only `memory.export` verb (JSON/NDJSON filtered set) to WP05 or WP10 — serves bulk-export-before-purge. NOT a new WP. | [PROPOSE] |
| "Schema is the product" — user-maintained domain taxonomy/policy the worker reads | extend `.dreaming/<name>.md` (§5.1) with an optional free-text `instructions:` block prepended to the worker distill prompt (one schema field + WP08b prompt change) | [PROPOSE] |

### E.3 PARTIAL-richer-in-wiki — SPEC gestures at it; wiki sharper (all [PROPOSE], need ADR/SPEC gate)

| Wiki concept | engram today | Proposed one-line SPEC/WP addition |
|--------------|--------------|-----------------------------------|
| Supersession (typed) | lifecycle `dormant` + `derived_from` only; `contradicts` is v2 | add optional `superseded_by: <mem_id>` frontmatter (§3.4) written by worker on replace; doctor checks referent — no new edge kind |
| Ebbinghaus reset-on-reinforcement | mechanically true (access touch resets `last_used`) but not documented; `confirm` doesn't reset recency | §3.6 one sentence: access touch / `memory.confirm` / re-weight resets `last_used` (restarts decay); `confirm` should touch stats sidecar |
| Citation-coverage quality scoring | `confidence`/`verification_state`/`m_v` (trust), not citation coverage | §5.2 step 1: distilled semantic memory with empty `sources:` + `origin: self-authored` → `confidence: 0.3` pending corroboration |
| Contradiction *resolution* proposal | detect + queue_review; "never auto-resolve" | §5.3 manifest: contradiction entry gets optional `proposal?: {preferred_id, rationale, confidence}` for human review; never auto-applied |
| Merge-duplicate-entities | emergent-entity *create* only | §9.8 doctor check: `Duplicate semantic memories` (sim>0.95, same type, both active) → INFO; `--fix` proposes merge (never auto) |
| Crystallization (explicit digest of a work-chain) | implicit via distill pipeline | `dream.trigger {mode: "crystallize", sessionId}` → one episodic digest + N semantic extracts; WP08b adds a `crystallize` mode |

> All E.3 items are documentation-or-additive only; each needs your gate and (for
> SPEC changes) an ADR per CLAUDE.md "spec is a living document." None applied.

### E.4 ALREADY-COVERED (the bulk — cited for the record)

Confidence scoring §3.4/§3.6 · consolidation tiers (4 types) §3.1 · knowledge
graph §2.3/§6.2 · entity extraction §5.2/§6.2 · hybrid search (vector+BM25, graph
expansion) §4.D/§9.1 · event-driven hooks §5.6/§6.1 · self-healing lint (`doctor`/
`repair`) §9.8/§10.2 · contradiction *detection* §5.2 · shared/private scoping
§7.1 · privacy filter on ingest §6.1/§8.3 · audit trail (AppLog) §7.3/§10.2 ·
governance bulk ops §8.5/§10.2 · typed relationships (v1: `derived_from`/
`related_to`; rest v2) §3.4. Quality scoring & supersession are PARTIAL (E.3).

---

## F. Summary

- **1 WP scaffolded** (WP13, validated, plan still 100/100).
- **5 fills applied**; 2 subagent claims found false and not applied.
- **~20 items proposed** (SPEC-coverage C1–C5, docs/ADR D1–D9, wiki E.1–E.3) for
  your gate.
- **2 ADR-conflicts flagged** (RRF, LWW) — explicit do-not-adopt.
- New open questions already in OPEN_QUESTIONS (OQ-04..OQ-10 from review +
  these reinforce OQ-06/08).

**Recommended next:** (1) gate WP13 spec→hardened alongside WP06/WP08/WP12 in the
grilling pass; (2) decide D4/D5 (two ADRs) and D1–D3 (README) — they're
load-bearing for an installable app; (3) the wiki E.3 items are low-risk SPEC
enrichments, each needing a one-line ADR.
</content>
