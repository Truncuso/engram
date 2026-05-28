# Memory System Overhaul: Global + Per-Project, Unified

**Date**: 2026-05-17
**Updated**: 2026-05-18
**Author**: cunger
**Status**: COMPLETE — all phases done (2026-05-19). Superseded by Phase 2: `planning-2026-05-19-llm-wiki-architecture/`

> **Source plan:** the design session serialized into this directory.
> **Implementation guide:** `memory-init-implementation-guide.md` (in this directory) — the runnable, step-by-step guide modeled on the external guide but for the corrected architecture.

---

## Executive Summary

The Claude Code installation currently has **four** uncoordinated memory mechanisms:

1. The global system-prompt "auto memory" spec — typed `.md` files, but pinned to a per-slug path `~/.claude/projects/<slug>/memory/` (live; currently empty for the dotfiles slug).
2. Legacy populated memory under the same per-slug root for two other projects (`fsl-cleaningapplication`, `glite`).
3. Freeform `.memory/MEMORY.md` referenced by `setup-sdd-repo` (its template `~/.claude/templates/sdd/MEMORY.md` does **not exist on disk** — the path is broken).
4. Ralph-Loop `.memory/*.json` state from `implement-and-verify`.

A fifth path (an external implementation guide proposing `context/MEMORY.md` + `memsearch` + transcript capture + cron) was reviewed and **rejected** — see Corrections Log.

This plan unifies memory into **one architecture**:

- **Global scope** → `~/.claude/.memory/` (canonical). The harness-injected "auto memory" system-prompt block is **built-in and not editable** — it hard-codes `~/.claude/projects/<slug>/memory/`. `memory-init --global` therefore adds an explicit **override block** to `~/.claude/CLAUDE.md` (user instructions outrank the system-prompt default) that redirects all memory operations to `~/.claude/.memory/` and marks the per-slug path deprecated.
- **Project scope** → `<repo>/.memory/`.
- Both scopes use **typed memory files** (one `.md` per fact with frontmatter) indexed by a pointer-only `MEMORY.md`.
- Retrieval reuses the **QMD MCP** (already mandated as Tier-2 in `CLAUDE.md`); no new dependency.
- Session-start loads only indexes + 1-page summaries (≤ 2,500 tokens) via a new memory-load SessionStart hook.

**Delivery model (re-scoped 2026-05-18):** The entire setup is delivered through **one skill — `memory-init`** — not a chain of work packages. `memory-init` is idempotent and runnable for both `--global` and `--project`. Its phases perform scaffolding, legacy migration, gitignore wiring, and SessionStart-hook installation. The two companion skills (`memory-write`, `memory-curate`) and the five edited skills (`capture-learning`, `handoff`, `grill-with-memory`, `setup-sdd-repo`, `repo-governance`) are delivered as **one-time edits applied directly**, documented in the implementation guide. Every new and edited skill gets a **`skill-creator` review-and-improve pass**.

Raw transcript capture, cron, and `memsearch` are explicitly out of scope.

---

## Current State — verified 2026-05-18

A live end-to-end verification pass against the running installation confirms:

- **Infrastructure is built, installed, and working.** Global `~/.claude/.memory/`
  and project `~/.claude/.memory/` trees exist and are populated. The
  SessionStart (`session-start-memory.cjs`) and SessionEnd
  (`session-end-memory.cjs`) hooks are installed and registered in `hooks.json`;
  a live run returns valid `<memory-context>` JSON. QMD v2.1.0 has both
  collections registered and indexed; `qmd search` returns hits. The CLAUDE.md
  override is in place and the legacy per-slug path does not exist (no
  split-brain).
- **All phases are COMPLETE** (2026-05-19). D1–D13 delivered; the new
  `grill-with-memory` skill supersedes `grill-with-docs`, which has been removed.
  Phase 6 (edited-skill review + skill-stocktake: 9/9 Keep) and Phase 7 (E2E
  verification 37/37, write→recall flow, prompt-injection test, hook code-review
  with 2 HIGH fixes) are closed — see `skill-creator-review-findings.md`.
- **This plan is superseded by Phase 2** —
  `planning-2026-05-19-llm-wiki-architecture/` — the LLM-wiki overhaul.
- **An empty `daily/` directory is the expected state**, not a fault. Automatic
  background capture (Stop-hook transcript capture, cron distillation) was
  deliberately rejected on security grounds (see FINDINGS.md Findings 3–6).
  Memory fills only via explicit, user-visible triggers: `/memory-write`,
  `/handoff`, `capture-learning`, grilling. The QMD *index* refresh is automated;
  content *capture* is not.

---

## Deliverables

