# WP-A: Scaffold `memory-init` skill + templates

**Status**: TODO
**Severity**: HIGH (foundation; everything else depends on this)
**Created**: 2026-05-17
**Implemented**: —
**Depends on**: (none)
**Relevant Sources:** [SRC-01], [SRC-02], [SRC-03]

---

## Problem

Three uncoordinated memory approaches exist in the Claude Code installation: typed files at `~/.claude/projects/<slug>/memory/`, freeform `.memory/MEMORY.md`, and Ralph-Loop JSON state. A fourth path (external implementation guide) would create a fifth system without retiring any. We need a single `memory-init` skill that idempotently scaffolds the unified memory tree at both global and project scope, mirrors existing repo signals into seed entries, and prepares prerequisites for QMD-based retrieval.

---

## Target Files

- `~/.claude/skills/memory-init/SKILL.md` — main skill body (5 phases per plan §6.1)
- `~/.claude/skills/memory-init/assets/templates/MEMORY.md` — pointer-index template
- `~/.claude/skills/memory-init/assets/templates/USER.md` — identity + cross-project prefs
- `~/.claude/skills/memory-init/assets/templates/PROJECT.md` — build/branch/conventions snapshot
- `~/.claude/skills/memory-init/assets/templates/GLOSSARY.md` — cross-project terminology
- `~/.claude/skills/memory-init/assets/templates/memory-frontmatter.yaml` — copy-paste frontmatter block
- `~/.claude/skills/memory-init/references/implementation-guide.md` — annotated copy of external guide
- `~/.claude/skills/memory-init/references/migration-heuristics.md` — file-name → type-subdir rules

---

## Verified Evidence

- `/home/christoph/.claude/skills/skill-creator/skills/skill-creator/SKILL.md` — official scaffolding skill confirmed working
- `/home/christoph/dotfiles/claude/CLAUDE.md:78-99` — LSP → QMD → Grep → Read tiering policy that memory retrieval slots into
- `/home/christoph/dotfiles/claude/projects/-home-cunger-10-Projects-01-fsl-cleaningapplication/memory/` — real-world example of typed-file system in use (3 files, 82 lines)
- `/home/christoph/dotfiles/claude/projects/-home-cunger-10-Projects-11-private-OSRS-glite-glite-private/memory/MEMORY.md` — 7,660 lines, counter-example showing why typed files are needed (special-cased in WP-D)
- Plan source: `/home/christoph/.claude/plans/eview-the-media-christoph-samsung-evo990-zesty-lovelace.md` §3, §6.1

---

## Implementation Steps

| Step | File | State |
|------|------|-------|
| 1. `/skill-creator` to scaffold `~/.claude/skills/memory-init/` directory | `~/.claude/skills/memory-init/` | TODO |
| 2. Author `SKILL.md` with frontmatter (name, description, argument-hint per plan §6.1) | `SKILL.md` | TODO |
| 3. Author Phase 1 VERIFY PREREQUISITES section | `SKILL.md` | TODO |
| 4. Author Phase 2 DISCOVER REPO SIGNALS section | `SKILL.md` | TODO |
| 5. Author Phase 3 SCAFFOLD section (create dirs, write templates, `.gitignore` updates) | `SKILL.md` | TODO |
| 6. Author Phase 4 MIRROR & MIGRATE section (defers to WP-D for legacy split) | `SKILL.md` | TODO |
| 7. Author Phase 5 VERIFY section (QMD search confirms, subagent token-budget check) | `SKILL.md` | TODO |
| 8. Create 5 templates in `assets/templates/` | `assets/templates/*.md` | TODO |
| 9. Create `references/implementation-guide.md` (annotated copy of external guide) | `references/` | TODO |
| 10. Create `references/migration-heuristics.md` | `references/` | TODO |
| 11. Test `/memory-init --global` idempotency | live | TODO |
| 12. Test `/memory-init` in dotfiles repo | live | TODO |
| 13. Verify SessionStart token budget < 2,500 with new tree | live | TODO |

---

## Verification

### Build Gates

| Gate | Command | Expected |
|------|---------|----------|
| Skill structure | `ls ~/.claude/skills/memory-init/{SKILL.md,assets,references}` | All paths exist |
| Frontmatter valid | `head -20 ~/.claude/skills/memory-init/SKILL.md` | Has `name`, `description`, `argument-hint` keys |
| Skill loaded | Claude Code session shows `memory-init` in available skills list | Skill is invocable |

### Specific Tests

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| WP-A-T01 | `/memory-init --global` on empty `~/.claude/.memory/` | Tree created with MEMORY.md, USER.md, GLOSSARY.md, user/, feedback/, reference/, daily/ | Bash `find ~/.claude/.memory/ -type f` |
| WP-A-T02 | `/memory-init --global` re-run on populated tree | No overwrite; idempotent; PROJECT/USER refresh only | Diff before/after |
| WP-A-T03 | `/memory-init` in `dotfiles/claude/` | `<repo>/.memory/` created; `.gitignore` updated; daily/+_archive/ ignored | `git check-ignore .memory/daily/` |
| WP-A-T04 | `/memory-init` re-run | Idempotent — only PROJECT.md refreshed from current git state | Diff before/after |
| WP-A-T05 | SessionStart token budget after init | Total ≤ 2,500 tokens across global+project tree loads | Token count in fresh session |
| WP-A-T06 | QMD indexes new tree | `mcp__plugin_qmd_qmd__search "memory"` returns seed entries | QMD search result |
| WP-A-T07 | PROJECT.md reflects current state | Build commands, branch, active WPs match git/repo state | Manual review |

---

## Recommended Agents

| Phase | Agent | Rationale |
|-------|-------|-----------|
| Skill scaffolding | `/skill-creator` (interactive) | Proven frontmatter validation, eval support |
| Template authoring | Claude main loop | Manual writing from plan §3.2-§3.3 spec |
| Idempotency test | Subagent in worktree | Isolated test environment |
| Token-budget verify | Subagent with fresh context | Measures actual SessionStart load |

---

## Review

| Reviewer | Verdict | Notes |
|-----------|---------|-------|
| (pending) | — | — |

---

## Follow-up Issues

| Issue | Severity | Status |
|-------|----------|--------|
| QMD vector embeddings may be missing on this machine (MCP note in system) | LOW | Defer to separate ticket; BM25 fallback is acceptable |
