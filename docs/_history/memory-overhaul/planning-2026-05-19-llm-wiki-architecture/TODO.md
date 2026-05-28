# TODO.md — Memory System Overhaul: LLM-Memory Architecture

**Date**: 2026-05-19
**Updated**: 2026-05-19
**Status**: MVP milestone (WP0–WP5) APPROVED 2026-05-19; WP6–WP15 PLANNING (Phase 2b)

---

## Active Set

> **IMMEDIATE PLANNING POINT (2026-05-21) — before executing WP4/WP14.** The
> graph-store choice (Kuzu embedded DB) was made before two alternatives were
> on the table. Compare, with the user, before any WP that builds graph
> machinery: **Kuzu** (`github.com/kuzudb/kuzu`) vs **graphify**
> (`github.com/safishamsi/graphify` — already assumed installed; indexes
> `.md .mdx .qmd .html .txt .yaml .docx` etc. + provides MCP graph search) vs
> **agentmemory** (`github.com/rohitg00/agentmemory`). Open question: could
> `agentmemory` alone replace the QMD+Kuzu combination, while still allowing an
> LLM-wiki-style knowledge base — and could planning + session knowledge be
> linked into the same store? See OQ-12. Tracked for a dedicated review session.

**MVP milestone = WP0–WP5 (APPROVED, fully spec'd). WP6–WP15 = Phase 2b — all
fully spec'd & plan-reviewed (2026-05-19); pending milestone approval.**

| WP | Title | Severity | Milestone | Status / Next Action |
|----|-------|----------|-----------|----------------------|
| WP0 | Framework Git Repo Scaffold | HIGH | MVP | Spec'd — ready to execute |
| WP1 | Directory Migration + Vault Setup | HIGH | MVP | Spec'd — ready (OQ-11 resolved: private git repo) |
| WP2 | llm-memory Core Architecture Skill | HIGH | MVP | Spec'd — ready to execute |
| WP3 | memory-init Overhaul | HIGH | MVP | Spec'd — ready to execute |
| WP4 | Core Ingest + Query Skills | HIGH | MVP | Spec'd — ready (OQ-10 resolved: append-mode) |
| WP5 | Housekeeping Skills | HIGH | MVP | Spec'd — ready to execute |
| WP6 | memory-write Overhaul | HIGH | 2b | Spec'd + reviewed — ready |
| WP7 | memory-curate Overhaul | HIGH | 2b | Spec'd + reviewed — ready |
| WP8 | Synthesis + Digest + Tag + Research + Ingest | MEDIUM | 2b | Spec'd + reviewed — ready |
| WP9 | Agent History Ingest | MEDIUM | 2b | Spec'd + reviewed — ready |
| WP10 | Visualization + Export | MEDIUM | 2b | Spec'd + reviewed — ready |
| WP11 | Existing Skill Edits | MEDIUM | 2b | Spec'd + reviewed — ready |
| WP12 | Cron Setup | MEDIUM | 2b | Spec'd + reviewed — ready |
| WP13 | Framework Repo Finalization | MEDIUM | 2b | Spec'd + reviewed — ready |
| WP14 | Autonomous Upkeep Agent | MEDIUM | 2b | Spec'd + reviewed — ready (OQ-8/9 resolved) |
| WP15 | E2E Verification + Documentation | HIGH | 2b | Spec'd + reviewed — ready |
| WP16 | Memory Agent (LLM-powered autonomous worker) | MEDIUM | 2b | Spec'd — ready |

---

## Phase 1: Planning & Research

- [x] Analyze current state of Phase 1 memory overhaul (85% complete, 37/37 verified)
- [x] Explore obsidian-wiki repo (35 skills, architecture patterns, templates)
- [x] Explore agentic-memory notes (QMD-vs-MemSearch, Karpathy LLM Memory, context graphs)
- [x] Survey current infrastructure (hooks, QMD collections, CLAUDE.md, skills)
- [x] Design architecture: vault structure, symlink topology, scope auto-discovery
- [x] Make 15 key design decisions
- [x] Tier 35 skills into 5 priority levels
- [x] Define 16 work packages with dependencies
- [x] Write OVERVIEW.md, FINDINGS.md, SOURCES.md, VERIFICATION.md
- [x] Resolve 7 open questions with user
- [x] Launch review agents on plan (architect-reviewer + plan-reviewer → AF-1..5, SF-1..4)
- [x] MVP audit + finalization (2026-05-19): WP0–WP5 spec'd to full quality;
  OQ-10 resolved (append-mode); OQ-11 raised (vault git-tracking); G3/G4 recorded
- [x] Phase the plan: MVP = WP0–WP5; WP6–WP15 deferred to Phase 2b

### Open Questions

| ID | Question | Blocks | Status |
|----|----------|--------|--------|
| OQ-8 | Upkeep agent implementation details (inotify vs periodic scan, session architecture) | WP14 (Phase 2b) | **Resolved 2026-05-19** — periodic scan, two separate cron jobs, queue-with-backoff |
| OQ-9 | Budget enforcement mechanism (token counting, cost tracking, overflow behavior) | WP14 (Phase 2b) | **Resolved 2026-05-19** — trust API counts, estimate local, overflow→queue-with-backoff |
| OQ-10 | Ingestion idempotency (overwrite vs merge on re-ingestion) | WP4 | **Resolved 2026-05-19** — append-mode default, two-hash manifest, `--full` opt-in |
| OQ-11 | Should `~/memory/` be its own git repo, or un-tracked? | WP1 | **Resolved 2026-05-19** — Yes, private git repo with .gitignore for daily/, _archive/, .obsidian/workspace*.json |
| OQ-12 | Graph/retrieval store: keep QMD+Kuzu, or adopt graphify, or adopt agentmemory (possibly replacing the combination)? Must still support an LLM-wiki knowledge base and link planning + session knowledge. | WP4, WP14, WP10 | **OPEN (2026-05-21)** — dedicated review session needed; compare Kuzu vs graphify vs agentmemory before building graph machinery |

**MVP (WP0–WP5):** OQ-10 resolved; OQ-11 resolved (private git repo). All MVP WPs unblocked.
**Phase 2b (WP6–WP15):** OQ-8/OQ-9 resolved (2026-05-19); WP14 unblocked.

---

## Phase 2: Implementation (Phase 0: Foundation)

- [ ] WP0: Create framework repo at `~/10_Projects/Agentic AI/Workflows/automatic-workflows/llm-memory`
- [ ] WP0: git init, scaffold directory structure, copy setup.sh from obsidian-wiki
- [ ] WP0: Push to GitHub (`gh repo create cunger/llm-memory --public`)
- [ ] WP0: Move existing memory skills into repo, symlink back to `~/.claude/skills/`

---

## Phase 3: Implementation (Phase 1: Core Infrastructure)

- [ ] WP1: Backup and migrate `~/.claude/.memory/` → `~/memory/`
- [ ] WP1: Create full vault directory structure with all categories
- [ ] WP1: Create symlinks (`~/.claude/memory → ~/memory/`, `_raw/plans/ → ~/.claude/plans/`)
- [ ] WP1: Write `~/.llm-memory/config`
- [ ] WP1: Update hook scripts for new paths, re-register QMD
- [ ] WP2: Port `llm-memory/SKILL.md` from obsidian-wiki
- [ ] WP2: Adapt paths, add scope resolution, preserve full frontmatter schema
- [ ] WP2: skill-creator scaffold + review
- [ ] WP3: Rewrite `memory-init/SKILL.md` (12-phase idempotent setup)
- [ ] WP3: skill-creator rewrite + review
- [ ] WP4: Port memory-ingest, memory-query, daily-update
- [ ] WP4: skill-creator 2 passes each (6 total)
- [ ] WP5: Port memory-lint, memory-status, cross-linker
- [ ] WP5: skill-creator 2 passes each (6 total)

---

## Phase 4: Implementation (Phase 2: Lifecycle Operations)

- [ ] WP6: Rewrite memory-write (memory frontmatter, CAPTURE mode, UPDATE mode)
- [ ] WP7: Rewrite memory-curate (REBUILD, STAGE, DEDUP modes)
- [ ] WP8: Port memory-synthesize, memory-digest, tag-taxonomy, memory-research, ingest-url, data-ingest

---

## Phase 5: Implementation (Phase 3–6: Advanced Features)

- [ ] WP9: Port memory-history-ingest, claude-history-ingest, memory-history-search
- [ ] WP10: Port graph-colorize, memory-export, memory-dashboard, memory-bridge, memory-context-pack
- [ ] WP11: Update 5 existing skills for new paths + memory conventions
- [ ] WP12: Create 3 CronCreate jobs: daily (9:57), weekly (Sat 10:07), monthly (1st 9:17)
- [ ] WP13: Finalize framework repo: README, AGENTS.md, CI, tag v1.0.0
- [ ] WP14: Build QMD index daemon, ingestion agent script, curation agent script
- [ ] WP14: Install systemd timers via memory-init

---

## Phase 6: Verification & Review

- [ ] WP15: Extend `verify-e2e.sh` for full memory pipeline
- [ ] WP15: Run full E2E flow (ingest → query → lint → cross-link → synthesize → digest)
- [ ] WP15: Verify cron jobs, hook token budget, per-project scaffolding
- [ ] WP15: Write migration guide
- [ ] WP15: Run `skill-stocktake` Quick Scan on all skills
- [ ] Run `make verify` — all scripts pass
- [ ] Launch `code-reviewer` agent on all skill + hook changes
- [ ] Archive completed WPs
