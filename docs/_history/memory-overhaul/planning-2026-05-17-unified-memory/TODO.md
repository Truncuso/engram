# TODO — Memory System Overhaul

**Date**: 2026-05-17
**Updated**: 2026-05-18
**Status**: COMPLETE — all phases done (Phase 6 + 7 closed 2026-05-19). Superseded by Phase 2 (`planning-2026-05-19-llm-wiki-architecture/`).

> Delivery is via the `memory-init` skill plus directly-applied skill edits.
> See `memory-init-implementation-guide.md` for the runnable step-by-step.

---

## Phase 1 — Planning & Research (DONE)

- [x] Read external implementation guide (`/home/cunger/10_Projects/Agentic AI/Workflows/Notes/agentic-memory/Claude Code Memory Improvements - Implementation Guide.md`)
- [x] Read QMD-vs-MemSearch analysis
- [x] Survey existing memory infrastructure (auto-memory path, legacy `projects/<slug>/memory/`, `.memory/`, hooks)
- [x] Security review — 3 CRITICAL/HIGH blockers identified and mitigated
- [x] Architecture review — typed-file + QMD-only architecture accepted
- [x] Correct 5 factual errors in the plan (christoph→cunger paths, 7660→143 Glite figure, canonical path, session-start hook reality, SRC-01 path)
- [x] Re-scope: 9 work packages collapsed into one `memory-init` skill deliverable

---

## Phase 2 — Build `memory-init` skill (D1) — DONE

`memory-init` is the single setup deliverable. Idempotent; `--global` and `--project` modes.

- [x] `/skill-creator` → scaffold `~/.claude/skills/memory-init/`
- [x] Author `SKILL.md` — frontmatter (`name`, `description`, `argument-hint: [--global|--project]`)
- [x] Phase 1 VERIFY PREREQUISITES — QMD MCP reachable; detect scope from arg/cwd
- [x] Phase 2 DISCOVER SIGNALS — git branch, build files, existing CLAUDE.md, existing memory
- [x] Phase 3 SCAFFOLD — create tree, write templates, write `.gitignore` (project mode)
- [x] Phase 4 OVERRIDE (global mode) — add a memory-override block to `~/.claude/CLAUDE.md` redirecting memory ops to `~/.claude/.memory/` (harness auto-memory default is not editable)
- [x] Phase 5 MIGRATE — heuristic-split legacy `~/.claude/projects/<slug>/memory/` into typed subdirs (copy, not move)
- [x] Phase 6 INSTALL HOOK — write `session-start-memory.cjs`; add `SessionStart` entry to `hooks.json` if absent
- [x] Phase 7 VERIFY — QMD indexes new tree; fresh-subagent token-budget check ≤ 2,500
- [x] Create `assets/templates/`: `MEMORY.md`, `USER.md`, `PROJECT.md`, `GLOSSARY.md`, `memory-frontmatter.yaml`
- [x] Create `assets/session-start-memory.cjs` — the hook script template
- [x] Create `references/migration-heuristics.md` — file-name → type-subdir rules
- [x] Create `references/implementation-guide.md` — annotated copy of the external guide (rationale trail)
- [x] Test `/memory-init --global` (idempotent re-run; CLAUDE.md block rewritten once)
- [x] Test `/memory-init` in dotfiles repo (`.gitignore` correct; daily/ + _archive/ ignored)
- [x] Test legacy migration against `fsl-cleaningapplication` and `glite` slugs (originals preserved)
- [x] Verify SessionStart memory-load fires silently; token budget ≤ 2,500

---

## Phase 3 — Companion skills (D2, D3) — DONE

- [x] `/skill-creator` → `memory-write`; author per implementation guide
  - triggers: "remember this", "note that", "save this", "/remember", "/forget"
  - scope decision (global vs project); type decision (user/feedback/project/reference)
  - pre-write checks: dedup, CONTEXT.md nudge, file-name collision
  - actions: add / replace / remove (remove confirms first)
  - [x] Test: user fact (global), project decision, glossary mirror, feedback correction, forget, dedup
- [x] `/skill-creator` → `memory-curate`; author per implementation guide
  - phases: DISTILL → PRUNE → MERGE → ARCHIVE → RE-INDEX → REPORT
  - [x] Test against synthetic stale daily logs; archive logs > 30 days

---

## Phase 4 — Direct skill edits (D4–D8) — DONE

Applied directly as one-time edits. Each documented in the implementation guide.

- [x] D4 `capture-learning` — Phase 3 add scope+type sub-decision; Phase 4 write typed file w/ frontmatter; Phase 5 index correct scope
- [x] D5 `handoff` — after tmp/handoff write: write `project/handoff_<slug>.md`; append `daily/<today>.md`; add index line
- [x] D6 `grill-with-memory` (supersedes `grill-with-docs`; declares `grill-me`/`grill-with-docs`/`grill docs` as backward-compatible aliases) — trigger aliases added, one-question interactive clarification enforced, writes `reference/glossary_<term>.md` after a CONTEXT.md term and `project/adr_<NNN>_<slug>.md` after an ADR; suggests `/memory-init --project` when `.memory/` is missing. Old `grill-with-docs/` directory removed.
- [x] D7 `setup-sdd-repo` — remove broken `templates/sdd/MEMORY.md` reference; replace with `/memory-init --project` delegation
- [x] D8 `repo-governance` — extend scan to walk `.memory/{user,feedback,project,reference}/`; validate `links:` targets

