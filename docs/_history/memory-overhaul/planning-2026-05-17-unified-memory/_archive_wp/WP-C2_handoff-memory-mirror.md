# WP-C2: Update `handoff` (write memory mirror)

**Status**: TODO
**Severity**: MEDIUM
**Created**: 2026-05-17
**Implemented**: —
**Depends on**: WP-C1
**Relevant Sources:** [SRC-03]

---

## Problem

`handoff` currently writes only to `tmp/handoffs/YYYY-MM-DD/<slug>.md`. Under the unified architecture, handoff output is the discoverable session boundary: it should also (a) write a `project/handoff_<slug>.md` memory entry that points back to the tmp file, and (b) append the session block to `daily/<today>.md`. This is the v1 daily-log trigger (per user decision: daily log written only on `/handoff`, not every Stop).

---

## Target Files

- `~/.claude/skills/handoff/SKILL.md` (42 lines, EDIT — add post-write memory mirror + daily-log append)

---

## Verified Evidence

- `/home/christoph/dotfiles/claude/skills/handoff/SKILL.md` — existing 42-line skill; frontmatter has `session_id` for "QMD indexing and future `recall` integration" → confirms intent for memory integration
- Plan source §7 row "handoff" — proposed change spec
- Plan source §5.1 — daily-log block format

---

## Implementation Steps

| Step | File | State |
|------|------|-------|
| 1. Read existing `~/.claude/skills/handoff/SKILL.md` | (read) | TODO |
| 2. After tmp/handoff file is written, write `<repo>/.memory/project/handoff_<slug>.md` | SKILL.md | TODO |
| 3. Memory entry frontmatter: `type: project`, `scope: project`, `description: "Handoff <date>: <goal>"`, `links: ["tmp/handoffs/<date>/<slug>.md"]` | SKILL.md | TODO |
| 4. Append session block to `<repo>/.memory/daily/<today>.md` per plan §5.1 format | SKILL.md | TODO |
| 5. Add index line to project `MEMORY.md` under `## Project` section | SKILL.md | TODO |
| 6. Run `skill-stocktake` Quick Scan | live | TODO |
| 7. End-to-end test: invoke `/handoff` and verify all three artifacts | live | TODO |

---

## Verification

### Build Gates

| Gate | Command | Expected |
|------|---------|----------|
| `skill-stocktake` Quick Scan | run on handoff skill | PASS |

### Specific Tests

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| WP-C2-T01 | Invoke `/handoff` with goal="test memory mirror" | `tmp/handoffs/2026-MM-DD/<slug>.md` created (existing behavior) | File exists |
| WP-C2-T02 | Same invocation | `<repo>/.memory/project/handoff_<slug>.md` created with correct frontmatter | File exists + frontmatter inspect |
| WP-C2-T03 | Same invocation | `<repo>/.memory/daily/<today>.md` has new `#### Session N` block | grep daily file |
| WP-C2-T04 | Same invocation | `<repo>/.memory/MEMORY.md` has new index line under `## Project` | grep MEMORY.md |
| WP-C2-T05 | Existing frontmatter unchanged | tmp/handoff file frontmatter is identical to pre-WP-C2 behavior | Diff against prior schema |

---

## Recommended Agents

| Phase | Agent | Rationale |
|-------|-------|-----------|
| Reading existing skill | Claude main loop | Single short file |
| End-to-end test | Subagent | Isolated test environment |

---

## Review

| Reviewer | Verdict | Notes |
|-----------|---------|-------|
| (pending) | — | — |

---

## Follow-up Issues

| Issue | Severity | Status |
|-------|----------|--------|
| Future `recall` skill could query daily logs by session_id | LOW | Out of scope for this WP |
