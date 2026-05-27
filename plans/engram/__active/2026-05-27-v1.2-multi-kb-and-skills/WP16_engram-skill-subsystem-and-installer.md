---
name: wp16-engram-skill-subsystem-and-installer
title: engram skill subsystem (orchestrator + chain.yaml + children) + `engram agent install`
type: work-package
stage: spec
severity: HIGH
created: 2026-05-27
updated: 2026-05-27
plan: 2026-05-27-v1.2-multi-kb-and-skills
tags: [skills, subsystem, installer, agent-surface]
relationships:
  - blocked-by: [[wp13-multi-kb-registry-and-kbplugin-seam]]
  - blocked-by: [[wp17-episodic-semantic-linkage-hardening]]
sources: [SRC-01, SRC-02]
---
<!-- Template: WP v2 (frontmatter-first) -->

# WP16: engram skill subsystem + installer

## Problem

engram needs an agent-facing surface above the MCP verbs: slash skills,
capture hooks, and a one-command installer per platform. A flat bundle
forces duplication when skills compose (`/handoff` = recall + remember +
summarize). ADR-0008 picks a skill subsystem — one `engram` orchestrator + a
sibling `chain.yaml` + modular children — matching the user's global
agentic-OS layer policy ("Modular > mega; chain.yaml at runtime;
self-improvement is propose-only out-of-band").

WP16 ships the orchestrator, `chain.yaml`, the seed children, the installer,
and uninstall. Installer targets Claude Code first; Codex / Gemini CLI /
OpenCode / Cursor adapters land in the same WP via target-specific
sub-modules.

## Target Files

- `src/agent-pack/SKILL.md` — `engram` orchestrator (the bundle's entry).
- `src/agent-pack/chain.yaml` — declarative chain definition.
- `src/agent-pack/children/engram-recall/SKILL.md` — search across connected
  KBs; optional bridge expansion.
- `src/agent-pack/children/engram-remember/SKILL.md` — write a memory.
- `src/agent-pack/children/engram-forget/SKILL.md` — lifecycle transition or
  audited `governance_delete`.
- `src/agent-pack/children/engram-recap/SKILL.md` — recall + remember.
- `src/agent-pack/children/engram-handoff/SKILL.md` — recap + remember.
- `src/agent-pack/children/engram-session-history/SKILL.md`.
- `src/agent-pack/children/engram-commit-context/SKILL.md`.
- `src/agent-pack/children/engram-connect-kb/SKILL.md` — discover + register +
  schedule.
- `src/agent-pack/children/engram-wiki-ingest/SKILL.md` — pull transcripts
  into an `llm-wiki` KB.
- `src/agent-pack/children/engram-session-rollup/SKILL.md`.
- `src/agent-pack/children/engram-grill-with-memory/SKILL.md`.
- `src/agent-pack/hooks/{session-start,user-prompt-submit,pre-tool-use,post-tool-use,post-tool-use-failure,pre-compact,subagent-start,subagent-stop,stop,session-end}.sh` — 10 hook scripts (8 capture + `PreCompact` + the two subagent hooks if §6.1 doesn't already have them).
- `src/agent-pack/installer/index.ts` — `engram agent install`.
- `src/agent-pack/installer/targets/{claude-code,codex,gemini-cli,opencode,cursor}.ts` — per-platform adapters.
- `src/cli/agent.ts` — `engram agent {install,uninstall}`.
- `tests/integration/agent-install-claude-code.test.ts`, …per target.

## Verification Gate

| # | Check | Test |
|---|-------|------|
| 1 | `engram agent install --target claude-code` writes the orchestrator + chain.yaml + ≥10 child skills + MCP entry + capture hooks; `engram agent uninstall` removes exactly those files. | `tests/integration/agent-install-claude-code.test.ts` (SC-22) |
| 2 | Installer is idempotent: re-running install does not duplicate entries. | `tests/integration/agent-install-idempotent.test.ts` |
| 3 | `chain.yaml` is read at runtime by the orchestrator; children are NOT rewritten by the orchestrator. | `tests/unit/agent-pack/chain-determinism.test.ts` |
| 4 | Each child skill is independently invocable via the platform's slash interface — `/engram-recall`, `/engram-remember`, etc. — and calls MCP verbs. | `tests/integration/child-skills-standalone.test.ts` |
| 5 | All 5 install targets succeed against a stub platform fixture (Codex, Gemini CLI, OpenCode, Cursor). | `tests/integration/agent-install-<target>.test.ts` |
| 6 | Hooks fire fire-and-forget within their budget; `PreCompact` is request/response (§6.1). | `tests/integration/hooks-budget.test.ts` |
| 7 | `engram agent install` fails with a clear error when engramd is unreachable. | `tests/integration/agent-install-no-daemon.test.ts` |

## Implementation Steps

| Step | File | State |
|------|------|-------|
| Orchestrator + `chain.yaml` | `src/agent-pack/{SKILL.md,chain.yaml}` | TODO |
| Seed child skills (×11) | `src/agent-pack/children/**` | TODO |
| Hook scripts (×10) | `src/agent-pack/hooks/**` | TODO |
| Installer core + 5 target adapters | `src/agent-pack/installer/**` | TODO |
| `engram agent {install,uninstall}` CLI | `src/cli/agent.ts` | TODO |
| Integration tests per target | `tests/integration/agent-install-*.test.ts` | TODO |

## Verified Evidence

— (none yet — WP in `spec` stage)

## Agents

| Stage | Agent | Reason |
|-------|-------|--------|
| design | `skill-creator` | child-skill descriptions / triggering accuracy |
| impl | `typescript-pro` | installer + target adapters |
| review | `code-reviewer` | install idempotence + uninstall completeness |

## Review

| Reviewer | Verdict | Notes |
|----------|---------|-------|
| — | — | — |

## Follow-up Issues

| Issue | Severity | Status |
|-------|----------|--------|
| — | — | — |
