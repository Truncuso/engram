# WP-13: Framework Repo Finalization

**Severity**: MEDIUM
**Status**: Phase 2b — PLANNING
**Depends on**: WP0 (repo scaffold), WP2 (llm-memory schema), WP3 (memory-init), WP4
(ingest+query), WP5 (housekeeping), WP11 (existing skill edits), WP12 (cron).
Execution: Phase 5 in OVERVIEW.md.

## Problem

WP0 created the skeleton repo and pushed it to GitHub. By Phase 5, all ~24 skills
have been iteratively written, all verification scripts exist, all agent configs
are tested, WP11 has updated the pre-existing skills, and WP12 has wired the cron
jobs. The framework repo is functionally complete but not **public-ready**:

- The README is a stub from the obsidian-wiki template — it references
  `obsidian-wiki` skills, `OBSIDIAN_VAULT_PATH`, `~/.obsidian-wiki/config`, and
  links to the wrong GitHub org.
- The AGENTS.md is a stub — it needs a full routing table mapping all ~24
  `llm-memory` skill triggers, not obsidian-wiki's 30+ skill routing.
- No CI workflow exists to validate setup.sh idempotency or run `make verify`.
- The repo has never been secrets-scanned for tokens, real paths, or PII in
  committed files (G3).
- The v1.0.0 tag has not been cut.
- Symlink verification: it has not been confirmed that each of the 12 agent
  platforms resolves correctly through the `setup.sh` topology.

This WP is the **public-release gate**. It turns the functional repo into a
publicly consumable framework and produces the formal v1.0.0 release.

## Target Files

All modifications are in the framework repo
(`~/10_Projects/Agentic AI/Workflows/automatic-workflows/llm-memory/`):

- `README.md` — complete rewrite: architecture overview, Obsidian integration guide,
  quick start, agent compatibility table, skill catalog, QMD setup, contributing
- `AGENTS.md` — full skill routing table for the llm-memory system (not
  obsidian-wiki), config resolution protocol (`.env` → `~/.llm-memory/config`,
  not `~/.obsidian-wiki/config`), core principles, vault structure reference,
  scope auto-discovery
- `.github/workflows/ci.yml` — new CI workflow: runs setup.sh, asserts idempotency,
  runs `make verify`
- `scripts/verify/verify-setup.sh` — new: validates setup.sh idempotency
  (second run produces clean working tree), all symlinks resolve, all agent config files
  are reachable
- VCS tag: `v1.0.0` on the final commit

## Implementation Steps

### Step 1: Secrets scan (G3 enforcement)

Before any public-facing finalization, run a deliberate scan of every committed
file in the framework repo for secrets:

```
Scan checklist:
1. grep -rE '(ghp_|gho_|github_pat_|sk-|api.?key|token.?=)' .skills/ scripts/ config/ agent-config/
2. grep -rE '(~/\.claude/|/home/cunger/)' .skills/ scripts/ config/ agent-config/ — flags real local paths that should be $HOME or ~ placeholders
3. grep -rE '(password|secret|credential)' .skills/ scripts/ config/ agent-config/ --ignore-case
4. Verify .env.example contains only placeholder values (no real tokens)
5. Check .gitignore covers .env, .obsidian/, .trash/, _raw/, _archive/
```

Any finding is a **BLOCK** — fix before proceeding. Document scan results in the
commit message for this step.

### Step 2: Rewrite README.md

Replace the obsidian-wiki-derived stub with a complete `llm-memory` README:

**Required sections:**

1. **Banner + one-liner** — "A multi-agent framework for building and maintaining
   an LLM-managed Obsidian knowledge base. Compile once, query forever."
2. **Quick Start** — `git clone` + `bash setup.sh`; `memory-init --global` for
   vault; `memory-ingest` a first source.
3. **Architecture Overview** — diagram/description of the three-layer system:
   - Layer 1: Memory Store (`~/memory/` vault + `<repo>/.memory/` per-project)
   - Layer 2: Skills (ingest, query, lint, cross-link, synthesize, digest, curate,
     history-ingest, visualization, export)
   - Layer 3: Autonomous Upkeep (QMD index daemon, ingestion agent, curation agent)
4. **How It Works** — Karpathy compile-once pattern: Ingest, Extract, Resolve,
   Schema. Adapted from the obsidian-wiki README but with `~/memory/` vault paths,
   per-project `.memory/`, and our own skill catalog.
5. **Agent Compatibility** — table of 12 supported agents, their bootstrap files,
   skill directories, and slash-command support. Copied from OVERVIEW.md agent
   bootstrap table but formatted as end-user documentation.
