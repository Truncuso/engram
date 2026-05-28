# Detailed Findings — Memory System Overhaul: LLM-Memory Architecture

**Date**: 2026-05-19
**Updated**: 2026-05-19
**Analysis Method**: Parallel Explore agents (3 agents: current memory system, obsidian-wiki repo, agentic-memory notes) + Plan agent (architecture design)
**Sources:** SRC-01 through SRC-12

---

## Corrections Summary

| Original Claim | Corrected Finding |
|----------------|-------------------|
| Phase 1: memory at `~/.claude/.memory/` | Moved to `~/memory/` as canonical; `~/.claude/.memory/` → symlink |
| Phase 1: 6 memory skills, no memory lifecycle | Phase 2 adds 35 adapted skills for full memory operations |
| Phase 1: No cron, manual-only curation | Phase 2 adds session-based cron + background daemon + LLM upkeep agent |

---

## Finding 1: Current Memory Infrastructure is Solid but Sparse

### 1.1 Current State

Phase 1 built a working system: typed `.md` files with frontmatter in `~/.claude/.memory/` (global) and `<repo>/.memory/` (project). QMD indexes both scopes. SessionStart/SessionEnd hooks load and refresh memory context. Six skills exist: memory-init, memory-write, memory-curate, memory-onboard, grill-with-memory, capture-learning. Infrastructure verified at 37/37 green.

However, only 1 user fact and 1 project fact exist. The system stores facts as isolated files with no cross-linking, provenance tracking, confidence scoring, lifecycle management, or visualization.

### 1.2 Structural Differences

| Aspect | Current | Target (obsidian-wiki) |
|--------|---------|------------------------|
| Storage | Typed files in user/, feedback/, reference/, daily/ | Memory vault categories: concepts/, entities/, skills/, references/, synthesis/, journal/, projects/ |
| Frontmatter | Minimal (name, description, type, scope, created) | Rich (title, category, tags, sources, summary, base_confidence, lifecycle, tier, provenance, relationships) |
| Linking | None (files are isolated) | [[wikilinks]] with typed relationships (extends, implements, contradicts, derived_from, uses, replaces, related_to) |
| Retrieval | QMD (BM25+semantic) | QMD + tiered retrieval (index pass → semantic → section grep → full read) |
| Maintenance | Manual (memory-curate) | Autonomous agent (QMD daemon + ingestion + curation) |
| Visualization | None | Obsidian graph view + graph-colorize + memory-export (JSON/GraphML/HTML) |
| Provenance | None | extracted/inferred/ambiguous markers + source quality scoring |
| Lifecycle | None | draft → reviewed → verified → disputed → archived |

### 1.3 Confirmed Gap

The infrastructure works but the content model is minimal. The system needs the full memory lifecycle: ingest pipelines, cross-linking, provenance, confidence, synthesis, digest, visualization, and autonomous upkeep.

**Impact**: Memory accumulates but doesn't compound. Facts are stored but can't be crossed-referenced, synthesized, or visualized. The system is half-built.

### 1.4 Relevant Code Paths

| File | Line(s) | Role |
|------|---------|------|
| `~/.claude/.memory/MEMORY.md` | 1–20 | Pointer-index (only 1 entry) |
| `~/.claude/.memory/user/terse-tool-outputs.md` | 1–15 | Only user fact |
| `~/.claude/scripts/hooks/session-start-memory.cjs` | 1–163 | SessionStart memory loader |
| `~/.claude/scripts/hooks/qmd-refresh.cjs` | 1–137 | QMD index refresh |
| `~/.claude/CLAUDE.md` | 175–189 | Memory override block |

---

## Finding 2: obsidian-wiki Provides a Complete, Battle-Tested Pattern

### 2.1 Current State

The obsidian-wiki repo at `SRC-01` is a mature MIT-licensed framework with 35 skills covering full memory lifecycle. It has been adapted for Claude Code, Cursor, Windsurf, Gemini CLI, Codex, Hermes, OpenClaw, Copilot CLI, and Kiro — proving the pattern is portable.

Key architectural patterns:
- **Config Resolution Protocol**: CWD-walk for `.env` → fallback to `~/.llm-memory/config`
- **Three-layer architecture**: Raw Sources (immutable) → Memory vault (LLM-maintained) → Schema (skill files + config)
- **Manifest system**: `.manifest.json` with SHA-256 content hashes for delta computation
- **Page template**: Rich frontmatter with provenance, confidence, lifecycle, tier, typed relationships
- **Retrieval Primitives**: Cost-ordered escalation (index → frontmatter grep → section grep → full read)
- **Setup**: `setup.sh` one-command install with symlink farm for multi-agent support

