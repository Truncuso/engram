# WP-2: llm-memory Core Architecture Skill

**Severity**: HIGH
**Status**: Pending
**Depends on**: WP0 (framework repo — the skill lives there), WP1 (vault paths exist)

## Problem

`llm-memory` is the "theory" skill — the schema layer. It defines *how* the memory vault is
structured: categories, the page frontmatter template, wikilink relationship
types, the tiered retrieval protocol, and the three-layer model (raw → memory vault →
schema). Every other memory skill (`memory-ingest`, `memory-query`, `memory-lint`, …)
relies on this skill's conventions. Without it there is no shared contract and
each operational skill would re-invent the page format.

The obsidian-wiki source skill (`obsidian-wiki/.skills/llm-memory/SKILL.md`, 526
lines) is the blueprint. This WP **adapts** it — it is not a verbatim copy. Two
substantive changes are required for our system:

1. **Path model.** obsidian-wiki resolves the vault from `OBSIDIAN_VAULT_PATH` in
   a `.env`. Our system has two scopes — global `~/memory/` and project
   `<repo>/.memory/` — resolved by a scope-discovery rule, not a single env var.
2. **Retrieval backend.** obsidian-wiki's retrieval primitives assume grep + file
   reads. Ours must name QMD as Tier-2 (consistent with `~/.claude/CLAUDE.md`'s
   LSP→QMD→Grep→Read tiering) — the index pass is a `qmd search`, not a frontmatter grep.

## Target Files

- `<framework-repo>/.skills/llm-memory/SKILL.md` — adapted port (new)
- `<framework-repo>/templates/page.md` — the canonical page template referenced
  by the skill (new)
- `<framework-repo>/templates/frontmatter.yaml` — frontmatter field reference (new)

## Implementation Steps

### Step 1: Scaffold via skill-creator

Scaffold `llm-memory` in the framework repo's `.skills/`. Frontmatter `name:
llm-memory`, `description:` adapted from the source (drop Obsidian-specific
phrasing; keep "theory skill, other skills handle operations").

### Step 2: Port the three-layer architecture section

Keep verbatim in spirit — raw sources (immutable) → memory vault (LLM-maintained) →
schema. Adapt: "raw sources live wherever the user keeps them" → "raw sources are
reached through the vault's `_raw/` symlink farm" (the Hybrid-D bridge, OQ-1).

### Step 3: Port + adapt the category model

Categories: `concepts/`, `entities/`, `skills/`, `references/`, `synthesis/`,
`journal/`, plus `projects/<name>/` for project-scoped knowledge. This is the
same set WP-1 scaffolds and WP-3's `memory-init` creates — the three WPs must
agree on the category list; `llm-memory` is the source of truth for it.

### Step 4: Add the scope-resolution rule (NEW — not in the source)

Document the rule every operational skill uses to pick a vault:

- Walk up from CWD looking for a `.memory/` directory, not crossing the git root.
  If found → **project scope**, vault = `<repo>/.memory/`.
- Else → **global scope**, vault = `~/memory/` (resolved via
  `~/.llm-memory/config` `LLM_MEMORY_VAULT_PATH`).
- Project facts win over global on conflict (carried over from Phase 1).

This rule replaces obsidian-wiki's single `OBSIDIAN_VAULT_PATH`.

### Step 5: Port the page frontmatter template

Adopt the rich schema (the Phase 1 minimal frontmatter is a strict subset):

```yaml
title:          # human-readable page title
category:       # concept | entity | skill | reference | synthesis | journal | project
tags: []        # taxonomy tags
sources: []     # provenance — paths/URLs the page was distilled from
summary:        # one-line; used by index.md and the index-pass retrieval tier
created:        # ISO 8601
updated:        # ISO 8601
base_confidence: # high | medium | low — provenance-derived
lifecycle:      # draft | reviewed | verified | disputed | archived
provenance:     # extracted | inferred | ambiguous
relationships:  # typed wikilinks: extends, implements, contradicts,
                #   derived_from, uses, replaces, related_to
```

Write `templates/page.md` and `templates/frontmatter.yaml` as the referenced
artifacts. Migrated Phase 1 facts (WP-1 Phase 2) use `lifecycle: verified`,
`provenance: extracted`.

### Step 6: Port + adapt the tiered retrieval protocol

Cost-ordered escalation, with QMD as the index tier:

1. **Index pass** — `qmd search` over the vault collection; read `summary`
   frontmatter of hits only.
2. **Semantic pass** — `qmd vector_search` when BM25 misses.
3. **Section grep** — grep within candidate pages for the relevant `##` section.
4. **Full read** — read the whole page only when sections are insufficient.

This mirrors the `CLAUDE.md` tiering policy — name that alignment explicitly.

### Step 7: Document special files

`index.md` (catalog), `log.md` (operations log), `hot.md` (recent/active focus),
`.manifest.json` (SHA-256 content hashes for delta computation). These are
created by `memory-init` (WP-3) and maintained by `daily-update` (WP-4).

### Step 8: Define skill YAML standard (NEW — canonical reference for all skills)

Document the full `SKILL.md` frontmatter schema in `llm-memory/SKILL.md` as the
single source of truth for every WP4-WP11 skill's frontmatter. Include:

1. **Required fields**: `name`, `description` (pushy: what + when to use), `when_to_use`
   (specific trigger phrases, >=3 per skill).
2. **Invocation control**: `disable-model-invocation: true` for destructive skills
   (memory-init, memory-curate, memory-write, memory-research, ingest-url).
   `user-invocable: false` for background skills (daily-update as cron target).
3. **Subagent execution**: `context: fork` + `agent: memory` for skills that read
   10+ pages or make 5+ LLM calls. The `agent: memory` type is defined by WP16's
   `~/.claude/agents/memory.md`.
4. **Tool pre-approval**: `allowed-tools` on every operational skill, matching the
   tools the skill body actually uses (e.g., `Bash(qmd *)` for QMD operations).
5. **Scope gating**: `paths: "~/memory/**,**/.memory/**"` on all 20 vault-scoped
   skills. No `paths:` on the 4 portable skills.
6. **Model tiering**: `model: haiku` for cheap/frequent ops (daily-update,
   memory-status), `model: opus` for deep reasoning (memory-synthesize,
   memory-digest monthly, memory-research), `model: inherit` for everything else.
7. **Arguments**: `argument-hint` + `arguments` on parameterized skills;
   `$ARGUMENTS`/`$N`/`$name` substitution in body.
8. **Bundled scripts**: `${CLAUDE_SKILL_DIR}` for all references to bundled
   scripts — never hardcode `~/.claude/skills/<name>/` paths.

Reference: `SKILL_YAML_REVIEW.md` for the complete per-skill recommendations table.

### Step 9: skill-creator review pass

Run a skill-creator review; confirm frontmatter, triggers, and that the skill
stays "theory only" — no operational steps that belong in `memory-ingest`/`-query`.

## Recommended Agents

- `skill-creator` — scaffold + review.
- `code-reviewer` — final SKILL.md review.

## Verification

See VERIFICATION.md WP-2 section:
- V2.1: `verify-skill-frontmatter.sh` passes on `llm-memory/SKILL.md`.
- V2.2: the category list in `llm-memory` matches what WP-1 scaffolds and WP-3's
  `memory-init` creates (no drift between the three).
- V2.3: `templates/page.md` parses as valid YAML frontmatter + body and contains
  every field in Step 5.
- V2.4: the scope-resolution rule is documented and matches WP-3 Phase 1's scope
  detection.
- V2.5: the retrieval protocol names QMD as Tier-2, consistent with
  `~/.claude/CLAUDE.md`.