6. **Skill Catalog** — table of all ~24 skills with description, slash command,
   and whether portable (global) or vault-scoped. Grouped by category
   (core, ingest, housekeeping, synthesis, history, visualization).
7. **Obsidian Integration** — open `~/memory/` as an Obsidian vault; graph view;
   graph-colorize for color-coding; Obsidian Bases dashboards; recommended
   companion skills (kepano/obsidian-skills).
8. **QMD Semantic Search** — optional setup: `qmd index`, `.env` vars
   (`QMD_WIKI_COLLECTION`, `QMD_TRANSPORT`), graceful degradation when absent.
9. **Project Structure** — tree diagram of the framework repo.
10. **Cross-Project Usage** — how `memory-update` and `memory-query` work from
    any project directory.
11. **Contributing** — how to add a new skill (create `.skills/<name>/SKILL.md`,
    run `setup.sh`).
12. **License** — MIT badge.

Use the obsidian-wiki README as a structural reference but replace ALL path
references (`OBSIDIAN_VAULT_PATH` → `LLM_MEMORY_VAULT_PATH`,
`~/.obsidian-wiki/config` → `~/.llm-memory/config`, `Ar9av/obsidian-wiki` →
`cunger/llm-memory`), all skill names, and all agent config file names
(`obsidian-wiki.mdc` → `llm-memory.mdc`, etc.).

### Step 3: Rewrite AGENTS.md with full skill routing table

Replace the obsidian-wiki-derived AGENTS.md with `llm-memory` content:

1. **Configuration** — config resolution via `.env` walk or `~/.llm-memory/config`;
   read `$LLM_MEMORY_VAULT_PATH/AGENTS.md` if it exists (owner-specific conventions).
2. **Vault Structure Reference** — `~/memory/` layout (concepts/, entities/, skills/,
   references/, synthesis/, journal/, projects/, _raw/, _meta/, _archive/) and
   per-project `./.memory/` layout.
3. **Skill Routing Table** — complete mapping of user intents to skills. Each entry:
   user says something like X, invoke skill Y. Cover all ~24 adapted skills
   (standalone + fused):

| User says… | Skill |
|---|---|
| "set up my memory vault" / "memory-init" / "/memory-init" | `memory-init` |
| "add this to the memory vault" / "ingest this" / "process these docs" | `memory-ingest` |
| "what do I know about X" / "find info on Y" / any knowledge question | `memory-query` |
| "audit" / "lint" / "find broken links" / "memory vault health" | `memory-lint` |
| "what's the status" / "what's been ingested" / "show the delta" / "memory insights" | `memory-status` |
| "link my pages" / "cross-reference" / "connect my memory" | `cross-linker` |
| "save this fact" / "remember this" / "/memory-write" / "/memory-capture" | `memory-write` |
| "curate my memory" / "rebuild the memory vault" / "dedup" / "stage" | `memory-curate` |
| "synthesize my memory vault" / "find connections" / "what concepts co-occur" | `memory-synthesize` |
| "daily digest" / "weekly digest" / "monthly digest" / "what did I learn" | `memory-digest` |
| "normalize tags" / "tag audit" / "fix my tags" | `tag-taxonomy` |
| "/memory-research [topic]" / "research X" | `memory-research` |
| "/ingest-url <url>" / "add this URL" | `ingest-url` |
| "import my data" / "process this export" / logs, transcripts | `data-ingest` |
| "import my Claude history" / "mine my conversations" | `claude-history-ingest` |
| "/memory-history-ingest" / "import agent history" | `memory-history-ingest` |
| "/memory-claude [topic]" / "/memory-codex [topic]" etc. | `memory-history-search` |
| "color my graph" / "color code obsidian" / "color by tag/category" | `graph-colorize` |
| "export memory" / "export graph" / "graphml" / "neo4j" | `memory-export` |
| "create a dashboard" / "vault dashboard" / "show all X as a table" | `memory-dashboard` |
| "/memory-bridge" / "compare tool memories" / "cross-tool memory" | `memory-bridge` |
| "/memory-context-pack" / "export context for a session" | `memory-context-pack` |
| "/daily-update" / "morning sync" / "refresh the memory vault index" | `daily-update` |
| "update memory vault" / "sync to memory vault" / "save project knowledge" | `memory-update` |
| "/llm-memory" / "explain the memory architecture" | `llm-memory` |

4. **Core Principles** — compile don't retrieve, track everything, connect with
   `[[wikilinks]]`, frontmatter required, single source of truth, keep context warm.
5. **Scope Auto-Discovery** — how the agent determines global vs project scope from
   CWD; Hybrid D symlink topology.
6. **Visibility Tags** (optional) — `visibility/public`, `visibility/internal`,
   `visibility/pii`.

### Step 4: Create CI workflow

