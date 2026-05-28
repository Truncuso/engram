# Detailed Findings — Memory System Overhaul

**Date**: 2026-05-17
**Updated**: 2026-05-18
**Analysis Method**: Multi-agent (security-reviewer, architect, Explore × 3) parallel review of the external implementation guide and existing memory infrastructure, plus a 2026-05-18 factual-accuracy correction pass against the live installation.
**Sources:** [SRC-01], [SRC-02], [SRC-03], [SRC-04]

---

## 2026-05-18 Factual Corrections

A verification pass against the actual installation found errors in the original findings:

| Original (erroneous) statement | Corrected statement |
|--------------------------------|---------------------|
| Glite `MEMORY.md` is **7,660 lines** (300×–600× over cap) | Glite `MEMORY.md` is **143 lines** — already well-structured. The 7,660 figure was fabricated. Typed-file rationale re-grounded — see Finding 1 (revised). |
| Plan paths use `/home/christoph/...` | Actual user is `cunger`; home is `/home/cunger`. All paths corrected. |
| `~/.claude/.memory/` is a fresh tree; auto-memory uses `projects/<slug>/memory/` | Auto-memory **already writes** to `~/.claude/projects/<slug>/memory/`. `memory-init --global` retargets the `CLAUDE.md` auto-memory block to `~/.claude/.memory/` to unify. |
| `WP-E` "extends" `session-start.js` to load memory | `session-start.js` reads no memory and is flag-gated via `lifecycle-launcher.js`. Memory-load is a **net-new** sibling script. |

---

## Corrections Summary

| Original Claim (from external guide [SRC-01]) | Corrected Finding (this plan) |
|-----------------------------------------------|-------------------------------|
| `context/MEMORY.md` with 2,500-char cap is the right shape | Typed-file architecture is superior — see Finding 1 |
| `memsearch` Python package is the retrieval backend | QMD MCP (already installed, Tier-2 in CLAUDE.md) is the only backend — see Finding 2 |
| Stop hook captures 500 chars per response into transcripts | Defer raw transcript capture entirely — see Finding 3 |
| Three cron jobs (daily/nightly/weekly) automate distillation | Single manually-invoked `memory-curate` skill — see Finding 4 |
| `context/memory/` (daily logs) is committed | `.memory/daily/` is gitignored automatically — see Finding 5 |

---

## Finding 1 (revised): Typed-file architecture beats single-file MEMORY.md

### 1.1 Current State

Two real-world examples exist in `~/.claude/projects/<slug>/memory/`:
- **fsl-cleaningapplication** (3 files, 82 lines total): already partially typed (`feedback_*.md`, `*_wp_*_complete.md`) — the typed-file pattern in use.
- **glite** (1 file, **143 lines**): a single `MEMORY.md`. Well-structured by `##` sections (Documentation Structure, Key Architecture Patterns, etc.), not a blob.

> **Correction (2026-05-18):** an earlier draft of this finding claimed glite's `MEMORY.md` was **7,660 lines**. That figure was fabricated. The real file is **143 lines** (`wc -l` verified). The typed-file decision does **not** depend on a giant file existing — it stands on the structural grounds in §1.2.

The external guide [SRC-01] proposes a single capped `context/MEMORY.md` (2,500 chars). The fsl-cleaningapplication pattern (typed files) is the better model regardless of corpus size.

### 1.2 Structural Differences — the real rationale

| Aspect | Single capped MEMORY.md (guide) | Typed files (this plan) |
|--------|-------------------------------|------------------------|
| Session-start token cost | Whole file loaded every session | Pointer-index only; content retrieved on demand |
| Search precision | Section-based grep over one blob | Per-file frontmatter (`type`, `scope`) + QMD ranked search |
| Write friction | Read-merge-rewrite-check-`wc -c` every write | Append one new file + one index line |
| Concurrency | Two sessions race on one file | Two sessions append two distinct files — no race |
| Linkability | Section anchors break on edits | `[[slug]]` resolves to a stable file path |
| Lifecycle | Manual surgical edits, prone to drift | Per-file deletion / archive / rename |
| Scope metadata | Naming convention (`global_` / `project_` prefixes) | First-class `scope:` frontmatter key — error-proof |

The decisive arguments are **concurrency safety**, **write friction**, **scope metadata**, and **per-file lifecycle** — none of which require a large corpus to motivate. A 143-line glite file and an 82-line fsl file both benefit.

