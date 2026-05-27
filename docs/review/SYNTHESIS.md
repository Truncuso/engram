---
title: "engram — Review Synthesis (Round 1)"
project: engram
created: 2026-05-22
inputs:
  - docs/review/security-review.md (security-reviewer)
  - docs/review/architecture-review.md (architect-reviewer)
  - docs/review/failure-safety-review.md (debugger)
  - docs/review/observability-review.md (devops-engineer)
status: synthesis — Round 1 + Round 2 complete; author decisions pending on R-1..R-5
round2_inputs:
  - docs/research/qmd-graphify-verification.md
  - docs/research/agentmemory-patterns.md
  - docs/research/mcp-agentsdk-contract.md
  - docs/research/memory-reliability-literature.md
---

# Review Synthesis — Round 1

Four parallel specialist reviews of `SPEC.md` (DRAFT 2026-05-22). This document
consolidates them: what they agree on, where they overlap, the cross-cutting
themes, the conflicts, and the decisions that need the author's call before the
spec is rewritten.

**Headline.** The macro-architecture is sound — every reviewer affirmed the
microkernel + detached-dreaming-worker shape and the four plugin seams. The spec
is *strong on vision, weak on contracts and on the non-functional dimensions*
(security, failure, observability). None of the findings require redesign; they
require **finishing the spec** — four missing contracts and three missing
cross-cutting sections.

---

## 1. The four reviews at a glance

| Review | Verdict | CRITICALs | Biggest contribution |
|--------|---------|-----------|----------------------|
| **Security** | needs a threat model + 8 mitigations | 6 | A full threat model + the staging-integrity / agent-identity / prompt-injection mitigations |
| **Architecture** | sound shape, unfinished contracts | 7 | Wrote all 4 missing contracts verbatim (data-flow, plugin, MCP, job-queue); resolved 6 OQs |
| **Failure/Safety** | reliability undefined | (12 gaps) | A complete §13 Failure & Safety section (recall chain, retry policy, job state machine, forgetting rails) |
| **Observability** | underspecified | (12 gaps) | A complete §OT section (app-log schema, metrics, dream-run audit, ops/systemd) |

---

## 2. Strong convergence — where multiple reviewers independently agree

These are high-confidence: more than one reviewer arrived at them separately.

### C-1. The job queue + dream state machine is the spine (all 4 touch it)
Architecture (F-5.1), Failure (§3), and Observability (OT.4) all independently
specify a SQLite `jobs` table with a state machine, heartbeat, and a reaper.
Security (S-09/S-11) constrains what the worker may do. **This is the single
most-requested addition.** The three state-machine proposals differ only in
naming; they must be unified into one canonical machine.

### C-2. The dreaming worker / orchestrator labour split must be explicit
Architecture (F-4.3): orchestrator owns queue+schedule+merge, **no LLM**; worker
owns **all LLM** + branch + manifest, no merge. Security (S-10, S-11) requires
the *daemon* (orchestrator) — not the worker — to enforce budget and the
safe/gated classification. Failure (§3) puts merge-validation in the
orchestrator. **Unanimous: the worker proposes (branch + manifest); the
orchestrator disposes (validates, merges, gates).**

### C-3. "Safe vs gated" merge must be deterministic, not LLM-decided
Security (S-11) and Architecture (F-1.6) agree: the worker emits a structured
**manifest**; the orchestrator classifies each change with a **deterministic
predicate** over the diff (new file = safe; lifecycle/visibility/importance
change = gated; delete = gated). The LLM must never self-classify its output as
safe. Failure (§8) adds rate-limits on top.

### C-4. Files are truth → everything else is derived and reconciled on startup
Architecture (F-5.4) and Failure (§6) agree on the same invariant and the same
write ordering: **(1) atomic file write (tmp→rename), (2) app-log append,
(3) git commit** — with the file authoritative and app-log/git/index rebuildable
and reconciled at startup via recovery spools.

### C-5. recall must never be a versioned write
Architecture (F-1.7): `recency`/`access_count` touch-on-recall must go to a
non-versioned **stats sidecar** (`.engram/stats.db`), never the memory file +
git + app-log. Security (S-18) independently flags the same fields as a
ranking-manipulation surface and also says the **daemon** must own them, not the
agent. Converges: scoring-stat fields are daemon-managed, sidecar-stored.

