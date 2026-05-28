# Memory System Overhaul — Phase 2: LLM-Memory Architecture

**Date**: 2026-05-19
**Updated**: 2026-05-20
**Author**: cunger
**Status**: APPROVED — MVP milestone (WP0–WP5) audited & finalized 2026-05-19; WP6–WP16 remain PLANNING (Phase 2b). Skill YAML best-practices audit added (SKILL_YAML_REVIEW.md).
**Depends on**: Phase 1 (COMPLETE — all phases done, 37/37 verified, 2026-05-19)

---

## Executive Summary

Phase 1 built a working typed-file memory store (`~/.claude/.memory/` + `<repo>/.memory/`) with QMD retrieval, hooks, and 6 skills. It's solid infrastructure but nearly empty and lacks the full memory lifecycle: ingest pipelines, provenance tracking, cross-linking, synthesis, digesting, visualization, and agent-history mining.

The obsidian-wiki repo (`/home/cunger/10_Projects/Agentic AI/Workflows/automatic-workflows/obsidian-wiki/`, MIT license) provides a complete LLM-memory framework with 35 skills. This phase transplants that operational model onto our memory infrastructure.

**Key outcomes:**
- `~/memory/` becomes the canonical Obsidian vault with full memory vault category structure
- 35 skills adapted from obsidian-wiki, tiered by priority, each validated with skill-creator
- Autonomous upkeep agent (3 layers: QMD index daemon, ingestion agent, curation agent) with configurable LLM backend and budget controls
- Public framework repo (`cunger/llm-memory`) for sharing the system
- Per-project `.memory/` as independent Obsidian vaults with scope auto-discovery
- Script-first design: every skill backed by deterministic verification scripts

**Design Constraint — Script-First**: Every skill and workflow is backed by deterministic scripts, not ad-hoc agent behavior. Each skill has a verification script (pass/fail), an improvement loop, and deterministic workflows. Scripts are committed to the framework repo and CI-validated.

**Design Constraint — Skill-Creator Gate**: Every skill MUST pass `skill-creator` review before being considered complete. This validates frontmatter correctness, trigger phrases, description quality, and structural completeness. No skill is merged without a skill-creator pass. This applies to all ~24 adapted skills + all fused-mode skills.

**Design Constraint — Skill YAML Standard** (added 2026-05-20): Every skill SKILL.md must use the full Claude Code skill YAML feature set: `description` + `when_to_use` split, `paths` for vault-scoped gating, `disable-model-invocation` for destructive ops, `allowed-tools` for operational skills, `context: fork` + `agent: memory` for heavy processing, `model`/`effort` tiering, `arguments`/`argument-hint` for parameterized skills, and `${CLAUDE_SKILL_DIR}` for bundled script paths. The canonical reference is `SKILL_YAML_REVIEW.md`; the WP2 `llm-memory` skill defines the schema all other skills follow.

---

## Active Work Packages