| # | Deliverable | Type | Notes |
|---|-------------|------|-------|
| D1 | `memory-init` skill | NEW skill | Scaffold + migrate + gitignore + SessionStart-hook install; `--global` and `--project`. Idempotent. |
| D2 | `memory-write` skill | NEW skill | Explicit save: "remember this" / "/remember" / "/forget". Scope+type routing. |
| D3 | `memory-curate` skill | NEW skill | Manual housekeeping: DISTILL → PRUNE → MERGE → ARCHIVE → RE-INDEX → REPORT. |
| D4 | `capture-learning` edit | Skill edit | Phase 3–5: route phase-boundary observations to correct scope+type. |
| D5 | `handoff` edit | Skill edit | Write `project/handoff_<slug>.md` mirror + append `daily/<today>.md`. |
| D6 | `grill-with-memory` edit | Skill edit | Add trigger aliases (`grill me`, `grill with documents`, `grill docs`), enforce one-question interactive clarification, and write `reference/glossary_<term>.md` + `project/adr_<NNN>_<slug>.md` stubs (with `/memory-init --project` suggestion path when `.memory/` is missing). |
| D7 | `setup-sdd-repo` edit | Skill edit | Replace broken inline `.memory/` template with `/memory-init --project` delegation. |
| D8 | `repo-governance` edit | Skill edit | Extend stale-link scan to walk typed memory subdirs and check `links:` frontmatter. |
| D9 | SessionStart memory-load hook | NEW hook script | `~/.claude/scripts/hooks/session-start-memory.cjs`, registered in `~/.claude/hooks/hooks.json`. Refreshes the QMD index if stale before loading. |
| D9b | QMD index-freshness machinery | NEW hook scripts | `qmd-refresh.cjs` (one-shot `qmd update` + gated/debounced `qmd embed`) + `session-end-memory.cjs` (SessionEnd refresh). Trigger-driven, no cron, no daemon, no LLM. |
| D10 | `~/.claude/CLAUDE.md` memory override block | Edit | Add an override block that redirects memory ops to `~/.claude/.memory/` (the harness auto-memory default cannot be edited directly); document precedence. |
| D11 | `memory-init-implementation-guide.md` | Doc | Runnable guide (this directory). |
| D12 | `skill-creator` review pass | Process | Run on D1–D8 after edits land. Capture findings in this directory. |
| D13 | `memory-onboard` skill | NEW skill | Token-efficient cold-repo onboarding: cheap signals → QMD sampling → grill (invokes `grill-with-memory`) → seed CONTEXT.md/ADR/project memory. Hard source-file read budget; runs after `/memory-init --project`. |

No work-package files. The legacy `WP-*.md` files in this directory are archived (see Archived section).

---

## Architecture (corrected)

### Directory layout

```text
~/.claude/.memory/                 # GLOBAL scope (canonical)
├── MEMORY.md                     # pointer-index only
├── USER.md                       # identity + cross-project prefs (1-page)
├── GLOSSARY.md                   # cross-project terminology
├── user/        *.md             # typed: user facts
├── feedback/    *.md             # typed: corrections / validated approaches
├── reference/   *.md             # typed: pointers to external systems
└── daily/       <YYYY-MM-DD>.md  # session traces (NOT applicable globally; project only)

<repo>/.memory/                   # PROJECT scope
├── MEMORY.md                     # pointer-index only
├── PROJECT.md                    # build/branch/conventions snapshot (1-page)
├── user/        *.md
├── feedback/    *.md
├── project/     *.md             # typed: project facts, decisions, WP outcomes
├── reference/   *.md
├── daily/       <YYYY-MM-DD>.md  # gitignored
└── _archive/    ...              # gitignored
```

`.gitignore` (written by `memory-init --project`): ignore `.memory/daily/` and `.memory/_archive/`; everything else under `.memory/` is committable.

### Precedence

Project-scoped facts win over global on conflict (more specific = higher priority). Documented in every `MEMORY.md` header, in `~/.claude/CLAUDE.md`, and in the SessionStart hook framing.

### Retrieval tiering

Memory slots into the existing `CLAUDE.md` policy as **Tier 2 (QMD)**. No new backend. If QMD lacks vector embeddings on a machine, `vector_search` degrades to BM25 — acceptable.

---

## Archived / Closed / Deleted

| Item | Outcome |
|------|---------|
| `WP-A` … `WP-E`, `WP-C1` … `WP-C5` (9 files) | ARCHIVED 2026-05-18 — superseded by the single-skill delivery model. Content folded into `memory-init` phases and the implementation guide. Files retained for rationale trail; not active. |

---

## Corrections Log

