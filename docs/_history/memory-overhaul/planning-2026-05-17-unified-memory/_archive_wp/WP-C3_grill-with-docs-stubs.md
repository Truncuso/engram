# WP-C3: Update `grill-with-docs` (mirror stubs)

**Status**: TODO
**Severity**: MEDIUM
**Created**: 2026-05-17
**Implemented**: —
**Depends on**: WP-C2
**Relevant Sources:** [SRC-03]

---

## Problem

`grill-with-docs` writes terminology to `CONTEXT.md` and decisions to `docs/adr/`. Under the unified architecture, these need discoverable memory pointers so future sessions can find them by index instead of by full-text scan. Add 3-line stub mirrors that point back to the source — CONTEXT.md and docs/adr/ remain source-of-truth, the memory stubs are the discoverable index entries. This is what enables "speak the same language at project and global" (terminology consistency requirement from the user).

---

## Target Files

- `~/.claude/skills/grill-with-docs/SKILL.md` (98 lines, EDIT — add post-write stub generation)

---

## Verified Evidence

- `/home/christoph/dotfiles/claude/skills/grill-with-docs/SKILL.md` — existing 98-line skill; updates CONTEXT.md and `docs/adr/` inline
- Plan source §4.4 "Language alignment" — three integration points; grill-with-docs is integration point #1
- Plan source §7 row "grill-with-docs" — proposed change spec

---

## Implementation Steps

| Step | File | State |
|------|------|-------|
| 1. Read existing `~/.claude/skills/grill-with-docs/SKILL.md` | (read) | TODO |
| 2. After adding/updating a term in CONTEXT.md, write `<repo>/.memory/reference/glossary_<term>.md` with `links: ["CONTEXT.md#<term>"]` | SKILL.md | TODO |
| 3. After creating an ADR `docs/adr/<NNN>-<slug>.md`, write `<repo>/.memory/project/adr_<NNN>_<slug>.md` mirror with `links: ["docs/adr/<NNN>-<slug>.md"]` | SKILL.md | TODO |
| 4. Stubs are pointers, not duplicates — 3 lines body max | SKILL.md | TODO |
| 5. Add index lines to project `MEMORY.md` under `## Reference` and `## Project` | SKILL.md | TODO |
| 6. Run `skill-stocktake` Quick Scan | live | TODO |
| 7. Test stub generation against new CONTEXT.md term + new ADR | live | TODO |

---

## Verification

### Build Gates

| Gate | Command | Expected |
|------|---------|----------|
| `skill-stocktake` Quick Scan | run on grill-with-docs skill | PASS |

### Specific Tests

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| WP-C3-T01 | Grill session adds term "cancellation" to CONTEXT.md | `<repo>/.memory/reference/glossary_cancellation.md` created | File exists |
| WP-C3-T02 | Stub frontmatter | `type: reference`, `scope: project`, `links: ["CONTEXT.md#cancellation"]` | Frontmatter inspect |
| WP-C3-T03 | Stub body | ≤ 3 lines; refers to CONTEXT.md as source-of-truth | Line count + manual review |
| WP-C3-T04 | Grill session creates ADR 007 "storage layer" | `<repo>/.memory/project/adr_007_storage_layer.md` mirror created | File exists |
| WP-C3-T05 | Index lines added | Project MEMORY.md has new entries under `## Reference` and `## Project` | grep MEMORY.md |
| WP-C3-T06 | Source-of-truth links resolve | Each stub's `links:` target file exists and contains the referenced anchor | Path + grep |

---

## Recommended Agents

| Phase | Agent | Rationale |
|-------|-------|-----------|
| Reading existing skill | Claude main loop | One file |
| End-to-end test | Subagent | Isolate from main context |

---

## Review

| Reviewer | Verdict | Notes |
|-----------|---------|-------|
| (pending) | — | — |

---

## Follow-up Issues

| Issue | Severity | Status |
|-------|----------|--------|
| Stub-source drift if CONTEXT.md term renamed | LOW | Detected by `repo-governance` extended scan (WP-C5) |