### 1.3 Confirmed Gap

The arbitrary 2,500-char cap conflates two separate concerns: *what is stored on disk* and *what is loaded at session start*. A typed system separates them — the on-disk store is unbounded; the session-start snapshot is bounded by loading only indexes + 1-page summaries (≤ 2,500 tokens). The cap, applied to a single file, instead forces destructive consolidation as memory grows.

**Impact**: Adopting the guide as-written would force consolidation/truncation pressure on every write once the cap is hit. Typed files remove that pressure entirely.

### 1.4 Relevant Code Paths

| File | Size | Role |
|------|------|------|
| `~/.claude/projects/-home-cunger-10-Projects-01-fsl-cleaningapplication/memory/` | 3 files, 82 lines | Working example of the typed-file pattern |
| `~/.claude/projects/-home-cunger-10-Projects-11-private-OSRS-glite-glite-private/memory/MEMORY.md` | 143 lines | Single-file example; section-structured; migrated by `memory-init` Phase 5 |

---

## Finding 2: QMD is the correct retrieval backend; memsearch is rejected

### 2.1 Current State

`~/.claude/CLAUDE.md` already mandates LSP → QMD → Grep → Read tiering for all code intelligence and markdown search. QMD is installed, configured, and Tier-2 by policy. The external guide [SRC-01] proposes installing a separate Python package `memsearch` (558MB ONNX model download) for hybrid BM25 + vector retrieval over memory files only.

### 2.2 Confirmed Gap

Adding `memsearch` would create a duplicate retrieval pipeline parallel to QMD. The security review [SRC-02] flagged:
- `memsearch` PyPI metadata is missing (no author, no homepage, no source repo)
- Default `embedding.provider = "openai"` — running `memsearch index` without overriding sends memory contents to OpenAI's embedding endpoint
- ONNX model fetched from HuggingFace without integrity check at runtime

QMD already handles memory's use case. If QMD lacks vector embeddings on a given machine (noted in the MCP block for this user), `vector_search` degrades to BM25 — acceptable; fix QMD if vector quality becomes the bottleneck.

**Impact**: Saved a 558MB download, eliminated supply-chain + OpenAI-default risks, removed duplicate infrastructure.

### 2.3 Relevant Code Paths

| File | Line(s) | Role |
|------|---------|------|
| `~/.claude/CLAUDE.md` | "Code Intelligence (LSP-first retrieval)" section | Canonical tiering policy |
| Plan source §4.2 | 196-213 | Memory retrieval tiering table |

---

## Finding 3: Raw transcript capture is too risky to ship

### 3.1 Current State

The external guide [SRC-01] proposes a Node.js `Stop` hook that captures the first 500 chars of every assistant response into `context/transcripts/{YYYY-MM-DD}.md`.

### 3.2 Confirmed Gap

Security review [SRC-02] classified this as HIGH risk:
- 500-char capture is unfiltered — if user pastes a secret and the agent echoes it back, the secret is written to disk
- Any session involving `.env`, API credentials, customer data → those summaries land in `context/transcripts/`
- The guide gitignored `context/transcripts/` (good) but the **daily memory log** at `context/memory/` was NOT gitignored — so structured summaries (which an LLM could also incorporate secrets into) would land in git history permanently

**Impact**: Defer the capture entirely. Use agent-authored summaries on `/handoff` only. The agent has discretion to summarize without echoing literal secret values. Daily logs are still gitignored.

### 3.3 Relevant Code Paths

| File | Line(s) | Role |
|------|---------|------|
| Plan source §9 risk table | (Risks & Security section) | HIGH-risk classification |
| Plan source §5.1 | 273-284 | Replacement: agent-authored daily-log format |

---

## Finding 4: Cron-driven automation is rejected; manual curation only

### 4.1 Current State

The external guide [SRC-01] proposes three cron jobs:
- 23:00 daily — distill daily logs into MEMORY.md (haiku model)
- 02:00 nightly — re-index memsearch (haiku model)
- 09:00 Sundays — curator pass (sonnet model)

### 4.2 Confirmed Gap

- Cron jobs invoke LLM with full memory contents. If memory ever contains a secret → it's sent to API every night.
- `notify: on_failure` semantics depend on cron runner implementation; failure output may include memory excerpt to a world-readable log path.
- Adds hidden complexity (cron daemon + LLM calls + memsearch reindex pipeline) for a feature unproven in user workflow.