| Previous Claim | Corrected Finding |
|----------------|-------------------|
| Global memory tree is net-new at `~/.claude/.memory/`; "auto memory" lives at `~/.claude/projects/<slug>/memory/` | The harness-injected auto-memory block **already writes** to `~/.claude/projects/<slug>/memory/` and is **live** (empty for the dotfiles slug). That block is **built into the harness** — not in any editable file. **Resolution:** `~/.claude/.memory/` is canonical; `memory-init --global` adds an **override block** to `~/.claude/CLAUDE.md` (user instructions outrank the system-prompt default) that redirects memory ops to `~/.claude/.memory/` and deprecates the per-slug path. |
| Glite `MEMORY.md` is **7,660 lines** — "300×–600× over the 2,500-char cap" — the headline justification for typed files | **FALSE.** Actual file is **143 lines**, already well-structured. The 7,660 figure is fabricated. **Resolution:** corrected everywhere; typed-file decision re-justified on real grounds (scope metadata, no concurrent-append race, per-file lifecycle, QMD search precision) — none of which need a giant file to motivate. |
| `WP-E` "extends" `~/.claude/scripts/hooks/session-start.js` to load memory | The existing `session-start.js` injects via `hookSpecificOutput.additionalContext` and **reads no memory files**. It is routed through `lifecycle-launcher.js` with flag-gating (`minimal,standard,strict`) in `hooks.json`. **Resolution:** memory-load is a **net-new** script `session-start-memory.cjs`, registered as an additional `SessionStart` entry — the existing hook is left untouched. |
| Plan paths use `/home/christoph/...`, author `christoph`, source on `/media/christoph/Samsung_Evo990/...` | Actual user is `cunger`, home `/home/cunger`. **Resolution:** all paths corrected. SRC-01 real location: `/home/cunger/10_Projects/Agentic AI/Workflows/Notes/agentic-memory/Claude Code Memory Improvements - Implementation Guide.md`. |
| External guide: `memsearch` Python package (BM25 + ONNX) as retrieval backend | Rejected — QMD MCP already mandated as Tier-2; no new dependency; supply-chain + OpenAI-embedding-default risks (see `QMD-vs-MemSearch-Analysis.md`). |
| External guide: Stop hook capturing first 500 chars of every assistant response | Rejected — HIGH risk for secret leakage. Replaced with agent-authored daily summary written only on `/handoff`. |
| External guide: single `context/MEMORY.md` with 2,500-char cap | Rejected — typed-file architecture preserves search granularity, lower write friction, no concurrent-append race, per-file lifecycle. |
| External guide: three cron jobs (daily distill, nightly index, weekly curator) | Rejected — replaced with single manually-invoked `memory-curate` skill; keeps the user in the loop on what is sent to an LLM. |
| Cron rejection read as "no automatic indexing at all" | Clarified 2026-05-18 — the rejection targeted **LLM-invoking** cron (jobs sending memory *contents* to an LLM nightly). `qmd update`/`qmd embed` are pure local indexers — no LLM, no network. The system needs them automatic or search goes stale after every write. **Resolution:** trigger-driven freshness (SessionStart-if-stale, SessionEnd, on-write in skills) via `qmd-refresh.cjs`; still no cron, no daemon. The **Stop** hook is deliberately not used (per-response re-index is wasteful; Stop is also where the rejected transcript-capture risk lived). |
| QMD retrieval "just works" once a collection exists | Corrected 2026-05-18 — `qmd update`/`qmd embed` are one-shot; QMD has no watcher. A fact written mid-session is not searchable until a refresh runs. Freshness must be triggered explicitly (D9b). |
| External guide: `daily/` committed by default | Corrected — `daily/` and `_archive/` are gitignored automatically by `memory-init`. |
| Delivery via 9 sequential work packages | Re-scoped 2026-05-18 — single `memory-init` skill is the deliverable; companion skills and the 5 edits applied directly. WP files archived. |

---

## Key Decisions

