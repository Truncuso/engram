# WP-C1: Update `capture-learning` (scope+type routing)

**Status**: TODO
**Severity**: HIGH
**Created**: 2026-05-17
**Implemented**: —
**Depends on**: WP-B
**Relevant Sources:** [SRC-03], [SRC-04]

---

## Problem

`capture-learning` Phase 3 currently routes observations to one of six destinations (local skill, rule, WP finding, CONTEXT.md, ADR, MEMORY.md) using a single-file MEMORY.md model. Under the new architecture, MEMORY.md is a pointer-only index and observations must be split by scope (global vs project) and type (user/feedback/project/reference). Phase 4 must write the typed file; Phase 5 must update the correct scope's index.

---

## Target Files

- `~/.claude/skills/capture-learning/SKILL.md` (203 lines, EDIT — extend Phases 3-5)

---

## Verified Evidence

- `/home/christoph/dotfiles/claude/skills/capture-learning/SKILL.md` — existing 203-line skill, classification table at ~line 88-98 of the existing body (the table mapping observations to destinations)
- Plan source §7 row "capture-learning" — proposed change spec
- Plan source §3.2 — typed-file frontmatter format that Phase 4 must produce

---

## Implementation Steps

| Step | File | State |
|------|------|-------|
| 1. Read existing `~/.claude/skills/capture-learning/SKILL.md` in full | (read) | TODO |
| 2. Extend Phase 3 CLASSIFY: after destination decision (skill/rule/finding/CONTEXT/ADR/MEMORY), add scope+type sub-decision when destination = MEMORY | SKILL.md | TODO |
| 3. Document scope decision logic (mirror memory-write §6.2) | SKILL.md | TODO |
| 4. Document type decision logic (user/feedback/project/reference) | SKILL.md | TODO |
| 5. Extend Phase 4 CREATE: produce typed `.md` file with full frontmatter (name, description, type, scope, created, origin_session, links) | SKILL.md | TODO |
| 6. Extend Phase 5 INDEX: append index line to correct scope's `MEMORY.md` under correct type section | SKILL.md | TODO |
| 7. Update phase-boundary triggers documentation if needed | SKILL.md | TODO |
| 8. Run `skill-stocktake` Quick Scan | live | TODO |
| 9. Hand-curated test: 5 observations route correctly | live | TODO |

---

## Verification

### Build Gates

| Gate | Command | Expected |
|------|---------|----------|
| Skill loads | Session lists `capture-learning` with new description | Skill invocable |
| `skill-stocktake` Quick Scan | run skill-stocktake on changed skill | PASS |

### Specific Tests

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| WP-C1-T01 | Observation: "user wants no emoji in commit messages" | Routes to feedback, global, file `~/.claude/.memory/feedback/feedback_no_emoji.md` | Decision-trace inspection |
| WP-C1-T02 | Observation: "this repo's auth uses JWT not OAuth" | Routes to project, project, file `<repo>/.memory/project/auth_uses_jwt.md` | Decision-trace inspection |
| WP-C1-T03 | Observation: "Linear project INGEST tracks pipeline bugs" | Routes to reference, global, file `~/.claude/.memory/reference/linear_ingest.md` | Decision-trace inspection |
| WP-C1-T04 | Observation: "user is a senior data scientist" | Routes to user, global, file `~/.claude/.memory/user/user_role.md` | Decision-trace inspection |
| WP-C1-T05 | Observation: "WP-M1 visualization rework completed 2026-05-15" | Routes to project, project, file `<repo>/.memory/project/wp_m1_complete.md` | Decision-trace inspection |
| WP-C1-T06 | Phase 5 INDEX updates correct MEMORY.md (project for WP-C1-T02, global for T01,T03,T04) | One line added in correct scope's index under correct type section | grep MEMORY.md |

---

## Recommended Agents

| Phase | Agent | Rationale |
|-------|-------|-----------|
| Reading existing skill | Claude main loop | Just one file |
| Test case evaluation | Subagent | Avoid contaminating main context with test traces |

---

## Review

| Reviewer | Verdict | Notes |
|-----------|---------|-------|
| (pending) | — | — |

---

## Follow-up Issues

| Issue | Severity | Status |
|-------|----------|--------|
| Decision logic may need tuning after ≥ 20 real-world classifications | LOW | Collect telemetry; revisit in WP-C5 follow-up |