Create `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup QMD
        uses: tobi/qmd-action@v1  # or install via cargo/brew

      - name: Run setup.sh (idempotency test)
        run: |
          bash scripts/setup.sh --ci  # --ci flag skips interactive prompts
          git diff --exit-code  # setup.sh must not leave a dirty working tree
          bash scripts/setup.sh --ci  # second run
          git diff --exit-code  # second run must also be clean

      - name: Run make verify
        run: make verify
```

The `--ci` flag on setup.sh suppresses interactive prompts (agent detection,
config prompts) and uses default/CI-safe paths. The flag is added in this WP if
it does not already exist in setup.sh from WP0.

### Step 5: Symlink verification

Verify that all 12 agent platforms resolve correctly through the setup.sh topology:

1. After `setup.sh --ci` runs, for each agent platform, assert:
   - The skill symlink directory exists (e.g., `~/.claude/skills/`, `~/.gemini/skills/`)
   - For Claude Code specifically: 4 portable skills symlinked globally; all ~24
     in project-local `.claude/skills/`
   - For all other agents: all ~24 skills symlinked globally
   - Each symlink target resolves to an existing `.skills/<name>/SKILL.md` in the
     framework repo
   - Agent config files are reachable at their documented paths
2. Add these checks to `scripts/verify/verify-symlinks.sh` (extends the WP0/WP1
   symlink verification to cover the full agent topology).

### Step 6: Create verify-setup.sh

Write `scripts/verify/verify-setup.sh` — deterministic pass/fail for setup
idempotency:

```
Test: setup.sh second run produces no changes
1. Take a clean checkout of the framework repo
2. Run setup.sh --ci
3. Capture git status
4. Run setup.sh --ci again
5. Assert git status is identical (no file creations, modifications, deletions)
6. Assert all symlinks still resolve (same as Step 5)
```

### Step 7: Final review pass

1. Confirm no references to `obsidian-wiki`, `OBSIDIAN_VAULT_PATH`,
   `~/.obsidian-wiki/config`, or `Ar9av/obsidian-wiki` remain in any committed
   file (grep the whole repo).
2. Confirm all skill SKILL.md files have valid YAML frontmatter (name, description
   present and non-empty) — reuse `verify-skill-frontmatter.sh`.
3. Confirm `make verify` passes — every verification script under
   `scripts/verify/` green (`verify-all.sh` is the authoritative runner; do not
   hard-code a script count, it grows with the skill set).
4. Run `code-reviewer` on README.md and AGENTS.md for correctness and completeness.

### Step 8: Tag v1.0.0

```
git tag -a v1.0.0 -m "v1.0.0: llm-memory framework — initial public release

- 24 skills adapted from obsidian-wiki
- Multi-agent support: 12 agent platforms
- Autonomous upkeep: QMD daemon, ingestion agent, curation agent
- Script-first verification: one deterministic verify script per skill + WP
- Per-project .memory/ vaults with scope auto-discovery
- MIT licensed"
git push origin v1.0.0
```

Confirm the tag appears on GitHub and the release page shows the full README.

## Recommended Agents

- `code-reviewer` — final review of README.md, AGENTS.md, CI workflow
- `security-reviewer` — G3 secrets scan validation
- `superpowers:finishing-a-development-branch` — tag and push workflow

## Verification

See VERIFICATION.md WP-13 section:
- V13.1: `verify-setup.sh` — setup.sh idempotent; second run produces no changes
  to working tree; all symlinks resolve; all agent config files reachable.
- V13.2: `verify-skill-frontmatter.sh` — all ~24 skills have valid SKILL.md
  with required frontmatter fields (name, description).
- V13.3: README.md documents all required sections — quick start, architecture,
  agent compatibility, skill catalog, Obsidian integration, QMD setup, contributing.
  `grep` asserts each section heading is present.
- V13.4: AGENTS.md routing table covers all ~24 skills — every skill from the
  `.skills/` directory has a corresponding routing entry with trigger phrases.
- V13.5: Secrets scan passes — zero findings for token patterns, real local paths
  (other than placeholder examples), or PII in committed files.
- V13.6: CI workflow present and syntactically valid YAML; `--ci` flag exists
  in setup.sh.
- V13.7: git tag v1.0.0 exists and points to the final commit.

## Complexity Delta

- **Added**: CI workflow (new `.github/` directory), `--ci` flag on setup.sh,
  `verify-setup.sh` script.
- **Removed**: None (stub files replaced with full content).
- **Justification**: CI and idempotency verification are essential for a public
  framework repo serving as a single source of truth for 12 agent platforms.
  Without CI, regressions in setup.sh go undetected. Without the secrets scan,
  a public repo is a leakage risk (G3).
