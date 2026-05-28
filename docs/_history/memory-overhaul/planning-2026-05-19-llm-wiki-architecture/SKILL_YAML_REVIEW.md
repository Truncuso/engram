# Skill YAML Best Practices Review

**Date**: 2026-05-19
**Audit Scope**: All 24 planned skills in the llm-memory architecture
**Source**: [Claude Code Skills Documentation](https://code.claude.com/docs/en/skills)

---

## Executive Summary

The plan specifies skill YAML with only `name` and `description` fields. Claude Code
supports 13 frontmatter fields, 4 string substitutions, `context: fork` subagent
execution, and per-skill hooks. The plan is missing **9 out of 13 fields** across
all 24 skills. This review documents every gap and provides per-skill
recommendations to be applied during WP implementation.

**Key gaps:**

| # | Gap | Impact |
|---|-----|--------|
| 1 | `when_to_use` field unused | Trigger phrases embedded in description; no separation of what vs when |
| 2 | `context: fork` + `agent:` unused | Heavy skills run inline consuming main context; no isolation for research/synthesis |
| 3 | `allowed-tools` unused | Every skill invocation prompts for tool approval; no pre-approval for safe operations |
| 4 | `paths` unused | Vault-scoped skills may activate outside vault context |
| 5 | `disable-model-invocation` unused | Destructive skills (curate, dedup) could be auto-invoked by Claude |
| 6 | `model` / `effort` unused | No model-tiering: cheap daily ops use same model as deep synthesis |
| 7 | `arguments` / `argument-hint` used on only 2 skills | Most skills with CLI args don't declare them |
| 8 | String substitutions unmentioned | `${CLAUDE_SKILL_DIR}`, `$ARGUMENTS`, `${CLAUDE_SESSION_ID}` not in plan |
| 9 | `hooks` (per-skill) unused | No pre/post validation hooks scoped to individual skills |

---

## 1. Field-by-Field Analysis

### 1.1 `description` + `when_to_use` — Trigger Quality

**Current state**: All trigger phrases are embedded in `description` or listed in a
separate "Triggers" column in `SYSTEM_DOCUMENTATION.md` §5. No distinction between
_what the skill does_ and _when Claude should invoke it_.

**Best practice** (from docs + skill-creator):
- `description`: What the skill does + pushy activation hint. "Make sure to use this
  skill whenever the user mentions X, Y, or Z."
- `when_to_use`: Specific trigger phrases, example requests, contexts. Appended to
  description in the skill listing. Combined cap: 1,536 characters.
- Descriptions must be "pushy" to combat undertriggering.

**Recommendation**: Every skill SKILL.md must split into `description` (what + pushy
hint) and `when_to_use` (concrete trigger phrases). The "Triggers" column in
SYSTEM_DOCUMENTATION.md §5 should move into `when_to_use` fields.

### 1.2 `context: fork` + `agent:` — Subagent Execution

**Current state**: Only WP16 defines a subagent (`~/.claude/agents/memory.md`). No
skill uses `context: fork`.

**Best practice**: Skills with heavy LLM work, research, or long-running tasks should
run in forked context to avoid consuming the main agent's context window. The docs
show `context: fork` + `agent: Explore` for research skills.

**Skills that SHOULD use `context: fork`:**

| Skill | Agent | Reason |
|-------|-------|--------|
| `memory-research` | Explore | 3-round web research in isolation |
| `memory-synthesize` | general-purpose | Co-occurrence analysis over entire vault |
| `memory-digest` | general-purpose | Period summarization over many pages |
| `claude-history-ingest` | general-purpose | Large JSONL mining |
| `memory-lint` | general-purpose | Full-vault audit (13 issue types) |
| `cross-linker --deep` | general-purpose | Emergent entity detection (LLM-heavy) |
| `memory-export` | general-purpose | GraphML/JSON/HTML export generation |
| `memory-dashboard` | general-purpose | Dashboard generation |

**Skills that should NOT use `context: fork`:**
- `memory-init` — needs filesystem side effects in main context
- `memory-write` — quick single-fact saves
- `memory-query` — needs conversation context for follow-ups
- `daily-update` — lightweight, fast
- `memory-status` — quick delta computation

### 1.3 `allowed-tools` — Pre-Approved Tool Access

**Current state**: Not mentioned in any WP spec. Every tool call during skill
execution requires per-use approval.

**Best practice**: Skills that run deterministic scripts or safe CLI operations
should pre-approve those tools. This is a UX and security best practice — it doesn't
restrict other tools; it just removes approval prompts for the listed ones.

Note: `allowed-tools` uses pattern matching. `Bash(git *)` matches all git commands.

**Recommendations:**

| Skill | allowed-tools |
|-------|---------------|
| `memory-ingest` | `Bash(qmd *) Bash(git *) Read Write Edit Glob Grep` |
| `memory-query` | `Bash(qmd *) Read Grep Glob` |
| `daily-update` | `Bash(qmd *) Bash(git *) Read Glob` |
| `memory-lint` | `Read Grep Glob Bash(node *)` |
| `cross-linker` | `Read Write Edit Grep Glob` |
| `memory-write` | `Read Write Edit Bash(qmd *)` |
| `memory-curate` | `Read Write Edit Bash(git *) Bash(qmd *)` |
| `memory-synthesize` | `Read Write Grep Glob` |
| `memory-digest` | `Read Write Glob` |
| `memory-init` | `Bash(*) Read Write Edit` |
| `memory-export` | `Read Bash(node *) Bash(python3 *)` |
| `graph-colorize` | `Read Write Edit Bash(node *)` |

### 1.4 `paths` — Scope-Based Activation

**Current state**: Not used. Vault-scoped skills rely on the user being in the right
directory or explicit scope flags.

**Best practice**: Use `paths` glob patterns so skills auto-activate only when
working with relevant files.

```
Portable skills (4):    No paths restriction — activate anywhere
Vault-scoped skills:   paths: "~/memory/**" or paths: "**/.memory/**"
```

**Recommendation**: All 20 vault-scoped skills should set:
```yaml
paths: "~/memory/**,**/.memory/**"
```

The 4 portable skills (`memory-init`, `memory-write`, `memory-query`, `memory-update`)
should NOT set `paths` — they work from any directory.

### 1.5 `disable-model-invocation` — Destructive Operation Guard

**Current state**: Not used on any skill spec. Claude could auto-invoke destructive
operations.

**Best practice**: Set `disable-model-invocation: true` on skills that:
- Modify or delete vault content in ways that can't be auto-reversed
- Execute multi-step write operations that should be user-initiated
- Have side effects (deployments, external API calls)

**Recommendations:**

| Skill | Setting | Reason |
|-------|---------|--------|
| `memory-curate` | `true` | DEDUP/REBUILD modes are destructive; must be user-initiated |
| `memory-write` | `true` | Writing facts should be user-initiated (explicit save) |
| `memory-init` | `true` | Scaffolding creates directories, symlinks, hook config |
| `memory-research` | `true` | Web research costs tokens + time |
| `ingest-url` | `true` | External network call |
| `memory-digest` (save mode) | `true` | Write operation |
| All others | `false` (default) | Safe for auto-invocation |

### 1.6 `model` / `effort` — Compute Tiering

**Current state**: Not mentioned. All skills use the session's active model.

**Best practice**: Assign model based on task complexity and frequency:
- **Haiku**: Cheap, frequent, deterministic (daily-update, memory-status)
- **Sonnet**: Balanced (most skills)
- **Opus**: Deep reasoning, rare (monthly digest, synthesis, research)

**Recommendations:**

| Skill | model | Reason |
|-------|-------|--------|
| `daily-update` | `haiku` | Runs daily via cron; cheap and fast |
| `memory-status` | `haiku` | Quick delta computation |
| `memory-lint` | `sonnet` | Needs thoroughness for 13 issue types |
| `memory-synthesize` | `opus` | Deep co-occurrence analysis over entire vault |
| `memory-digest` (monthly) | `opus` | Monthly deep synthesis |
| `memory-digest` (daily/weekly) | `sonnet` | Lighter period summaries |
| `memory-research` | `opus` | Autonomous multi-round research |
| `cross-linker --deep` | `sonnet` | Emergent entity detection |
| `claude-history-ingest` | `sonnet` | Large JSONL processing |
| All others | `inherit` | Use session default |

### 1.7 `arguments` / `argument-hint` — Structured Parameters

**Current state**: Only `memory-init` and `memory-write` use `argument-hint`.
`memory-curate` has `argument-hint` too. Others lack parameter declarations.

**Skills needing `arguments` / `argument-hint`:**

| Skill | argument-hint | arguments |
|-------|---------------|-----------|
| `memory-init` | `[--global \| --project] [--dry-run]` | `[scope, dry_run]` |
| `memory-write` | `[fact text \| --forget <slug>]` | `[fact]` |
| `memory-curate` | `[--global \| --project \| --both]` | `[scope]` |
| `memory-digest` | `[daily \| weekly \| monthly]` | `[period]` |
| `memory-query` | `[query text]` | `[query]` |
| `memory-ingest` | `[source-path]` | `[source]` |
| `memory-research` | `[topic]` | `[topic]` |
| `memory-export` | `[json \| graphml \| html]` | `[format]` |
| `ingest-url` | `[url]` | `[url]` |
| `cross-linker` | `[--deep]` | `[flags]` |
| `memory-lint` | `[--consolidate]` | `[flags]` |
| `graph-colorize` | `[by-category \| by-tag \| by-tier \| by-lifecycle \| by-confidence]` | `[mode]` |
| `tag-taxonomy` | `[audit \| normalize \| add <tag> \| remove <tag>]` | `[mode, tag]` |
| `data-ingest` | `[file-path]` | `[path]` |

### 1.8 String Substitutions

**Current state**: Not mentioned in any WP spec or SKILL.md template.

**Available substitutions** (from docs):

| Variable | Use |
|----------|-----|
| `$ARGUMENTS` | All arguments passed to skill |
| `$0, $1, ...` | Positional arguments |
| `$name` | Named argument from `arguments:` field |
| `${CLAUDE_SKILL_DIR}` | Path to skill directory (for bundled scripts) |
| `${CLAUDE_SESSION_ID}` | Current session ID (for logging) |
| `${CLAUDE_EFFORT}` | Current effort level |

**Usage in plan skills:**

- `memory-init` should use `${CLAUDE_SKILL_DIR}` to reference bundled assets:
  ```
  node ${CLAUDE_SKILL_DIR}/assets/verify-memory.cjs
  ```
  instead of hardcoding `~/.claude/skills/memory-init/assets/verify-memory.cjs`

- `memory-digest` should use `${CLAUDE_SESSION_ID}` for session-scoped digest naming
- All skills with bundled scripts should use `${CLAUDE_SKILL_DIR}` for path resolution
- `memory-research` should use `$ARGUMENTS` for the research topic

### 1.9 `hooks` — Per-Skill Lifecycle

**Current state**: Not mentioned.

**Best practice**: Hooks can be scoped to individual skills. Useful for:
- Pre-flight validation before destructive operations
- Post-execution verification (run verify script)
- Session cleanup

**Recommendation**: Defer per-skill hooks to WP11/WP13. Add a note in WP2 that the
hook architecture supports per-skill hooks as an advanced pattern. The
`memory-curate` skill is the primary candidate (pre-flight: check vault integrity,
post-execution: run verify-lint.sh).

---

## 2. Per-Skill YAML Template

Each skill's SKILL.md frontmatter should follow this template (fill only the
fields that apply):