| Decision | Rationale | Date |
|----------|-----------|------|
| `~/.claude/.memory/` is the canonical global store | One store; `memory-init` adds a CLAUDE.md override block (the harness auto-memory default is not editable) so the per-slug path no longer fragments memory | 2026-05-18 |
| Single `memory-init` skill, no work packages | User decision: setup must be one runnable, idempotent skill — not a WP chain | 2026-05-18 |
| 5 existing-skill edits applied directly (not re-run by memory-init) | Keeps `memory-init` focused on scaffold/migrate/hook; edits are one-time and documented in the guide | 2026-05-18 |
| All new + edited skills get a `skill-creator` review pass | User decision: verify frontmatter, triggers, descriptions; improve quality | 2026-05-18 |
| Typed files over single capped `MEMORY.md` | Better search, lower write friction, no concurrent-append race, stable `[[slug]]` links, per-file lifecycle | 2026-05-17 |
| QMD-only retrieval | Already mandated as Tier-2 in `CLAUDE.md`; no new deps | 2026-05-17 |
| Defer raw transcript capture | Security review HIGH risk; agent-authored summaries instead | 2026-05-17 |
| Mixed gitignore: `daily/`+`_archive/` ignored, `user/`+`feedback/`+`project/`+`reference/` committed | Team-shareable knowledge without session-trace leakage | 2026-05-17 |
| Daily log v1: only on `/handoff` | Skip per-Stop summarization complexity | 2026-05-17 |
| Manual `memory-curate` instead of cron | Reject hidden complexity; user sees what is sent to an LLM | 2026-05-17 |
| QMD index freshness is trigger-driven, not cron | `qmd update`/`qmd embed` are local no-LLM indexers — safe to automate. SessionStart refreshes if >10min stale; SessionEnd forces a refresh; memory-write/capture-learning/handoff refresh on-write. No cron, no daemon. Stop hook excluded (per-response re-index wasteful). | 2026-05-18 |
| `qmd embed` gated on searchMode + debounced ≤ 1/hour | Embedding is the slow part; BM25 (`qmd update`) is cheap and runs every trigger. `QMD_MEMORY_SEARCH_MODE=search` skips embedding entirely. | 2026-05-18 |
| Cold-repo onboarding is a separate skill (`memory-onboard`), not a `memory-init` mode | `memory-init` is mechanical + idempotent; onboarding is interpretive (grilling, judgement). Mixing them makes `memory-init` non-idempotent and unpredictable. Separate skills, agent-driven hand-off. | 2026-05-18 |
| `memory-onboard` reads cheap signals → QMD-samples → grills, under a hard read budget | Big repos cannot be read whole. Bound the agent's *reading*; the user's grilled answers supply expensive context for near-zero tokens. Default budget: 12 source files. | 2026-05-18 |
| Skill chaining is documented agent-driven hand-offs, not a formal state machine | Matches existing `capture-learning`/`diagnose` chaining in this repo: each skill names the next; the agent decides. Flexible, no rigid coupling to maintain. | 2026-05-18 |
| `MEMORY.md` is pointer-index only | Cheap to load; content lives in typed files retrieved on demand | 2026-05-17 |
| Project memory wins on conflict with global | Project context is more specific | 2026-05-17 |
| SessionStart memory-load is a net-new hook script | Existing `session-start.js` reads no memory; cleaner to add a sibling than to entangle | 2026-05-18 |

---

## Project Sources

See `SOURCES.md` for full metadata. Key sources: external implementation guide (SRC-01, superseded), QMD-vs-MemSearch analysis, security + architect reviews, existing-infrastructure survey, `CLAUDE.md` tiering policy.

---

## Acceptance Criteria

- [ ] `/memory-init --global` creates `~/.claude/.memory/` tree; idempotent on re-run; adds a memory-override block to `~/.claude/CLAUDE.md` redirecting memory ops to `~/.claude/.memory/`.
- [ ] `/memory-init` in a repo creates `<repo>/.memory/` tree with correct `.gitignore` entries; idempotent.
- [ ] `/memory-init` installs the SessionStart memory-load hook (`session-start-memory.cjs` + `hooks.json` entry) if absent; idempotent.
- [ ] `/memory-write` routes to correct scope+type, writes typed file + index line, echoes confirmation.
- [ ] `/memory-curate` walks both scopes interactively; archives daily logs > 30 days.
- [ ] `capture-learning` routes 5/5 hand-curated observations to correct scope+type.
- [ ] `handoff` writes `tmp/handoffs/...` AND `<repo>/.memory/project/handoff_<slug>.md` AND appends `daily/<today>.md`.
- [ ] `grill-with-memory` writes **project-scope only** stubs/mirrors (`<repo>/.memory/reference/glossary_<term>.md`, `<repo>/.memory/project/adr_<NNN>_<slug>.md`) and suggests `/memory-init --project` when `.memory/` is missing.
- [ ] `setup-sdd-repo` delegates `.memory/` scaffolding to `/memory-init --project`; no dangling `templates/sdd/MEMORY.md` reference.
- [ ] `repo-governance` extended scan flags dangling `links:` targets in typed memory files.
- [ ] Legacy `~/.claude/projects/<slug>/memory/` files heuristic-split into typed subdirs; originals preserved.
- [ ] SessionStart in a memory-initialized repo loads global+project memory silently; total ≤ 2,500 tokens; wrapped in `<memory-context>` framing.
- [ ] `skill-creator` review pass run on all 8 new/edited skills; findings recorded.