| WP | Title | Severity | Status |
|----|-------|----------|--------|
| WP0 | Framework Git Repo Scaffold | HIGH | **MVP** — spec'd, ready |
| WP1 | Directory Migration + Vault Setup | HIGH | **MVP** — spec'd, ready |
| WP2 | `llm-memory` Core Architecture Skill | HIGH | **MVP** — spec'd, ready |
| WP3 | `memory-init` Overhaul (fuse with memory-setup) | HIGH | **MVP** — spec'd, ready |
| WP4 | Core Ingest + Query Skills | HIGH | **MVP** — spec'd, ready (OQ-10 resolved) |
| WP5 | Housekeeping Skills (lint, status, cross-link) | HIGH | **MVP** — spec'd, ready |
| WP6 | `memory-write` Overhaul | HIGH | Phase 2b — PLANNING (spec'd) |
| WP7 | `memory-curate` Overhaul | HIGH | Phase 2b — PLANNING (spec'd) |
| WP8 | Synthesis + Digest + Tag + Research + Ingest | MEDIUM | Phase 2b — PLANNING (spec'd) |
| WP9 | Agent History Ingest | MEDIUM | Phase 2b — PLANNING (spec'd) |
| WP10 | Visualization + Export | MEDIUM | Phase 2b — PLANNING (spec'd) |
| WP11 | Existing Skill Edits | MEDIUM | Phase 2b — PLANNING (spec'd) |
| WP12 | Cron Setup | MEDIUM | Phase 2b — PLANNING (spec'd) |
| WP13 | Framework Repo Finalization | MEDIUM | Phase 2b — PLANNING (spec'd) |
| WP14 | Autonomous Upkeep Agent | MEDIUM | Phase 2b — Spec'd + reviewed (OQ-8/9 resolved) |
| WP15 | E2E Verification + Documentation | HIGH | Phase 2b — Spec'd + reviewed |
| WP16 | Memory Agent (LLM-powered autonomous worker) | MEDIUM | Phase 2b — Spec'd (augments WP14) |

---

## MVP Milestone — WP0–WP5 (APPROVED 2026-05-19)

The plan is phased. The **first approved milestone is WP0–WP5**: the framework
repo, the vault migration, the `llm-memory` schema skill, the `memory-init`
overhaul, and the core operational skills (ingest, query, daily-update, lint,
status, cross-linker). This delivers an end-to-end usable memory vault — ingest a source,
query it, keep it healthy — and is independently verifiable.

**WP6–WP15 are deferred to Phase 2b** (lifecycle ops, synthesis/digest, history
ingest, visualization, cron, the autonomous upkeep agent, framework finalization,
full E2E). They remain in `PLANNING` — their WP files are fully spec'd and are to
be drafted to WP0/WP3 spec quality before that milestone is approved.

All six MVP WP files (`WP0`–`WP5`) carry full specifications: problem, target
files, step-by-step implementation, recommended agents, verification matrix.

### 35-skill → WP mapping (G2 reconciliation)

The "adapt ALL 35 obsidian-wiki skills" decision is reconciled against the WP
breakdown below. Some source skills become **standalone skills**; others are
folded as **modes** of an existing memory skill; a few are **dropped** with
rationale.

| Source skill | Disposition | WP |
|--------------|-------------|-----|
| `llm-memory` | standalone | WP-2 |
| `memory-setup` | fused into `memory-init` | WP-3 |
| `memory-ingest`, `memory-query`, `daily-update` | standalone | WP-4 |
| `memory-lint`, `memory-status`, `cross-linker` | standalone | WP-5 |
| `memory-capture` | fused into `memory-write` (CAPTURE mode) | WP-6 |
| `memory-update` | standalone (portable skill, global ~/.claude/skills/) | WP-6 |
| `memory-rebuild`, `memory-dedup`, `memory-stage-commit` | fused into `memory-curate` (REBUILD/DEDUP/STAGE modes) | WP-7 |
| `memory-synthesize`, `memory-digest`, `tag-taxonomy`, `memory-research`, `ingest-url`, `data-ingest` | standalone | WP-8 |
| `claude-history-ingest`, `memory-history-ingest`, `memory-history-search` | standalone | WP-9 |
| `codex-/copilot-/hermes-/openclaw-history-ingest` | **deferred — out of MVP+2b scope.** Only Claude history is on this machine; the other-agent variants are added if/when those agents are used. | — |
| `graph-colorize`, `memory-export`, `memory-dashboard`, `memory-bridge`, `memory-context-pack` | standalone | WP-10 |
| `memory-switch` | **dropped** — multi-vault switching; our scope-discovery rule (WP-2) replaces it. | — |
| `obsidian-memory-ingest` | **dropped** — obsidian-memory-specific bootstrap; `memory-init` (WP-3) covers our equivalent. | — |
| `impl-validator` | **deferred to 2b review** — value unclear for our use; decide during WP-11. | — |
| `skill-creator` | not ported — already a global dev tool in `~/.claude/skills/`. | — |

**Net:** "all 35" resolves to ~24 adapted skills (standalone + fused), 4
history-ingest variants deferred, 3 dropped with rationale, 1 (`skill-creator`)
already present, plus `impl-validator` pending a 2b decision. The OVERVIEW
"35 skills" phrasing should be read as "the full obsidian-wiki skill set,
reconciled" — not 35 new standalone skills.

### Audit residuals (2026-05-19 finalization)

The MVP audit surfaced two items beyond the earlier architect/plan review:

- **G3 — public framework repo secrets hygiene.** WP0 creates a *public* GitHub
  repo. Before `gh repo create … --public` (WP0 Step 8) and before WP13's
  finalization, a deliberate secrets scan must confirm no real config, tokens, or
  sensitive memory content leak via the committed skills or `config/`. The
  `.env.example` pattern is correct; the scan is to catch anything else. To be
  enforced as a WP0 verification step and re-checked in WP13.
- **G4 — Phase 1 deferred code-review findings.** Phase 1's hook code-review
  deferred three latent items here: the `startsWith` path-boundary check in
  `findProjectMemory` (both the hook and `verify-memory.cjs` copies), the double
  `ignored()` subprocess calls in `verify-memory.cjs`, and the duplicated
  `qmdAvailable()` helper. WP-1 Step 4 folds in the `startsWith` fix; the
  remaining two are to be addressed in WP-3 (which rewrites `memory-init` and its
  verification script) — added as an explicit WP-3 sub-task.

---

## Archived / Closed / Deleted

| WP | Outcome |
|----|---------|
| (none yet) | |

---

## Corrections Log

| Previous Claim | Corrected Finding |
|----------------|-------------------|
| Phase 1: memory at `~/.claude/.memory/` | Moved to `~/memory/` as canonical; `~/.claude/.memory/` → symlink |
| Phase 1: 6 memory skills, no memory lifecycle | Phase 2 adds 35 adapted skills for full memory operations |
| Phase 1: No cron, manual-only curation | Phase 2 adds session-based cron + background daemon + LLM upkeep agent |
| Phase 1: Single QMD collection per scope | Phase 2: per-vault QMD collections for namespace isolation |

---

## Execution Strategy

```
Phase 0: WP0 (framework repo scaffold) ← FIRST — becomes canonical skill source

Phase 1: WP1 (Migration) → WP2 (llm-memory) → WP3 (memory-init)
         → WP4 (ingest+query+daily) → WP5 (lint+status+cross-link)

Phase 2: WP6 (memory-write) → WP7 (memory-curate)
         → WP8 (synthesize+digest+tag+research+ingest)

Phase 3: WP9 (history ingest)

Phase 4: WP10 (visualization+export)

Phase 5: WP11 (existing skill edits) + WP12 (cron) + WP13 (framework finalize)

Phase 6: WP14 (upkeep agent daemons) + WP15 (E2E verification + docs)
```

WP11 and WP12 can run in parallel with WP9/WP10.
WP14 depends on core skills (WP2 through WP5) and WP12 (cron setup).

---

## Key Decisions

| Decision | Rationale | Date |
|----------|-----------|------|
| `~/memory/` is canonical global vault; `~/.claude/.memory/` → symlink | Accessibility for other agents; Obsidian opens `~/memory/` directly | 2026-05-19 |
| Adapt ALL 35 obsidian-wiki skills, tiered by priority | Full lifecycle coverage; each WP uses skill-creator for quality | 2026-05-19 |
| Session-based cron (CronCreate) for daily/weekly/monthly maintenance | Transparent, auditable, no system crontab | 2026-05-19 |
| Per-project `.memory/` mirrors global memory vault structure same structure as global vault | Uniform tooling | 2026-05-19 |
| Framework git repo (public) is CANONICAL skill source | Git-tracked, symlinked into agent dirs — obsidian-wiki pattern | 2026-05-19 |
| `~/memory/_raw/` holds symlinks to plans directories | Design rationale becomes QMD-searchable | 2026-05-19 |
| Global + per-project `.obsidian/` are independent | Each vault has its own graph visualization | 2026-05-19 |
| Skills auto-discover scope from CWD | No manual flags for operational skills | 2026-05-19 |
| Hybrid D: `_raw/projects/<name>/` → symlink to `<repo>/.memory/` | QMD indexes everything; human-readable layer is curated | 2026-05-19 |
| Per-vault QMD collections | Namespace isolation, precise scoping | 2026-05-19 |
| Autonomous memory upkeep agent with configurable LLM backend | Watches _raw/, ingests, archives. Retry logic with budget controls | 2026-05-19 |
| `_raw/` is immutable source material; `_archive/` holds stale pages | Clean separation: raw → memory vault → archive | 2026-05-19 |
| Phase 1 content migrates via direct mapping: user→entities/, feedback→references/ | Clean, predictable migration path | 2026-05-19 |
| Skills split: portable (global, 4 skills) vs vault-scoped (project-local, 31 skills) | Portable: memory-init, memory-write, memory-query, memory-update — work from any project. Vault-scoped: all ingest/lint/synthesize/graph skills — only in vault context | 2026-05-19 |
| Multi-agent support: 12 agent platforms via symlink farm + agent-specific config files | Claude Code, Cursor, Windsurf, Gemini CLI, Codex, Hermes, OpenClaw, Copilot CLI, Trae, Kiro, Antigravity, AGENTS.md-generic. Each gets skill symlinks + context bootstrap | 2026-05-19 |
| Framework repo is the single source of truth for all skills, templates, scripts, configs, and agent bootstraps | One `git clone` + `setup.sh` configures all agents. Skills edited in one place; all agents see changes | 2026-05-19 |
| Framework repo `CLAUDE.md` (memory bootstrap) is separate from user's `~/.claude/CLAUDE.md` (memory override block) | Two files, different purposes: framework CLAUDE.md has routing table + principles for agents IN the repo; user's CLAUDE.md has memory override redirecting to `~/memory/` | 2026-05-19 |
| Config at `~/.llm-memory/config` (not `~/.obsidian-wiki/config`) | Our own namespace. Contains `LLM_MEMORY_VAULT_PATH` and `LLM_MEMORY_REPO`. Independent of obsidian-wiki tooling. | 2026-05-19 |
| `memory-init --project` creates per-project skill symlinks to framework repo | When inside a project with `.memory/`, all 35 memory skills are locally available via `.claude/skills/`, `.cursor/skills/` etc. relative symlinks | 2026-05-19 |

---

## Multi-Agent Support

The framework supports 12 agent platforms using the obsidian-wiki pattern: a single `setup.sh` creates symlinks and config files for every agent. Each agent discovers skills in its own way, but all point to the same canonical `.skills/` directory in the framework repo.

### Agent Bootstrap Mechanisms

| Agent | Bootstrap File(s) | Skill Discovery | Scope |
|-------|-------------------|-----------------|-------|
| **Claude Code** | `CLAUDE.md` | `.claude/skills/` (project-local), `~/.claude/skills/` (global, portable only) | Both |
| **Cursor** | `.cursor/rules/llm-memory.mdc` (alwaysApply) | `.cursor/skills/` (project-local symlinks) | Project |
| **Windsurf** | `.windsurf/rules/llm-memory.md` (always-on) | `.windsurf/skills/` (project-local symlinks) | Project |
| **Gemini CLI** | `GEMINI.md` | `~/.gemini/skills/` (global, all skills) | Global |
| **Antigravity** | `GEMINI.md` + `.agent/rules/llm-memory.md` (alwaysApply) + `.agent/workflows/llm-memory.md` (slash commands) | `~/.gemini/antigravity/skills/` + `.agents/skills/` | Both |
| **Codex** | `AGENTS.md` | `~/.codex/skills/` (global, all skills) | Global |
| **Hermes** | `.hermes.md` → `AGENTS.md` (symlink) | `~/.hermes/skills/` (global, all skills) | Global |
| **OpenClaw** | `AGENTS.md` | `~/.openclaw/skills/` + `~/.agents/skills/` (shared) | Both |
| **Copilot CLI** | `.github/copilot-instructions.md` (VS Code) | `~/.copilot/skills/` (global, all skills) | Both |
| **Trae** | `AGENTS.md` | `~/.trae/skills/` (global, all skills) | Global |
| **Kiro** | `.kiro/steering/llm-memory.md` (inclusion: always) | `.kiro/skills/` (project-local) + `~/.kiro/skills/` (global) | Both |
| **Generic** (OpenCode, Aider, Factory Droid) | `AGENTS.md` | `.agents/skills/` (project-local) + `~/.agents/skills/` (global) | Both |

### Portable vs Vault-Scoped Skills

Following the obsidian-wiki pattern, only skills useful from arbitrary projects are installed globally:

**Portable (global, 4 skills)** — installed in `~/.claude/skills/` and other global agent dirs:
- `memory-init` — scaffold global vault or per-project `.memory/`
- `memory-write` — save facts from any project context
- `memory-query` — search memory from any project
- `memory-update` — sync project knowledge into vault from any project

**Vault-scoped (local, 31 skills)** — available when CWD is inside a vault (global `~/memory/` or project `.memory/`):
- All ingest skills (memory-ingest, ingest-url, data-ingest, claude-history-ingest, memory-history-ingest, memory-history-search)
- All housekeeping skills (memory-lint, memory-status, cross-linker, memory-synthesize, memory-dedup, tag-taxonomy)
- All lifecycle skills (memory-curate, memory-rebuild, memory-stage-commit)
- All visualization skills (graph-colorize, memory-export, memory-dashboard, memory-bridge)
- All digest/report skills (memory-digest, daily-update, memory-research, memory-capture, memory-context-pack)
- Utility (memory-switch, obsidian-memory-ingest, impl-validator)

### Agent Config Files (in framework repo)

Each agent gets a context bootstrap file in the framework repo. On `setup.sh`, project-local configs are created as symlinks from the repo; global configs are installed to `~/.<agent>/`.

```
llm-memory/
├── .skills/                          # Canonical skill source (symlinked by all agents)
├── agent-config/                     # Agent-specific context bootstraps
│   ├── claude/
│   │   ├── CLAUDE.md                 # Bootstrap for Claude Code (symlinked to repo root)
│   │   └── .claude/skills/           # Project-local → ../../.skills/ (relative symlinks)
│   ├── cursor/
│   │   └── .cursor/rules/llm-memory.mdc  # alwaysApply: true
│   ├── windsurf/
│   │   └── .windsurf/rules/llm-memory.md # activation: always-on
│   ├── gemini/
│   │   ├── GEMINI.md                 # Gemini CLI bootstrap
│   │   └── .agent/rules/llm-memory.md    # Antigravity alwaysApply
│   ├── copilot/
│   │   └── .github/copilot-instructions.md
│   ├── kiro/
│   │   └── .kiro/steering/llm-memory.md  # inclusion: always
│   └── generic/
│       └── AGENTS.md                 # Codex, Hermes, OpenClaw, Trae, OpenCode, Aider, Droid
├── scripts/
├── templates/
├── config/
├── scripts/setup.sh
└── Makefile
```

Each agent config file contains the same core content adapted to the agent's format:
1. **Config Resolution Protocol** — walk CWD for `.env`, fallback to `~/.llm-memory/config`
2. **Skill Routing Table** — maps user intents to skill names
3. **Core Principles** — compile don't retrieve, track everything, connect with wikilinks, frontmatter required
4. **Vault Structure Reference** — category layout, special files
5. **Scope Auto-Discovery** — project `.memory/` vs global `~/memory/`

### setup.sh Multi-Agent Installation

```bash
setup.sh behavior:
1. Detect agent type (optional — installs for all by default)
2. Write ~/.llm-memory/config with vault path + framework repo path
3. Create .env from .env.example (if missing)
4. Bootstrap AGENTS.md aliases (CLAUDE.md → AGENTS.md, GEMINI.md → AGENTS.md, .hermes.md → AGENTS.md)
5. Project-local symlinks (relative):
   - .claude/skills/* → ../../.skills/* (all 35 skills)
   - .cursor/skills/* → ../../.skills/*
   - .windsurf/skills/* → ../../.skills/*
   - .agents/skills/* → ../../.skills/*
   - .kiro/skills/* → ../../.skills/*
6. Project-local config symlinks:
   - CLAUDE.md → AGENTS.md
   - .cursor/rules/llm-memory.mdc → agent-config/cursor/.cursor/rules/llm-memory.mdc
   - etc.
7. Global symlinks (absolute):
   - ~/.claude/skills/ → {memory-init, memory-write, memory-query, memory-update} only (portable)
   - ~/.gemini/skills/* → all 35 skills
   - ~/.codex/skills/* → all 35 skills
   - ~/.hermes/skills/* → all 35 skills
   - ~/.openclaw/skills/* → all 35 skills
   - ~/.copilot/skills/* → all 35 skills
   - ~/.trae/skills/* → all 35 skills
   - ~/.trae-cn/skills/* → all 35 skills
   - ~/.kiro/skills/* → all 35 skills
   - ~/.agents/skills/* → all 35 skills
8. Install hook scripts (Claude Code only):
   - ~/.claude/scripts/hooks/session-start-memory.cjs
   - ~/.claude/scripts/hooks/session-end-memory.cjs
   - ~/.claude/scripts/hooks/qmd-refresh.cjs
   - Register in ~/.claude/hooks/hooks.json
9. Run QMD collection registration
10. Create CronCreate jobs (Claude Code only)
11. Install systemd timers (QMD index daemon, Linux) or launchd plists (macOS)
12. Print summary
```

---

## Project Sources

| SRC | Title | Type | Relevance |
|-----|-------|------|-----------|
| SRC-01 | obsidian-wiki repo | GIT | All 35 skills, architecture patterns, templates |
| SRC-02 | Phase 1 overhaul plan (`planning-2026-05-17-unified-memory/`) | FILE | Existing infrastructure, 37/37 verified |
| SRC-03 | QMD-vs-MemSearch Analysis | FILE | Architecture decision: QMD over memsearch |
| SRC-04 | Karpathy LLM Memory gist | URL | Conceptual foundation for persistent memory vault pattern |
| SRC-05 | Current memory infrastructure (hooks, QMD, CLAUDE.md) | FILE | Baseline state |

---

## Review Findings (2026-05-19 — architect-reviewer + plan-reviewer)

Issues found and resolved during plan review:

### Architectural Fixes

**AF-1: Hybrid D — Exclude vault-internal directories from symlinks.** `~/memory/_raw/projects/<name>/` symlinks to `<repo>/.memory/` but MUST exclude `.obsidian/`, `.trash/`, and `.git/` subdirectories. Nested `.obsidian/` dirs inside a parent vault cause Obsidian graph view to fracture. Implementation: the symlink is to the vault root, but QMD is configured with exclude patterns. `memory-ingest` and `cross-linker` skip these paths.

**AF-2: WP0→WP3 bootstrapping requires atomic transition.** WP0 moves existing skills from `~/.claude/skills/` into the framework repo. Between WP0 and WP3 (memory-init overhaul), the old skill paths are gone but new symlinks aren't established. Resolution: WP0 creates the framework repo and copies skills (not moves). WP3 establishes the symlink farm, then removes originals. During the transition, both paths work.

**AF-3: Dotfiles git repo migration path.** `~/.claude/.memory/` is tracked in `~/dotfiles/claude/` (git repo). Moving to `~/memory/` requires: (1) `git rm --cached memory/` from dotfiles repo, (2) move contents to `~/memory/`, (3) add `~/memory/` to a new gitignore or track it separately, (4) create symlink `~/.claude/memory → ~/memory/`. WP1 steps updated accordingly.

**AF-4: Per-vault QMD multi-collection design.** `qmd-refresh.cjs` must enumerate all registered QMD collections from a config file (`_meta/qmd-collections.json`) rather than hardcoding one path. `memory-init` writes this file during scaffold. The refresh loop iterates all collections with the same staleness gate and debounce.

**AF-5: QMD daemon + SessionStart hook interaction.** The daemon refreshes every 10 min (no-LLM `qmd update`). The SessionStart hook checks staleness of QMD index mtime — if the daemon refreshed within the last 10 min, the hook skips its own refresh. No race condition: the daemon writes an `_meta/.qmd-last-refresh` timestamp that both processes read.

### Spec Fixes

**SF-1: OQ-8/OQ-9 block WP14; OQ-10 blocks WP4.** Statuses changed from Pending to Blocked in Active Work Packages table. WPs 0-3, 5-13, 15 can proceed.

**SF-2: Acceptance criteria count corrected.** Added 2 ACs (memory-write frontmatter completeness, memory-curate mode support) — now 16 total.

**SF-3: Manual tests flagged for script replacement.** V2.2, V3.7, V4.6, V5.5, V6.2, V6.3, V7.1, V7.2 marked as `[MANUAL — needs script]` in VERIFICATION.md. These will be converted to deterministic scripts during their respective WP implementation phases.

**SF-4: Missing WP9-13 verification sections added.** Placeholder sections created in VERIFICATION.md.

**SF-5: MEMORY.md → index.md transition specified.** The Phase 1 `MEMORY.md` (manual pointer-index) is superseded by `index.md` (auto-generated page catalog rebuilt by `daily-update` from all page frontmatter). The SessionStart hook load list is updated in WP3 Phase 7: global `index.md` + `hot.md`, project `index.md` + `hot.md`, ≤2,500 tokens, `<memory-context>` framing, project-overrides-global precedence. `hot.md` provides identity + recency context, absorbing the old `USER.md` and daily log roles. No functionality is lost — every original Implementation Guide feature has an equivalent or superior replacement in the memory model.

## Acceptance Criteria

- [ ] `memory-init --global` creates full `~/memory/` vault, idempotent on re-run
- [ ] `memory-init --project` creates `<repo>/.memory/` with `.obsidian/`, idempotent
- [ ] `memory-ingest` processes source → page with full frontmatter + >=2 wikilinks
- [ ] `memory-query` retrieves known facts via tiered pipeline
- [ ] `memory-lint` detects all 13 issue types on a test vault
- [ ] `cross-linker` detects and inserts unlinked mentions
- [ ] `memory-synthesize` discovers co-occurrence opportunities
- [ ] `memory-digest` generates correct output for known period
- [ ] QMD index daemon runs via systemd timer
- [ ] Ingestion agent processes new _raw/ content automatically
- [ ] Curation agent runs weekly lint + synthesize, monthly digest + dedup
- [ ] SessionStart hook loads memory ≤ 2,500 tokens
- [ ] `memory-write` writes with full memory frontmatter (provenance, confidence, lifecycle, tier)
- [ ] `memory-curate` supports REBUILD, STAGE, and DEDUP modes
- [ ] `make verify` passes all verification scripts
- [ ] Framework repo CI validates setup.sh idempotency
