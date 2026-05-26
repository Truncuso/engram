---
name: wp06-capture-captureintake-staging
title: Capture + CaptureIntake + staging
type: work-package
stage: spec
severity: HIGH
created: 2026-05-26
updated: 2026-05-26
plan: 2026-05-26-v1-build
tags: [capture, privacy, hooks]
relationships:
  - depends_on: [[wp05-mcp-server-coreservice-facade-16-verbs-bearer]]
  - blocks: [[wp08-dreaming-worker-orchestrator]]
sources: [SRC-01]
phases:
  - phase-1-captureintake-privacy-filter
  - phase-2-staging-jsonl-backpressure
  - phase-3-capture-plugin-cc-hooks
  - phase-4-capture-observability
---
<!-- Template: WP-folder OVERVIEW v2 (frontmatter-first) -->

# WP06: Capture + CaptureIntake + staging

> Folder work package. Phases live in `phase-N-<slug>.md`. `stage:` advances only
> when all phase `phase_status:` are `done`.

## Problem

Before the dreaming worker (WP08) can distill memories from agent sessions, there
must be a reliable, privacy-safe pipeline that captures what agents do. This WP
builds three things: (1) the **CaptureIntake kernel** with its multi-layer
privacy filter (S-01, fail-closed — a filter failure drops the observation, never
passes it), (2) the **per-session staging JSONL** append store with backpressure
and fallback, and (3) the **Claude Code CapturePlugin** that installs hook scripts
and attests each payload with a per-agent MAC secret (G2). Two hook classes per
G1: fire-and-forget capture hooks exit within 200ms; the `PreCompact` re-inject
hook is a separate request/response path (never fire-and-forget).

SPEC refs: §6.1 (capture flow, G1 two hook classes, 200ms exit, privacy filter
S-01 fail-closed, backpressure, capture-fallback 10MB/7d, G2 MAC), §2.3
`CapturePlugin` contract (install/uninstall/normalise), §4.A (capture flow),
§8.3 S-02 (staging MAC attestation), §10.5 (capture observability).

## Target Files

- `src/core/capture-intake.ts` — kernel CaptureIntake: `/capture-intake` POST handler; delegates to privacy filter; appends to staging
- `src/core/privacy-filter.ts` — 4-layer filter: regex (AWS/GitHub/JWT/PEM), entropy (high-entropy strings), path blocklist (`~/.ssh/`, `~/.aws/`, `/run/secrets/`), optional LLM sweep; fail-closed
- `src/core/staging.ts` — per-session JSONL append (atomic fsync); backpressure cap 500MB/10k files; capture-fallback 10MB/7d; drain on daemon start
- `src/plugins/capture/cc/index.ts` — `CapturePlugin` for Claude Code: implements `install`/`uninstall`/`normalise`
- `src/plugins/capture/cc/hooks/` — generated hook scripts: `post-tool-use.sh`, `post-tool-use-failure.sh`, `stop.sh`, `session-end.sh`, `user-prompt-submit.sh`, `subagent-start.sh`, `subagent-stop.sh`, `pre-compact.sh` (separate re-inject path)
- `src/plugins/capture/cc/mac.ts` — MAC attestation: reads `~/.engram/agent-secrets/<id>.mac` at invocation, computes HMAC over normalised payload, discards secret
- `tests/integration/capture-intake.test.ts` — privacy filter strips planted secret + audit log; staging append; MAC verify
- `tests/e2e/sc4-capture.test.ts` — SC-4: CC hooks → staging JSONL; planted secret stripped; filter audit log

## Phases

| Phase | Goal | Status |
|-------|------|--------|
| [phase-1](phase-1-captureintake-privacy-filter.md) | Kernel CaptureIntake: 4-layer privacy filter (regex/entropy/path-blocklist/optional-LLM), fail-closed, filter.audit_log, `/capture-intake` endpoint | pending |
| [phase-2](phase-2-staging-jsonl-backpressure.md) | Per-session JSONL append (atomic fsync), staging cap 500MB/10k files, capture-fallback 10MB/7d + drain on start | pending |
| [phase-3](phase-3-capture-plugin-cc-hooks.md) | CapturePlugin install/uninstall/normalise; CC hook scripts fire-and-forget 200ms; PreCompact request/response (G1); MAC attestation from 0600 file (G2) | pending |
| [phase-4](phase-4-capture-observability.md) | Capture-audit AppLog channel, `engram capture status` CLI | pending |

## Verification

> Required before `stage: done`. Aggregates per-phase `verify:` checks.

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| W06-1 | Payload containing a planted AWS secret key | Privacy filter strips it; `filter.audit_log` records the strip rule; observation dropped (fail-closed) | integration (SC-4) |
| W06-2 | CC capture hook fires on `PostToolUse`; engramd is up | Observation appended to `staging/<agent>/<session>.jsonl`; AppLog `capture` event | e2e (SC-4) |
| W06-3 | Staging JSONL append with fsync; daemon restart | Data survives restart; cursor-bounded drain finds new entries | integration |
| W06-4 | Staging at capacity (500MB or 10k files) | New observations dropped + logged; emergency dream queued | integration |
| W06-5 | engramd is down when hook fires | Hook writes to `capture-fallback/` (≤10MB/7d); daemon drains and re-filters on next start | integration |
| W06-6 | MAC attestation: staging entry with bad/missing MAC | Worker (WP08) rejects on verification; `capture` AppLog event notes invalid MAC | integration (SC-4 / S-02) |
| W06-7 | `engram capture status` | Prints: active session, staging backlog count+bytes, filter suppressions last hour, fallback size, hook failure count | e2e (SC-4) |

## Recommended Agents

| Phase | Agent | Rationale |
|-------|-------|-----------|
| phase-1 | `security-reviewer` | Privacy filter correctness; fail-closed; regex/entropy tuning; S-01 |
| phase-1 | `typescript-pro` | CaptureIntake endpoint + filter pipeline |
| phase-2 | `typescript-pro` | Atomic JSONL append, backpressure, fallback drain |
| phase-3 | `security-reviewer` | MAC attestation (S-02/G2); 0600 file read; hook script shell safety |
| phase-3 | `typescript-pro` | CapturePlugin contract implementation; shell script generation |
| phase-4 | `typescript-pro` | AppLog capture-audit channel; CLI status command |
| all | `code-reviewer`, `tdd-guide` | per-phase gate |