```yaml
---
name: <skill-name>
description: >
  <1-2 sentences: what it does + pushy activation hint.
  "Make sure to use this skill whenever the user mentions...">
when_to_use: >
  <Specific trigger phrases, example requests, contexts.
  E.g., "what do I know about X", "search my memory", "find everything related to Y">
argument-hint: "[<arg1>] [<arg2>]"
arguments: [arg1, arg2]
disable-model-invocation: true|false
user-invocable: true|false
allowed-tools: Tool1 Tool2(pattern *)
model: haiku|sonnet|opus|inherit
effort: low|medium|high|xhigh|max
context: fork          # only for heavy/research skills
agent: Explore|Plan|general-purpose|<custom>  # only with context: fork
paths: "glob,patterns" # for vault-scoped skills
---
```

---

## 3. Complete 24-Skill YAML Recommendations

### Portable Skills (4) — No `paths:`, available everywhere

| Skill | disable-model | context:fork | model | allowed-tools | argument-hint |
|-------|---------------|--------------|-------|---------------|---------------|
| `memory-init` | true | — | inherit | `Bash(*) Read Write Edit` | `[--global \| --project] [--dry-run]` |
| `memory-write` | true | — | inherit | `Read Write Edit Bash(qmd *)` | `[fact text \| --forget <slug>]` |
| `memory-query` | false | — | inherit | `Bash(qmd *) Read Grep Glob` | `[query text]` |
| `memory-update` | false | — | sonnet | `Read Write Edit Bash(git *) Bash(qmd *)` | — |

