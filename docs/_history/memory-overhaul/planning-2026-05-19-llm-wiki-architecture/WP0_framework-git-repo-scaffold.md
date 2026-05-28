# WP0: Framework Git Repo Scaffold

**Severity**: HIGH
**Status**: Pending
**Depends on**: None (first work package)

## Problem

The framework repo is the canonical source of truth for all skills, templates, scripts, hooks, configs, and agent bootstraps. It must be created first — everything else symlinks from it. Without it, skills are scattered across `~/.claude/skills/` with no git tracking, no multi-agent support, and no way to share the system publicly.

## Target Files

All created in `~/10_Projects/Agentic AI/Workflows/automatic-workflows/llm-memory/`:

```
llm-memory/
├── .skills/                          # Canonical skill source — ALL 35+ skills live here
├── agent-config/                     # Agent-specific bootstrap configs
│   ├── claude/
│   │   └── CLAUDE.md                 # Claude Code bootstrap (symlinked to repo root)
│   ├── cursor/
│   │   └── .cursor/rules/llm-memory.mdc  # alwaysApply: true, skill routing table
│   ├── windsurf/
│   │   └── .windsurf/rules/llm-memory.md # activation: always-on
│   ├── gemini/
│   │   ├── GEMINI.md                 # Gemini CLI bootstrap
│   │   └── .agent/rules/llm-memory.md    # Antigravity alwaysApply
│   ├── copilot/
│   │   └── .github/copilot-instructions.md
│   ├── kiro/
│   │   └── .kiro/steering/llm-memory.md  # inclusion: always
│   └── generic/
│       └── AGENTS.md                 # Codex, Hermes, OpenClaw, Trae, OpenCode, Aider, Droid
├── scripts/
│   ├── hooks/                        # session-start-memory.cjs, session-end-memory.cjs, qmd-refresh.cjs
│   ├── verify/                       # All verification scripts
│   ├── daemons/                      # qmd-index-daemon.sh, ingest-agent.sh, curate-agent.sh
│   └── setup.sh                      # One-command multi-agent install
├── templates/                        # Page, frontmatter, index, log, hot, manifest, project-overview
├── config/                           # .env.example, config.yaml.example, graph.json.default
├── hooks/                            # hooks.json.example
├── Makefile                          # make verify, make install, make clean
├── LICENSE                           # MIT
└── README.md
```

## Implementation Steps

### Step 1: Create repo skeleton
```
mkdir -p ~/10_Projects/Agentic\ AI/Workflows/automatic-workflows/llm-memory
cd ~/10_Projects/Agentic\ AI/Workflows/automatic-workflows/llm-memory
git init
```

### Step 2: Create directory structure
```
mkdir -p .skills agent-config/{claude,cursor,windsurf,gemini,copilot,kiro,generic}
mkdir -p scripts/{hooks,verify,daemons}
mkdir -p templates config hooks
```

### Step 3: Copy and adapt setup.sh from obsidian-wiki
- Copy `obsidian-wiki/setup.sh` as starting point
- Replace skill names, paths, and agent list with our system's
- Add hook installation, QMD registration, cron setup, daemon installation

### Step 4: Move existing skills into repo (atomic — copy, don't move)
```
# Copy existing 6 skills from ~/.claude/skills/ into repo
for skill in memory-init memory-write memory-curate memory-onboard grill-with-memory capture-learning; do
  cp -r ~/.claude/skills/$skill .skills/$skill
done
# Do NOT remove originals yet — WP3 establishes symlink farm, then removes originals
```

### Step 5: Create agent bootstrap configs
Each agent config file contains the same core adapted to format:
1. Config Resolution Protocol
2. Skill Routing Table (all 35 skills)
3. Core Principles (compile don't retrieve, track everything, wikilinks, frontmatter)
4. Vault Structure Reference
5. Scope Auto-Discovery

### Step 6: Create AGENTS.md as master bootstrap
- AGENTS.md is the canonical multi-agent bootstrap file
- CLAUDE.md, GEMINI.md, .hermes.md → symlinks to AGENTS.md
- Content: full routing table, config protocol, vault structure, core principles

### Step 7: Create Makefile
```makefile
.PHONY: verify install clean

verify:
    @for script in scripts/verify/*.sh; do echo "Running $$script..."; bash "$$script" || exit 1; done
    @echo "All verification scripts passed."

install:
    bash scripts/setup.sh

clean:
    # Remove all symlinks created by setup.sh
    find . -type l -name "*.md" -delete
```

### Step 8: Push to GitHub
```
git add -A && git commit -m "chore: initial framework repo scaffold"
gh repo create cunger/llm-memory --public --source=. --push
```

## Global Symlink Topology (setup.sh)

### Project-local symlinks (relative — inside framework repo dir)
When running `setup.sh` from the framework repo directory:

| Target | Source | Skills |
|--------|--------|--------|
| `.claude/skills/*` | `../../.skills/*` | ALL 35 (relative symlink) |
| `.cursor/skills/*` | `../../.skills/*` | ALL 35 |
| `.windsurf/skills/*` | `../../.skills/*` | ALL 35 |
| `.agents/skills/*` | `../../.skills/*` | ALL 35 |
| `.kiro/skills/*` | `../../.skills/*` | ALL 35 |

### Global symlinks (absolute — from anywhere on the system)

| Target | Source | Skills |
|--------|--------|--------|
| `~/.claude/skills/` | `<repo>/.skills/` | **4 portable only**: memory-init, memory-write, memory-query, memory-update |
| `~/.gemini/skills/` | `<repo>/.skills/` | ALL 35 |
| `~/.gemini/antigravity/skills/` | `<repo>/.skills/` | ALL 35 |
| `~/.codex/skills/` | `<repo>/.skills/` | ALL 35 |
| `~/.hermes/skills/` | `<repo>/.skills/` | ALL 35 |
| `~/.openclaw/skills/` | `<repo>/.skills/` | ALL 35 |
| `~/.copilot/skills/` | `<repo>/.skills/` | ALL 35 |
| `~/.trae/skills/` | `<repo>/.skills/` | ALL 35 |
| `~/.trae-cn/skills/` | `<repo>/.skills/` | ALL 35 |
| `~/.kiro/skills/` | `<repo>/.skills/` | ALL 35 |
| `~/.agents/skills/` | `<repo>/.skills/` | ALL 35 |

### Why 4 portable skills for Claude Code?
Following the obsidian-wiki pattern: Claude Code uses project-local `.claude/skills/` (relative symlinks, all 35 skills) when you're IN the vault directory. Globally (`~/.claude/skills/`), only skills useful from ANY project are installed:
- `memory-init` — scaffold a vault or `.memory/` from any project
- `memory-write` — save a fact from any project context
- `memory-query` — search memory from any project
- `memory-update` — sync project knowledge into vault from any project

Other agents (Gemini, Codex, etc.) don't have project-local skill discovery — they only use global paths — so they get ALL 35 skills globally.

## Verification

See VERIFICATION.md WP0 section:
- V0.1: `git status` in framework repo → clean working tree after scaffold
- V0.2: `~/.claude/skills/memory-init` resolves → symlink to framework repo
- V0.3: All existing 6 skills symlinked, SKILL.md accessible through each symlink