**Impact**: Replace with single foreground `memory-curate` skill. User invokes manually; sees what's sent to LLM. If skipped consistently after 4+ weeks of usage, revisit scheduling (deferred to OQ-02).

### 4.3 Relevant Code Paths

| File | Line(s) | Role |
|------|---------|------|
| Plan source §5.2 | 287-297 | Manual `/memory-curate` replacement |
| Plan source §6.3 | 384-410 | Full memory-curate skill spec |

---

## Finding 5: `daily/` must be gitignored by default

### 5.1 Current State

The external guide [SRC-01] gitignores `context/transcripts/` and `.memsearch/` but **omits** `context/memory/` (the daily logs). Daily logs record "Deliverables", "Decisions", "Open threads" — if an agent ever summarizes a credential or customer record, that goes into git history permanently.

### 5.2 Confirmed Gap

Even agent-authored summaries can incorporate sensitive data unintentionally. The risk is fundamental: any agent-written file in a shared repo path could leak. Mitigation: `.memory/daily/` and `.memory/_archive/` are gitignored automatically by `memory-init`. Typed subdirs (`project/`, `feedback/`, `reference/`) remain committed because they're explicitly-authored facts, not session-trace summaries — the user controls what goes in.

**Impact**: Zero per-session leak surface in git history. Typed memory remains team-shareable.

### 5.3 Relevant Code Paths

| File | Line(s) | Role |
|------|---------|------|
| Plan source §3.1 directory layout | 56-68 | Project tree with gitignore annotations |
| Plan source §9 risk table CRITICAL row 2 | (Risks & Security section) | Mitigation in `memory-init` Phase 3 |
| WP-A target file `<repo>/.gitignore` | runtime | `.gitignore` modification per `memory-init` |

---

## Finding 6: QMD index freshness needs trigger-driven refresh (added 2026-05-18)

### 6.1 Current State

`qmd update` (BM25 re-index) and `qmd embed` (semantic vectors) are **one-shot**
CLI commands. QMD has no built-in watcher or daemon. The SessionStart memory-load
hook only *reads* memory — nothing re-indexes it. End-to-end testing confirmed:
a fact written mid-session returns "No results" from `qmd search` until a refresh
runs manually.

The external guide [SRC-01] solved this with cron; this plan rejected cron
(Finding 4). The OpenClaw QMD-memory model (`docs.openclaw.ai/concepts/memory-qmd`)
runs `qmd update` periodically (~5 min) and `qmd embed` for semantic modes, as
**one-shot subprocesses, not a persistent watcher**, plus boot-time reconciliation.

### 6.2 Confirmed Gap

The cron rejection in Finding 4 was about **LLM-invoking** jobs — three cron
entries that sent memory *contents to an LLM* nightly (a data-exfiltration and
hidden-cost risk). `qmd update`/`qmd embed` are categorically different: pure
local indexers, no LLM, no network beyond QMD's own model cache. Conflating the
two left the system with no freshness mechanism at all.

### 6.3 Resolution — trigger-driven freshness

A shared one-shot script `qmd-refresh.cjs` runs `qmd update` (always, staleness-
gated) and `qmd embed` (gated on `searchMode`, debounced ≤ 1/hour). It is invoked
by three trigger classes — **no cron, no daemon**:

| Trigger | Mechanism | Behavior |
|---------|-----------|----------|
| Session start | `session-start-memory.cjs` calls it before loading | refresh if index > 10 min stale |
| Session end | `session-end-memory.cjs` (new SessionEnd hook) | `--force` refresh so the session's writes are indexed |
| On write | `memory-write` / `capture-learning` / `handoff` skill step | `--force` after writing a typed file |
| Curation | `memory-curate` RE-INDEX phase | `--force` full update + embed |

The **Stop** hook is deliberately excluded: it fires after every assistant
response, so a per-response re-index is wasteful — and Stop is also where the
external guide's rejected transcript-capture risk lived.

### 6.4 Verified

End-to-end test 2026-05-18: a new typed file was invisible to `qmd search`
before refresh, and returned a 64%-score hit after the trigger-driven refresh —
with no manual `qmd update`. The `verify-memory.cjs` loop gained freshness checks
(refresh script + SessionEnd hook installed and registered); 37/37 ALL GREEN.
