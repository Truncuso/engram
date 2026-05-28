# WP-B: New skills — `memory-write` + `memory-curate`

**Status**: TODO
**Severity**: HIGH (enables daily-use writes and weekly curation)
**Created**: 2026-05-17
**Implemented**: —
**Depends on**: WP-A
**Relevant Sources:** [SRC-02], [SRC-03]

---

## Problem

Once the global system prompt's "auto memory" instructions guide automatic writes, users still need an explicit write trigger ("remember this", "/remember") and a periodic distillation/pruning operation. The external guide proposed three cron jobs; security review rejected them (cron sends memory contents to LLM API nightly). Replace with two skills: `memory-write` for explicit save and `memory-curate` for manual weekly housekeeping.

---

## Target Files

- `~/.claude/skills/memory-write/SKILL.md`
- `~/.claude/skills/memory-curate/SKILL.md`

---

## Verified Evidence

- Plan source §6.2 — full memory-write spec (scope+type decision, dedup, CONTEXT.md check, confirmation echo)
- Plan source §6.3 — full memory-curate spec (DISTILL → PRUNE → MERGE → ARCHIVE → RE-INDEX → REPORT)
- Global system prompt "auto memory" spec — action types (add/replace/remove), confirmation patterns
- `~/.claude/skills/skill-creator/.../SKILL.md` — scaffolding mechanism

---

## Implementation Steps

| Step | File | State |
|------|------|-------|
| 1. `/skill-creator` to scaffold `~/.claude/skills/memory-write/` | `memory-write/` | TODO |
| 2. Author SKILL.md frontmatter (triggers: "remember this", "note that", "save", "/remember", "/forget") | `memory-write/SKILL.md` | TODO |
| 3. Author scope-decision logic (global vs project) | `memory-write/SKILL.md` | TODO |
| 4. Author type-decision logic (user/feedback/project/reference) | `memory-write/SKILL.md` | TODO |
| 5. Author pre-write checks (dedup, CONTEXT.md check, file-name collision) | `memory-write/SKILL.md` | TODO |
| 6. Author action types (add/replace/remove) with confirmation patterns | `memory-write/SKILL.md` | TODO |
| 7. `/skill-creator` to scaffold `~/.claude/skills/memory-curate/` | `memory-curate/` | TODO |
| 8. Author SKILL.md frontmatter (triggers: "/memory-curate", "curate memory", "consolidate memory") | `memory-curate/SKILL.md` | TODO |
| 9. Author 6 phases (DISTILL/PRUNE/MERGE/ARCHIVE/RE-INDEX/REPORT) | `memory-curate/SKILL.md` | TODO |
| 10. Test memory-write: 4 cases | live | TODO |
| 11. Test memory-curate against synthetic stale daily logs | live | TODO |

---

## Verification

### Build Gates

| Gate | Command | Expected |
|------|---------|----------|
| Skills loaded | Claude Code session lists `memory-write` and `memory-curate` | Both invocable |
| Frontmatter valid | `head -20` on both SKILL.md | Triggers documented |

### Specific Tests

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| WP-B-T01 | "remember that I prefer terse tool outputs" | Writes `~/.claude/.memory/user/terse_tool_outputs.md` with correct frontmatter; index line in global MEMORY.md | File exists + grep index |
| WP-B-T02 | "remember that this repo uses pnpm not npm" (in a repo) | Writes `<repo>/.memory/project/uses_pnpm.md`; index line in project MEMORY.md | File exists + grep index |
| WP-B-T03 | "remember 'cancellation' means soft-delete in this repo" (CONTEXT.md exists) | Nudges to update CONTEXT.md; on accept, writes `reference/glossary_cancellation.md` stub with link | Interactive prompt + file check |
| WP-B-T04 | "forget the staging-url memory" | Reads file, shows content, asks confirmation, deletes file + index line on yes | File deleted + index updated |
| WP-B-T05 | Dedup: second "remember terse outputs" | Detects existing; prompts "update or new"; correct path on either choice | Interactive prompt verified |
| WP-B-T06 | `/memory-curate` with 7 fresh daily logs | DISTILL proposes typed entries; user accepts/rejects each | Interactive flow verified |
| WP-B-T07 | `/memory-curate` with `daily/*.md` older than 30 days | Files moved to `_archive/YYYY-MM/`; QMD still indexes | `ls _archive/` + `qmd:search` |
| WP-B-T08 | `/memory-curate` final report | Shows file counts before/after + token-budget impact | Output inspection |

---

## Recommended Agents

| Phase | Agent | Rationale |
|-------|-------|-----------|
| Skill scaffolding | `/skill-creator` (interactive) | Per WP-A decision |
| Test cases | Claude main loop | Manual driven testing |

---

## Review

| Reviewer | Verdict | Notes |
|-----------|---------|-------|
| (pending) | — | — |

---

## Follow-up Issues

| Issue | Severity | Status |
|-------|----------|--------|
| If `memory-curate` is consistently skipped after 4 weeks, wire to a hook schedule | LOW | Defer until usage data |