---

## Phase 5 — CLAUDE.md + hooks + index freshness (D9, D9b, D10) — DONE

- [x] D10 `~/.claude/CLAUDE.md` — memory-override block added (redirects memory ops to `~/.claude/.memory/`); precedence documented
- [x] D9 `session-start-memory.cjs` — load order, `<memory-context>` framing, `$PWD`-bounded resolution, ≤ 2,500 token budget; refreshes index if stale before loading
- [x] D9b `qmd-refresh.cjs` — one-shot `qmd update` (staleness-gated) + `qmd embed` (gated on `searchMode`, debounced ≤ 1/hour); no cron, no daemon
- [x] D9b `session-end-memory.cjs` — SessionEnd hook forces a refresh so a session's writes are searchable next session; registered in `hooks.json`
- [x] On-write refresh wired into `memory-write`, `capture-learning`, `handoff`; `memory-curate` RE-INDEX forces a full refresh
- [x] Stop hook deliberately excluded from refresh triggers (per-response re-index wasteful)

---

## Phase 5b — `memory-onboard` skill (D13) — DONE

- [x] Build `~/.claude/skills/memory-onboard/SKILL.md` — token-bounded cold-repo onboarding
- [x] Tiers: cheap signals → QMD sampling → grill (`grill-with-memory`) → seed CONTEXT.md/ADR/project memory
- [x] Hard `--budget` source-file read cap (default 12)
- [x] Chaining: `memory-init` offers it; it invokes `grill-with-memory`; ends → suggest `/memory-curate`
- [x] Reviewed `memory-init`/`memory-write`/`memory-curate` (skill-creator pass) while building it

---

## Phase 6 — skill-creator review pass (D12) — DONE

- [x] `skill-creator` review of memory-init, memory-write, memory-curate, memory-onboard (done with D13)
- [x] `skill-creator` review of the 5 edited skills (one Sonnet subagent per skill:
  `capture-learning`, `handoff`, `grill-with-memory`, `setup-sdd-repo`, `repo-governance`) — done 2026-05-19; all spec-compliant after fixes
- [x] Record findings in `skill-creator-review-findings.md` (this directory)
- [x] `skill-stocktake` scan of all 9 — 2026-05-19: **9/9 Keep** (one Improve on
  `grill-with-memory` fixed in place — relative→absolute script paths). No
  retire/merge. Baseline saved to `~/.claude/skills/skill-stocktake/results.json`.
- [ ] (deferred to user) capture-learning Phase 4 dedup check — structural, see findings.
  NOTE: Phase 2 (LLM-wiki) rewrites `capture-learning` anyway — fold this in there.

---

## Phase 7 — End-to-end verification — DONE

See `VERIFICATION.md`.

- [x] `verify-memory.cjs` deterministic loop — **37/37 ALL GREEN** (2026-05-19)
- [x] Write → index → recall flow — probe fact written via `grill-with-memory`'s
  `write_memory_file.py`, confirmed as typed file + `MEMORY.md` pointer + `qmd
  search` hit (85%) + present in next-session `<memory-context>`; probe removed,
  `.memory/` left clean (2026-05-19)
- [x] Prompt-injection mitigation test — crafted entry with directive-shaped text
  (`IGNORE ALL PREVIOUS INSTRUCTIONS…`) in a `description` field; the SessionStart
  `<memory-context>` framing carries it only as a pointer-line description and the
  preamble explicitly labels description fields as DATA. Neutralized. Probe removed.
- [x] `code-reviewer` pass on the 4 hook/utility scripts (2026-05-19) — 0 CRITICAL,
  2 HIGH (coupled), 3 MEDIUM, 4 LOW. Fixed:
  - **HIGH** `session-start-memory.cjs` token budget excluded the framing wrapper —
    a full memory load could exceed 2,500 tokens and false-fail the verifier.
    Budget now reserves wrapper + truncation-marker length. (The coupled
    verify-memory.cjs HIGH is resolved by this fix — it measures the final string,
    which is now correctly capped; no verifier change needed.)
  - **MEDIUM** `qmd-refresh.cjs` — bare `--stale-minutes` (no value) yielded `NaN`
    and silently disabled the update; now falls back to the 10-minute default.
  - Deferred to Phase 2 (low-risk latent traps): `startsWith` path-boundary checks
    in both `findProjectMemory` impls; double `ignored()` subprocess calls in
    verify-memory.cjs; duplicated `qmdAvailable()` helper.
