# Verification Matrix — Memory System Overhaul

**Date**: 2026-05-17
**Updated**: 2026-05-18 — work packages archived; verification is now per-deliverable.

> Consolidated verification index. Runnable test steps live in
> `memory-init-implementation-guide.md` §7.

---

## Build Verification Gates (All deliverables)

This is a skill/markdown project, not Java. The standard build gates from the template do not apply. Instead:

| Gate | Command | Expected |
|------|---------|----------|
| Skill structure valid | `ls ~/.claude/skills/<name>/SKILL.md` | File exists with `name`+`description` frontmatter |
| `skill-stocktake` Quick Scan | Invoke `skill-stocktake` on changed/created skill | PASS — no broken triggers, no missing references |
| Markdown well-formed | `head -20` on every new SKILL.md | Valid frontmatter; no leftover template placeholders (double-brace markers) |
| Session load budget | Fresh subagent in test repo | Total memory load ≤ 2,500 tokens |
| Prompt-injection mitigation | Crafted memory entry test | `<memory-context>` framing prevents instruction execution |

---

## Static Guardrails

Architectural invariants that hold across all deliverables:

| Invariant | Where Enforced |
|-----------|----------------|
| Memory file frontmatter has `name`, `description`, `type`, `scope` keys | `memory-init` templates + `memory-write` + `capture-learning` |
| `MEMORY.md` contains only pointer lines, not content | `memory-init` template + index-update logic in all writers |
| Project memory wins over global on conflict | Every MEMORY.md header + CLAUDE.md override block + hook framing |
| `daily/` + `_archive/` never committed | `memory-init` `.gitignore` write |
| Typed subdirs (`user/`, `feedback/`, `project/`, `reference/`) committable | `memory-init` `.gitignore` does NOT add them |
| Cross-project memory cannot leak | `session-start-memory.cjs` uses `$PWD`-bounded resolution |
| External `memsearch` dependency NOT installed | Out of scope; no install step anywhere |
| No raw transcript capture | Out of scope; no Stop-hook capture script created |
| No cron jobs created | Out of scope; `memory-curate` is manual-only |

---

## Per-Deliverable Verification

> Work packages were archived 2026-05-18. Verification is now per-deliverable.
> The runnable test steps live in `memory-init-implementation-guide.md` §7.

| Deliverable | Verified by |
|-------------|-------------|
| D1 `memory-init` | Guide §7 steps 1–4, 11; idempotent re-run; legacy migration preserves originals |
| D2 `memory-write` | Guide §2 test cases (6); guide §7 steps 6–8 |
| D3 `memory-curate` | Guide §3 phases; guide §7 step 10 |
| D4–D8 skill edits | `skill-stocktake` Quick Scan per skill; guide §4 behaviors |
| D9 hook | Guide §7 steps 5, 12; `node -c` syntax check; token budget ≤ 2,500 |
| D10 CLAUDE.md override | Guide §7 steps 1–2; block added once, idempotent |
| D12 skill-creator review | `skill-creator-review-findings.md` recorded; stocktake passes for all 8 |

---

## End-to-End Verification

See `memory-init-implementation-guide.md` §7 for the 12-step matrix. Run in a fresh subagent.

| Step | Action | Expected Observation |
|------|--------|---------------------|
| 1 | `/memory-init --global` | `~/.claude/.memory/` tree exists with templates filled |
| 2 | `cd /home/cunger/dotfiles/claude && /memory-init` | `<repo>/.memory/` tree exists; `.gitignore` updated; PROJECT.md populated from git state |
| 3 | Start new session in dotfiles repo | SessionStart loads new files silently; total token budget < 2,500 |
| 4 | "remember that I prefer terse tool outputs" | `~/.claude/.memory/user/terse_tool_outputs.md` written; index line in global MEMORY.md |
| 5 | In fresh session: "what tone do I prefer?" | Agent finds USER.md or the `user/` entry without explicit search prompt |
| 6 | `mcp__plugin_qmd_qmd__search` for "terse" | Returns the new entry |
| 7 | Trigger `capture-learning` Phase 3 with fabricated observation | Scope+type routing decision is logged before write |
| 8 | Invoke `/handoff` | Both `tmp/handoffs/...` AND `<repo>/.memory/project/handoff_<slug>.md` written; `daily/<today>.md` appended |
| 9 | Add fake stale entry; run `/memory-curate` | Interactive pruning flow shown |
| 10 | Point `memory-init` at legacy memory dir | Heuristic-split runs; originals preserved |

---

## Live / CLI Verification

| ID | Command / Action | Expected Observation |
|----|------------------|---------------------|
| L01 | `ls -la ~/.claude/.memory/` | Tree exists with MEMORY.md, USER.md, GLOSSARY.md, 4 typed subdirs, daily/ |
| L02 | `cat ~/.claude/.memory/MEMORY.md \| head -20` | Pointer-index format with scope+precedence header |
| L03 | `ls -la <repo>/.memory/` | Mirror tree; `.gitignore` filters daily/ + _archive/ |
| L04 | `git check-ignore <repo>/.memory/daily/` | Path is gitignored |
| L05 | `git check-ignore <repo>/.memory/project/` | Path is NOT gitignored (committable) |
| L06 | `mcp__plugin_qmd_qmd__status` | Both `~/.claude/.memory/` and `<repo>/.memory/` indexed |
| L07 | Bash: `wc -c ~/.claude/.memory/MEMORY.md ~/.claude/.memory/USER.md` | Each < 5,000 bytes |
| L08 | Bash: count of skills containing `memory` in description | At least 3 (memory-init, memory-write, memory-curate) plus 5 updated |

---

## Verification Status

| Deliverable | Built | Stocktake | Tests | skill-creator review | Overall |
|-------------|-------|-----------|-------|----------------------|---------|
| D1 memory-init | ⏳ | ⏳ | ⏳ | ⏳ | TODO |
| D2 memory-write | ⏳ | ⏳ | ⏳ | ⏳ | TODO |
| D3 memory-curate | ⏳ | ⏳ | ⏳ | ⏳ | TODO |
| D4 capture-learning edit | ⏳ | ⏳ | ⏳ | ⏳ | TODO |
| D5 handoff edit | ⏳ | ⏳ | ⏳ | ⏳ | TODO |
| D6 grill-with-memory edit | ⏳ | ⏳ | ⏳ | ⏳ | TODO |
| D7 setup-sdd-repo edit | ⏳ | ⏳ | ⏳ | ⏳ | TODO |
| D8 repo-governance edit | ⏳ | ⏳ | ⏳ | ⏳ | TODO |
| D9 session-start-memory hook | ⏳ | n/a | ⏳ | n/a | TODO |
| D10 CLAUDE.md override | ⏳ | n/a | ⏳ | n/a | TODO |