### Vault-Scoped Skills (20) — `paths: "~/memory/**,**/.memory/**"`

**Core Operations (WP4-WP5):**

| Skill | disable-model | context:fork | model | allowed-tools |
|-------|---------------|--------------|-------|---------------|
| `memory-ingest` | false | — | sonnet | `Bash(qmd *) Bash(git *) Read Write Edit Glob Grep` |
| `daily-update` | false | — | haiku | `Bash(qmd *) Bash(git *) Read Glob` |
| `memory-lint` | false | fork | sonnet | `Read Grep Glob Bash(node *)` |
| `memory-status` | false | — | haiku | `Read Grep Glob` |
| `cross-linker` | false | fork (--deep only) | sonnet | `Read Write Edit Grep Glob` |

**Lifecycle (WP6-WP7):**

| Skill | disable-model | context:fork | model | allowed-tools |
|-------|---------------|--------------|-------|---------------|
| `memory-curate` | true | fork | sonnet | `Read Write Edit Bash(git *) Bash(qmd *)` |

**Synthesis + Research (WP8):**

| Skill | disable-model | context:fork | model | allowed-tools |
|-------|---------------|--------------|-------|---------------|
| `memory-synthesize` | false | fork | opus | `Read Write Grep Glob` |
| `memory-digest` | false | fork | opus (monthly), sonnet | `Read Write Glob` |
| `tag-taxonomy` | false | — | sonnet | `Read Write Edit Grep Glob` |
| `memory-research` | true | fork | opus | `WebSearch WebFetch Read Write` |
| `ingest-url` | true | fork | sonnet | `WebFetch Read Write` |
| `data-ingest` | false | — | sonnet | `Read Write` |

