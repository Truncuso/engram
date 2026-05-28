# WP-C5: Update `repo-governance` (extend stale-link scan)

**Status**: TODO
**Severity**: LOW
**Created**: 2026-05-17
**Implemented**: —
**Depends on**: WP-C4
**Relevant Sources:** [SRC-03]

---

## Problem

`repo-governance` currently scans `.memory/MEMORY.md` (legacy single-file model) for stale references. Under the unified architecture, memory has typed subdirs each containing files with `links:` frontmatter pointing at CONTEXT.md anchors, ADR files, plans/ WPs, and other memory entries via `[[slug]]`. The scan must walk all subdirs and verify every link's target still exists.

---

## Target Files

- `~/.claude/skills/repo-governance/SKILL.md` (102 lines, EDIT — extend scan logic)

---

## Verified Evidence

- `/home/christoph/dotfiles/claude/skills/repo-governance/SKILL.md` — existing 102-line skill
- Plan source §3.2 — typed-file `links:` frontmatter schema
- Plan source §7 row "repo-governance" — proposed change spec

---

## Implementation Steps

| Step | File | State |
|------|------|-------|
| 1. Read existing `~/.claude/skills/repo-governance/SKILL.md` | (read) | TODO |
| 2. Identify existing scan logic that walks `.memory/MEMORY.md` | SKILL.md | TODO |
| 3. Extend to walk `.memory/{user,feedback,project,reference}/` recursively | SKILL.md | TODO |
| 4. Parse each file's frontmatter `links:` list | SKILL.md | TODO |
| 5. Validate each link target: `[[slug]]` → memory file exists; `CONTEXT.md#<anchor>` → file + anchor exist; `plans/<wp>` → WP file/dir exists | SKILL.md | TODO |
| 6. Report dangling links with severity (HIGH for plans/, MEDIUM for CONTEXT, LOW for `[[]]`) | SKILL.md | TODO |
| 7. Run `skill-stocktake` Quick Scan | live | TODO |
| 8. Test against repo with seeded dangling link | live | TODO |

---

## Verification

### Build Gates

| Gate | Command | Expected |
|------|---------|----------|
| `skill-stocktake` Quick Scan | run on repo-governance | PASS |

### Specific Tests

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| WP-C5-T01 | Repo with all valid links | Clean report; 0 dangling | Output inspect |
| WP-C5-T02 | Repo with one missing `[[slug]]` target | LOW dangling-link warning with file path and missing slug | Output inspect |
| WP-C5-T03 | Repo with missing CONTEXT.md anchor | MEDIUM dangling-anchor warning | Output inspect |
| WP-C5-T04 | Repo with reference to archived plans/<wp> | HIGH dangling-plan warning | Output inspect |
| WP-C5-T05 | Scan walks all typed subdirs | All `.memory/{user,feedback,project,reference}/*.md` files inspected | Trace output |

---

## Recommended Agents

| Phase | Agent | Rationale |
|-------|-------|-----------|
| Editing | Claude main loop | Single skill file |
| Test fixture creation | Subagent in worktree | Isolated test repo |

---

## Review

| Reviewer | Verdict | Notes |
|-----------|---------|-------|
| (pending) | — | — |

---

## Follow-up Issues

| Issue | Severity | Status |
|-------|----------|--------|
| Add auto-fix mode (remove dangling links on confirm) | LOW | Defer to future iteration |
