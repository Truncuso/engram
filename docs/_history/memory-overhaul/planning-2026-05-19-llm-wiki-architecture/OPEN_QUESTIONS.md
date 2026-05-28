# Open Questions — Memory System Overhaul: LLM-Memory Architecture

**Date**: 2026-05-19
**Updated**: 2026-05-19

---

## Active Questions

_None open. All planning questions resolved — see the Resolution Log._

---

## Resolution Log

| ID | Question | Resolution | Resolved By | Date |
|----|----------|------------|-------------|------|
| OQ-1 | Global↔Local memory relationship | Hybrid D: `_raw/projects/<name>/` → symlink for QMD indexing; `projects/<name>/` curated pages | cunger | 2026-05-19 |
| OQ-2 | Phase 1 content migration strategy | Direct mapping: user→entities/, feedback→references/, reference→references/, MEMORY.md→index.md | cunger | 2026-05-19 |
| OQ-3 | One QMD collection or many? | Per-vault: `memory-global` + `memory-<project>` per project | cunger | 2026-05-19 |
| OQ-4 | Per-project `.obsidian/` shared or independent? | Independent: each `<repo>/.memory/.obsidian/` is standalone | cunger | 2026-05-19 |
| OQ-5 | `_raw/` symlink policy | Plans + session transcripts + project vaults. All `_raw/` gitignored (symlinks only) | cunger | 2026-05-19 |
| OQ-6 | Cron reliability mechanism | Session-based + background daemon + LLM upkeep agent | cunger | 2026-05-19 |
| OQ-7 | Cross-scope precedence and synthesis rules | Flag cross-scope contradictions, enable cross-scope synthesis, mark superseded global pages | cunger | 2026-05-19 |
| OQ-10 | Ingestion idempotency | Append-mode default with two-hash manifest; `--full` flag for overwrite. Spec'd in WP-4. | cunger | 2026-05-19 |
| OQ-11 | `~/memory/` git repo? | **Yes, private git repo.** `git init ~/memory/` with `.gitignore` for `daily/`, `_archive/`, `.obsidian/workspace*.json`. Version history without public exposure. | cunger | 2026-05-19 |
| OQ-8 | Upkeep agent implementation (detection / process model / offline fallback) | **(a) Periodic scan** — timer-driven script walks `_raw/`, SHA-256 vs `.manifest.json`, ingests changed sources. No resident daemon (matches the session-cron model). **(b) Two separate cron jobs** — ingestion and curation run independently, separate schedules + budgets + failure domains. **(c) Queue-with-backoff** — offline backend or over-budget → log + queue the work, retry with exponential backoff up to a max-attempts cap. The QMD index daemon (no-LLM) keeps running regardless. Spec'd in WP-14. | cunger | 2026-05-19 |
| OQ-9 | Budget enforcement (token counting / overflow) | **Trust API counts, estimate local.** API backends (Claude/OpenAI): use the response-reported usage. Local backends (Ollama): estimate via chars/4 on prompt+completion. `max_monthly_cost_usd` gates paid API backends only (local is cost-free → only the token budget gates it). Overflow → queue-with-backoff (per OQ-8c). Spec'd in WP-14. | cunger | 2026-05-19 |

---

## Escalation Protocol

If any question remains unresolved after 2 rounds of clarification:

1. Halt implementation.
2. Escalate to the project maintainer with:
   - The unresolved question(s)
   - Impact on dependent WPs
   - Proposed resolution or decision checkpoint (Option 1 / Option 2)
3. Do not proceed past Spec until resolved.