**History (WP9):**

| Skill | disable-model | context:fork | model | allowed-tools |
|-------|---------------|--------------|-------|---------------|
| `memory-history-ingest` | false | — | sonnet | `Read Glob Bash(node *)` |
| `claude-history-ingest` | false | fork | sonnet | `Read Glob Bash(node *)` |
| `memory-history-search` | false | — | inherit | `Bash(qmd *) Read Grep` |

**Visualization (WP10):**

| Skill | disable-model | context:fork | model | allowed-tools |
|-------|---------------|--------------|-------|---------------|
| `graph-colorize` | false | — | inherit | `Read Write Edit Bash(node *)` |
| `memory-export` | false | fork | inherit | `Read Bash(node *) Bash(python3 *)` |
| `memory-dashboard` | false | fork | inherit | `Read Write` |
| `memory-bridge` | false | — | inherit | `Read Grep Glob` |
| `memory-context-pack` | false | — | inherit | `Read Bash(node *)` |

**Reference (WP2):**

| Skill | disable-model | context:fork | model | allowed-tools |
|-------|---------------|--------------|-------|---------------|
| `llm-memory` | false | — | inherit | — |

---

## 4. WP Spec Updates Required

### WP2 (llm-memory schema skill)

Add after Step 5 (page frontmatter template):