### C-6. Capture is fire-and-forget, fail-closed, decoupled by staging
Failure (§4) and Security (S-01/S-02) agree: hooks write **directly to
`staging/`** (no daemon RPC at capture time), 200ms hard timeout, exit-0 on any
error, and the **privacy filter fails closed** (drop the observation rather than
store an unfiltered one). Architecture (F-2.4) adds that the `CapturePlugin`
interface was modelling the wrong thing (a callback can't fire cross-process).

### C-7. Provenance needs more than one `author` field
Observability (OT.5) and Security (S-17) agree the single `author` is
insufficient: split into `created_by` (immutable) + `last_modified_by`, with the
**full history in an app-log** that is integrity-protected (hash-chained per
Security S-17). Add `ingest_run_id` / `staged_from` provenance links.

---

## 3. Cross-cutting themes

### T-1. The spec is a narrative; it must become a contract
Every reviewer said a version of this. The four supplied contracts
(data-flow Flows A–E, the plugin contract, the 16-verb MCP contract, the
job-queue schema) plus the two supplied sections (§13 Failure, §OT
Observability) and the threat model are **ready to lift in nearly verbatim**.
The spec rewrite is mostly *assembly*, not invention.

### T-2. The detached worker is the security AND reliability AND observability hinge
The same architectural choice the user insisted on (process-isolated dreaming)
is what makes budget-DoS containment (S-10), crash recovery (Failure §3), and
dream-run auditing (OT.4) tractable. The reviews reinforce the B+C decision.

### T-3. v1 is over-scoped in three specific places
Architecture (F-7.1/7.3) flags: **6 relation kinds → 2 for v1** (`related_to`,
`derived_from`); **cross-agent dreaming → v2** (ship per-agent only);
`confidence` stored-not-consumed. Trimming these reduces v1 branching in scoring,
graph, and access-control code. (Note: this slightly walks back the earlier
grilling answer that cross-agent dreaming is a v1 feature — see Decision D-3.)

### T-4. "Local-first single-user" is the security boundary — and it has a hole
Security's threat model is honest: on a single-user machine, **any process as
that user can read the store files directly**; engram's access control is at the
MCP layer, not the filesystem. `hidden` memory therefore needs filesystem
permissions / encryption to mean anything (S-14). This must be stated plainly,
not implied.

---

## 4. Conflicts & tensions to resolve

### X-1. Health/metrics endpoint vs "no network surface in v1"
Observability (OT.3) proposes an HTTP `/health` + `/metrics` endpoint on
`127.0.0.1:7474`. Security's threat model lists "no network-facing surface in
v1" as an out-of-scope assumption and flags the v2 dashboard's HTTP server
(S-15) as needing localhost-binding + a session token. **Tension:** an HTTP
metrics port *is* a (local) network surface. Resolution options in D-5.

### X-2. App-log: SQLite vs hash-chained JSONL
Security (S-17) wants the app-log **hash-chained and append-only at the fd
level** for tamper-evidence (NDJSON with prev-hash chaining). Observability
(OT.2, OQ-K) wants it in **SQLite** to avoid full scans and to match the
job-queue decision. These pull opposite ways (SQLite is not naturally
append-only/hash-chained). Resolution in D-4.

### X-3. Cross-agent dreaming: v1 or v2?
The grilling session landed cross-agent dreaming as a v1 capability (the named
"dreaming memory" with multiple scopes). Architecture (F-7.3) and Security
(S-12) both argue it is v2: it needs the `cross-agent` capability, a cross-scope
visibility invariant, and confused-deputy protection (S-13). Resolution in D-3.

### X-4. Staging granularity: per-observation files vs per-session append log
Architecture (F-1.2) strongly recommends **per-session JSONL append logs**
(avoids tens of thousands of inodes; gives a clean truncate-on-merge watermark).
Failure (§4) describes **per-observation atomic files** (tmp→rename, collision-
safe names). Both are defensible; they must be reconciled into one model. (Lean:
per-session append log with atomic appends — fewer inodes, natural watermark.)

---

## 5. Decisions needed from the author (D-1 … D-7)

These are the calls the rewrite depends on. Most have a clear lean from the
reviews; a few are genuine product choices.

| ID | Decision | Lean from reviews |
|----|----------|-------------------|
| **D-1** | Adopt the architecture review's 4 contracts (data-flow, plugin, MCP-16-verb, job-queue) wholesale into the spec? | **Yes** — they're complete and self-consistent |
| **D-2** | Adopt §13 Failure & §OT Observability sections wholesale? | **Yes** — with X-1/X-2 reconciled |
| **D-3** | Cross-agent dreaming: v1 or v2? | **v2** (ship per-agent only in v1); keep the schema field |
| **D-4** | App-log storage: SQLite, hash-chained JSONL, or both? | **DECIDED: SQLite-only for v1.** Tamper-evidence (hash-chaining) deferred to v2 — acceptable under the single-user trust model. |
| **D-5** | Metrics/health: HTTP port, or CLI-only (`engram status` reads a unix socket)? | **CLI + unix socket** for v1 (no TCP port); HTTP `/metrics` as v2 with the dashboard. Avoids X-1. |
| **D-6** | Adopt the full threat model section + the 6 security CRITICAL mitigations as spec requirements? | **Yes** — they gate the relevant implementation phases |
| **D-7** | Trim v1 scope per T-3 (2 relation kinds, confidence stored-not-consumed)? | **Yes** |

### Author decisions recorded (2026-05-22)

- **D-1 = YES** — adopt the 4 contracts wholesale.
- **D-2 = YES** — adopt §13 Failure & §OT Observability, with X-1/X-2 reconciled.
- **D-3 = DEFER TO v2** — v1 ships per-agent-isolated dreaming only; the
  `mode: cross-agent` schema field is retained but unimplemented in v1. This
  removes the cross-scope visibility invariant, the `cross-agent` capability,
  and the confused-deputy surface (S-12/S-13) from v1.
- **D-4 = SQLite-ONLY (v1)** — app-log is SQLite for queryability; hash-chained
  tamper-evidence (S-17) is a v2 hardening, acceptable under the single-user
  trust model. Resolves conflict X-2 in favour of SQLite.
- **D-5 = CLI + unix socket (v1)** — applied.
- **D-6 = YES** — threat model + 6 CRITICAL mitigations are spec requirements.
- **D-7 = YES** — v1 trims applied (2 relation kinds, confidence stored-not-consumed).

---

## 6. New open questions surfaced (append to spec §10)

From the reviews, beyond the original OQ-A…OQ-J (most now resolved):

| ID | Question | Source | Lean |
|----|----------|--------|------|
| OQ-K | App-log format SQLite vs JSONL | Obs | see D-4 |
| OQ-L | Is app-log git-tracked? | Obs | yes, default |
| OQ-M | Health surface: port vs socket | Obs/Sec | socket (D-5) |
| OQ-N | Sample `access` events? | Obs | yes, 1-in-10 |
| OQ-O | capture-fallback buffer size/TTL | Obs | 10MB / 7d |
| OQ-Q | `engram doctor` repair vs report | Obs | report-only; `engram repair` separate |
| OQ-R | Agent identity: SO_PEERCRED socket creds? | Sec | yes for local MCP |
| OQ-S | Governance-delete cascade scope (git history, app-log, QMD, graph) | Sec | full cascade, dry-run default |
| OQ-T | Staging granularity (per-obs vs per-session log) | Arch/Fail | per-session append log |

## 7. OQ resolution status (original A–J)

| OQ | Status after Round 1 |
|----|----------------------|
| OQ-A (plugin in/out-of-process) | **RESOLVED** — one contract, two transports per-plugin (Arch §2.2) |
| OQ-B (MCP verb naming) | **RESOLVED** — `memory.*`/`dream.*`/`system.*` (Arch §3.4) |
| OQ-C (recall return shape) | **RESOLVED** — id+summary+score, `expand` opt-in |
| OQ-D (store paths) | open — `~/.engram/` global, `<repo>/.engram/` project (confirm) |
| OQ-E (agent identity) | **RESOLVED** — per-agent token + SO_PEERCRED (Arch §3.2 + Sec S-03) |
| OQ-F (QMD index staging/raw?) | **RESOLVED** — only `memories/`; worker reads staging directly |
| OQ-G (git-LFS vs ignore raw) | open — ignore default, LFS opt-in (confirm) |
| OQ-H (language = TS) | open — confirm TS in plan |
| OQ-I (job queue mechanism) | **RESOLVED** — SQLite `jobs` table (Arch §5.1) |
| OQ-J (dreaming worker: Agent SDK vs raw API) | open — **needs Round 2 research (mcp-agentsdk)** |

---

---

## 9. Round 2 — Research findings (2026-05-22)

Four research reports completed. They **validate the architecture** but surface
five findings that *change decisions already made* — flagged R-1…R-5 for the
author. They also resolve OQ-J and OQ-H.

### Round 2 confirmations
- **QMD** — GO (conditional). `QMDStore` library API stable since v2.0.0; maps
  cleanly to `RetrievalPlugin`. Blockers: local install is stale v0.9.0 (Bun-only)
  — upgrade to v2.5.1; verify `dist/index.js` loads under Node 22 (else use as
  MCP subprocess); ~2GB model cold-start download.
- **graphify** — GO (conditional, higher risk). Blockers: Markdown extraction
  needs an LLM call (mitigate with local Ollama backend); no headless build CLI
  (needs a wrapper script); ID-collision bug #952 affects multi-subdir layout.
- **OQ-J RESOLVED** — dreaming worker = **raw Anthropic API loop via the Vercel
  AI SDK behind `LlmPlugin`**, NOT the Claude Agent SDK (Agent SDK is
  Anthropic-only, bypasses the provider seam). Use raw `@anthropic-ai/sdk` inside
  `ClaudeLlmPlugin` for `cache_control` (1-hour prompt cache).
- **OQ-H RESOLVED** — TypeScript confirmed (QMD library, MCP SDK, Vercel AI SDK
  all TS-first).
- **agentmemory patterns** — staging buffer, 4-tier model, privacy filter,
  circuit-breaker/self-healing, `memory_diagnose`/`heal` all confirmed as sound
  reference patterns; its 51-tool surface and iii-engine dependency confirmed as
  over-built for engram (engram's 16-verb set + plain Node/SQLite is the right
  trim).

### Findings that CHANGE prior decisions — author calls needed

| ID | Finding | Prior decision it changes | Recommendation |
|----|---------|---------------------------|----------------|
| **R-1** | The memory-reliability literature (ElephantBroker, TierMem, MemMachine, the survey) is unanimous: **not consuming `confidence` in retrieval is a reliability mistake.** A speculative dreaming-authored memory (confidence 0.3) currently ranks equal to a human-confirmed fact (1.0). | Architecture review F-7.2 + D-7: "`confidence` stored-not-consumed in v1." | **Reverse it.** Confidence becomes a v1 multiplicative gate on the retrieval score. Cost: one multiplication + one `verification_state` enum field. Add `engram memory confirm <id>`. |
| **R-2** | One scoring formula across all four memory types is wrong — every peer system uses **per-type decay rates** (contextual decays in hours, semantic in months). | SPEC §3.6 applies one scoring model to all types. | **Adopt per-type decay constants.** Contextual memories also hard-transition to `dormant` at SessionEnd. |
| **R-3** | Transport should be **Streamable HTTP over `127.0.0.1`**, not stdio/unix-socket — engram's always-up daemon serving multiple per-session agents is the multi-client HTTP case; bearer-token auth is natural on HTTP. | SPEC OQ-M / D-5 leaned "CLI + unix socket, no TCP port"; original SPEC leaned stdio. | **Switch to Streamable HTTP on 127.0.0.1 + bearer token.** This also drops the SO_PEERCRED plan (OQ-R). Note: this reintroduces a localhost TCP surface — Security's X-1 concern — mitigated by 127.0.0.1-only bind + per-agent bearer token. |
| **R-4** | Episodic memories must be **immutable** during dreaming (MemMachine: episodic = ground truth; dreaming produces derivatives). Two-layer contradiction detection + a counterfactual validation gate before promoting procedural memories. | SPEC §5.2 does not prohibit dreaming from mutating episodic memories. | **Adopt** — episodic read-only in dreaming; `derived_from` backlinks mandatory; counterfactual gate for procedural promotion. |
| **R-5** | Add an **absolute active-pool floor** — `min(100, 20% of total)` memories must always stay `active` — on top of the 5% per-run rate limit, to prevent cascade-archiving. | Failure review §8 had only the 5% rate limit. | **Adopt** the floor as an additional unconditional merge constraint. |

### Round 2 — net effect on v1 scope
D-7 trimmed `confidence` out of v1; **R-1 puts it back** (it is cheap and
reliability-critical). The cross-agent-dreaming deferral (D-3) stands. The
relation-kinds trim (2 kinds) stands. Net: v1 gains confidence-weighted
retrieval and per-type decay; both are small code, high value.

## 10. Recommended next actions

1. **Author decisions on R-1…R-5** (§9) — five findings that revise earlier calls.
2. **Spec rewrite (SPEC v2)** folding in: the 4 contracts, §13 Failure, §OT
   Observability, the threat model, D-1…D-7, R-1…R-5, OQ-J/OQ-H resolutions,
   the v1-scope changes, and the Round-2 tool blockers as implementation notes.
3. **Round 3 verification** — lighter re-review of SPEC v2 for internal
   contract consistency (the `judge` skill or a single architect-reviewer pass).
4. Then `writing-plans` for the phased implementation plan.
5. Task 7 (archive old plan + remove `.claude` memory setup) — still pending,
   awaiting the author's call on the `claude/memory/` content question.

---

*Round 1 + Round 2 synthesis complete. Author decisions pending: R-1…R-5.*