### 2.2 Confirmed Gap

We need to transplant this operational model onto our memory infrastructure. Not a fork — an adaptation. Our typed-file backend (QMD + hooks + CLAUDE.md tiering) stays; the memory lifecycle (35 skills, frontmatter schema, manifest tracking, autonomous upkeep) layers on top.

**Impact**: Direct adaptation rather than reinvention. Every design decision in obsidian-wiki was already battle-tested across 9+ agent platforms.

### 2.3 Relevant Code Paths

| File | Line(s) | Role |
|------|---------|------|
| `obsidian-wiki/.skills/llm-memory/SKILL.md` | 1–300+ | Architecture blueprint, page template, retrieval primitives |
| `obsidian-wiki/.skills/memory-setup/SKILL.md` | 1–200+ | Vault initialization (fuse with our memory-init) |
| `obsidian-wiki/AGENTS.md` | 1–200+ | Multi-agent bootstrap with skill routing table |
| `obsidian-wiki/setup.sh` | 1–300+ | One-command install with symlink farm |
| `obsidian-wiki/scripts/daily-update.sh` | 1–100+ | Cron-compatible maintenance script |

---

## Finding 3: Global↔Local Bridge (Hybrid D) Balances Search and Curation

### 3.1 Current State

Phase 1 established the principle: project scope exists at `<repo>/.memory/`, global scope at `~/.claude/.memory/`. Project facts win over global on conflict. But there was no mechanism for cross-referencing between scopes.

### 3.2 Resolution

Hybrid D model selected:
- `~/memory/_raw/projects/<name>/` → symlink to `<repo>/.memory/` (QMD-indexed, raw, immutable)
- `~/memory/projects/<name>/` → curated overview pages (written by memory-update, human-readable)

QMD indexes everything through _raw/ symlinks. Human-readable layer stays clean. Curated overview pages summarize what matters; raw data is always searchable.

### 3.3 Confirmed Gap

This pattern enables cross-scope synthesis (memory-synthesize can operate on both global concepts and project-specific entities) while maintaining clear provenance (curated vs raw).

**Impact**: Without a bridge, project knowledge is siloed. With Hybrid D, QMD search finds everything but the human-readable layer is curated.

### 3.4 Relevant Code Paths

| File | Line(s) | Role |
|------|---------|------|
| `~/memory/_raw/projects/` | (to be created) | Symlink farm to per-project vaults |
| `~/memory/projects/<name>/` | (to be created) | Curated project overview pages |

---

## Finding 4: Autonomous Upkeep Agent is the Missing Infrastructure Layer

### 4.1 Current State

Phase 1 relies on explicit manual triggers or hook-driven refresh. QMD index is refreshed at SessionStart (if stale) and SessionEnd. Memory write is always explicit. No background processing.

### 4.2 Resolution

Three-layer autonomous agent:
- **Layer 1 (QMD daemon)**: systemd timer, `qmd update` every 10 min, `qmd embed` debounced ≤1/hour. Zero LLM calls.
- **Layer 2 (ingestion agent)**: watches `_raw/` for new content, runs `memory-ingest` on changed sources, budget-aware retry with exponential backoff.
- **Layer 3 (curation agent)**: weekly lint+synthesize+staleness scan, monthly digest+dedup. All operations dry-run first.

Configurable LLM backend (Claude, Ollama, Codex, OpenAI) with `max_daily_tokens` and `max_monthly_cost_usd` budgets.

### 4.3 Confirmed Gap

Without autonomous upkeep, the memory vault rots: stale pages accumulate, new _raw/ content sits unprocessed, cross-links never get inserted. The agent makes the system self-maintaining.

**Impact**: The difference between a static fact store and a living, compounding knowledge base.

### 4.4 Relevant Code Paths

| File | Line(s) | Role |
|------|---------|------|
| `llm-memory/scripts/daemons/qmd-index-daemon.sh` | (to be created) | QMD refresh loop |
| `llm-memory/scripts/daemons/ingest-agent.sh` | (to be created) | Content ingestion wrapper |
| `llm-memory/scripts/daemons/curate-agent.sh` | (to be created) | Curation wrapper |
| `~/.llm-memory/config` | (to be created) | Agent backend + budget config |