> **Step 5a: Define skill YAML standard.** Document the full SKILL.md frontmatter
> schema as the canonical reference for all memory skills. Include all 13 fields,
> the 4 string substitutions, and the vault-scoped vs portable `paths` convention.
> This is the single source of truth that WP4-WP11 skills reference for their own
> frontmatter.

### WP4 (ingest + query + daily-update)

Add to each skill's implementation step:

> **YAML configuration**: Set `allowed-tools` to pre-approve QMD operations and
> file reads. Set `paths: "~/memory/**,**/.memory/**"` for vault-scoped activation.
> Set `model: haiku` for `daily-update` (cheap, frequent). Split `description` and
> `when_to_use` following WP2's standard.

### WP6 (memory-write overhaul)

Add to Phase 1:

> **YAML configuration**: Set `disable-model-invocation: true` (explicit-save
> model). Set `allowed-tools: Read Write Edit Bash(qmd *)`. This is a portable
> skill — no `paths:` restriction.

### WP7 (memory-curate overhaul)

Add to implementation:

> **YAML configuration**: Set `disable-model-invocation: true` (destructive
> operations require user initiation). Set `context: fork` for DEDUP and REBUILD
> modes (run in isolation). Set `allowed-tools: Read Write Edit Bash(git *) Bash(qmd *)`.

### WP8 (synthesis + digest + research)

Add per-skill YAML requirements:

> - `memory-synthesize`: `context: fork`, `model: opus`, `paths: "~/memory/**"`
> - `memory-digest`: `context: fork`, `model: opus` (monthly), `sonnet` (daily/weekly)
> - `memory-research`: `disable-model-invocation: true`, `context: fork`, `agent: Explore`, `model: opus`
> - `ingest-url`: `disable-model-invocation: true`, `context: fork`

### WP13 (framework finalization)

Add to Step 7 (final review pass):

> **YAML audit**: Run `verify-skill-frontmatter.sh --extended` to check every skill
> has: `description` + `when_to_use` split, `paths` set on vault-scoped skills,
> `disable-model-invocation` set on destructive skills, `allowed-tools` set on
> operational skills, `arguments`/`argument-hint` set on parameterized skills.

---

## 5. Verification Additions

The following entries should be added to `VERIFICATION.md`:

### New verification script: `verify-skill-yaml.sh`

| ID | Test | Expected Result |
|----|------|-----------------|
| VY.1 | All 24 skills have `name` and `description` | Non-empty, valid YAML |
| VY.2 | All skills have `when_to_use` with >=3 trigger phrases | Non-empty |
| VY.3 | Destructive skills have `disable-model-invocation: true` | memory-curate, memory-write, memory-init, memory-research, ingest-url |
| VY.4 | Vault-scoped skills have `paths: "~/memory/**,**/.memory/**"` | 20 out of 24 skills |
| VY.5 | Portable skills do NOT have `paths:` | memory-init, memory-write, memory-query, memory-update |
| VY.6 | Skills with `context: fork` have valid `agent:` | agent is Explore, Plan, or general-purpose |
| VY.7 | Operational skills have `allowed-tools` set | Non-empty, valid tool names |
| VY.8 | Parameterized skills have `argument-hint` | Present where args accepted |
| VY.9 | Skills with bundled scripts use `${CLAUDE_SKILL_DIR}` | No hardcoded paths to skill directory |
| VY.10 | `$ARGUMENTS` or `$0`/`$1` used where args accepted | Substitution present in body |

### Updated WP13 verification (V13.2 replacement)

Replace V13.2 ("All skills have valid SKILL.md with required frontmatter") with:

> V13.2: `verify-skill-yaml.sh` passes — all 24 skills pass VY.1–VY.10 checks.
> `verify-skill-frontmatter.sh` continues to check basic YAML validity; the new
> script enforces the semantic rules above.

