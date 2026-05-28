# WP-C4: Update `setup-sdd-repo` (delegate to memory-init)

**Status**: TODO
**Severity**: MEDIUM
**Created**: 2026-05-17
**Implemented**: —
**Depends on**: WP-C3
**Relevant Sources:** [SRC-03]

---

## Problem

`setup-sdd-repo` currently scaffolds `.memory/MEMORY.md` inline using a template at `~/.claude/templates/sdd/MEMORY.md` (which does NOT exist on disk — confirmed during exploration). This is a duplicate of what `memory-init` will do under the unified architecture. Delegate entirely to `/memory-init --project` so there's a single source of truth for memory scaffolding.

---

## Target Files

- `~/.claude/skills/setup-sdd-repo/SKILL.md` (144 lines, EDIT — replace inline scaffolding with `/memory-init` invocation)

---

## Verified Evidence

- `/home/christoph/dotfiles/claude/skills/setup-sdd-repo/SKILL.md` — existing 144-line skill
- Survey finding: `~/.claude/templates/sdd/MEMORY.md` referenced but NOT present on disk → setup-sdd-repo's inline scaffolding is currently broken anyway
- Plan source §7 row "setup-sdd-repo" — proposed change spec

---

## Implementation Steps

| Step | File | State |
|------|------|-------|
| 1. Read existing `~/.claude/skills/setup-sdd-repo/SKILL.md` | (read) | TODO |
| 2. Locate the inline `.memory/MEMORY.md` scaffolding section | SKILL.md | TODO |
| 3. Remove inline template references (broken anyway — missing template file) | SKILL.md | TODO |
| 4. Replace with: "Invoke `/memory-init --project` to scaffold `.memory/` tree" | SKILL.md | TODO |
| 5. Document the dependency on `memory-init` in skill prerequisites | SKILL.md | TODO |
| 6. Run `skill-stocktake` Quick Scan | live | TODO |
| 7. End-to-end test in clean test repo | live | TODO |

---

## Verification

### Build Gates

| Gate | Command | Expected |
|------|---------|----------|
| `skill-stocktake` Quick Scan | run on setup-sdd-repo | PASS |
| No dangling template ref | grep for `templates/sdd/MEMORY.md` in skill | No matches |

### Specific Tests

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| WP-C4-T01 | Run `setup-sdd-repo` in clean test repo | `.memory/` tree scaffolded via `/memory-init --project` delegation | `ls <repo>/.memory/` |
| WP-C4-T02 | `setup-sdd-repo` does NOT write any `.memory/` files directly | Only memory-init writes; setup-sdd-repo invokes it | trace tool calls |
| WP-C4-T03 | `setup-sdd-repo` other behaviors intact | plans/, CONTEXT.md, docs/adr/ scaffolding unaffected | Inspect outputs |

---

## Recommended Agents

| Phase | Agent | Rationale |
|-------|-------|-----------|
| Editing | Claude main loop | Single skill file |
| End-to-end test | Subagent with worktree | Clean test repo |

---

## Review

| Reviewer | Verdict | Notes |
|-----------|---------|-------|
| (pending) | — | — |

---

## Follow-up Issues

| Issue | Severity | Status |
|-------|----------|--------|
| If users prefer setup-sdd-repo to remain self-contained, expose --skip-memory flag | LOW | Defer until requested |