---

## 6. Implementation Notes for Future WPs

### For WP4-WP11 skill authors:

1. **Always split `description` and `when_to_use`.** The `description` field says
   what the skill does with a pushy activation hint. The `when_to_use` field lists
   specific trigger phrases users would actually say.

2. **Set `paths:` on every vault-scoped skill.** This prevents the skill from
   activating when the user is working outside a memory vault. The portable skills
   are the exception.

3. **Pre-approve safe tools with `allowed-tools`.** Every `Bash(qmd *)` call in a
   skill body should have a corresponding `allowed-tools` entry. This eliminates
   approval prompts during normal operation.

4. **Use `context: fork` for any skill that reads 10+ pages or makes 5+ LLM calls.**
   This keeps the main context clean. The forked subagent gets the skill content as
   its prompt and returns a summary.

5. **Set `model:` on skills with non-default compute needs.** Haiku for cheap daily
   ops, Opus for deep monthly synthesis.

6. **Use `${CLAUDE_SKILL_DIR}` for bundled script paths**, never hardcode
   `~/.claude/skills/<name>/` — the skill might be installed at project or plugin
   level.

### For WP16 (Memory Agent) — Reconciliation with `context: fork`:

The WP16 Memory Agent and `context: fork` skills are complementary, not competing:

- **WP16 Memory Agent** = autonomous **scheduler** + **agent type definition**. It
  decides *which* skill to run, interprets results, maintains the self-improvement
  loop. It does NOT ingest, synthesize, or lint in-process.
- **`context: fork` skills** = isolated **workers**. When invoked, the SKILL.md
  frontmatter `context: fork` + `agent: memory` forks into a subagent with vault
  knowledge but its own context window.

**Critical architectural decision**: Forked memory skills should use `agent: memory`
(not `Explore` or `general-purpose`). The `~/.claude/agents/memory.md` definition
provides the vault-structure knowledge, tool access, and skill catalog that every
forked worker needs. Registering `memory` as a recognized agent type means:

1. Forked subagents inherit vault knowledge from the agent definition
2. The Memory Agent scheduler spawns workers using the same type it defines
3. Lightweight skills (`memory-query`, `memory-status`, `daily-update`) stay
   in-process — no fork overhead

**Updated `agent:` field for forked skills:**

| Skill | `agent:` |
|-------|----------|
| `memory-ingest` (batch mode) | `memory` |
| `memory-lint` | `memory` |
| `cross-linker --deep` | `memory` |
| `memory-curate` (DEDUP/REBUILD) | `memory` |
| `memory-synthesize` | `memory` |
| `memory-digest` | `memory` |
| `memory-research` | `memory` |
| `claude-history-ingest` | `memory` |
| `memory-export` | `memory` |
| `memory-dashboard` | `memory` |

This reconciles WP16 with the `context: fork` pattern — the agent type definition
lives at `~/.claude/agents/memory.md`, and skills declare `agent: memory` to fork
into that environment.

---

## 7. Decisions Log

| Decision | Rationale |
|----------|-----------|
| `memory-write` gets `disable-model-invocation: true` | Explicit-save model: user must say "remember this" |
| `memory-curate` gets `disable-model-invocation: true` | DEDUP/REBUILD are destructive |
| `memory-research` gets `disable-model-invocation: true` + `context: fork` + `agent: Explore` | Web research costs tokens; isolation prevents context pollution |
| `daily-update` gets `model: haiku` | Runs daily via cron; cheap and fast |
| `memory-synthesize` + `memory-digest` (monthly) get `model: opus` | Deep reasoning over entire vault |
| Vault-scoped skills get `paths: "~/memory/**,**/.memory/**"` | Prevents activation outside vault context |
| Portable skills do NOT get `paths:` | Must work from any project directory |
| All operational skills get `allowed-tools` | Eliminates per-use approval prompts |
| `${CLAUDE_SKILL_DIR}` required for bundled script references | Works regardless of install location |
