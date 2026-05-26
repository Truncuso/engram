---
name: wp11-e2e-verification-18-success-criteria
title: E2E verification (18 success criteria)
type: work-package
stage: ready
severity: HIGH
created: 2026-05-26
updated: 2026-05-28
plan: 2026-05-26-v1-build
tags: [e2e, verification, success-criteria, integration, acceptance]
related: [[wp23-v1-3-e2e-acceptance-gate]]
relationships:
  - depends_on: [[wp04-scoring-engine-recall-degradation-chain]]
  - depends_on: [[wp05-mcp-server-coreservice-facade-16-verbs-bearer]]
  - depends_on: [[wp06-capture-captureintake-staging]]
  - depends_on: [[wp07-ingest-worker-graphify-graphplugin-ollama]]
  - depends_on: [[wp08-dreaming-worker-orchestrator]]
  - depends_on: [[wp09-threat-model-hardening-planted-attack-tests]]
  - depends_on: [[wp10-governance-cascade-delete]]
  - depends_on: [[wp13-daemon-process-envelope-operational-layer-engramd-lifecycle-migrate-install-logger-retry]]
sources: [SRC-01]
---
<!-- Template: WP v2 (frontmatter-first) -->

# WP11: E2E verification (18 success criteria)

## Problem

SPEC §12.3 defines 18 success criteria (SC-1 through SC-18) that together
constitute the v1 acceptance gate. Each criterion is verifiable by an automated
test. This WP writes those tests as a final E2E suite that runs against a fully
assembled system (daemon up, all plugins wired, real store on disk). The suite is
the only gate that proves the system ships: **all 18 green**.

**v2.3 extension:** v1.3 introduces SC-19 through SC-27 (dashboard, multi-format
ingest, ADR-as-memory, hook test gate, install). Those are verified by WP23
(`wp23-v1-3-e2e-acceptance-gate`) in the v1.3 milestone — not this WP. WP11
remains scoped to the original 18 v1 success criteria; WP23 layers v1.3 on top
with the same test infrastructure.

No criterion is a unit test — each exercises a real engramd instance with real
plugin calls. Tests that touch the dreaming worker or ingest worker use a test
store on a temp path; they do not share state across test files.

The criterion-to-WP delivery map below documents which WP makes each test
passable; the test file names follow the criterion slug.

---

## SC -> Test File -> Delivering WP

| SC | Criterion summary (from §12.3) | Test file | Delivering WP |
|----|-------------------------------|-----------|---------------|
| SC-1 | `engram init` scaffolds a store (global or project), idempotent | `tests/e2e/sc01-init-idempotent.test.ts` | WP01 |
| SC-2 | Agent `remember`s a fact; later session `recall`s it with scored ranking and confidence-multiplier breakdown | `tests/e2e/sc02-remember-recall-scored.test.ts` | WP03, WP04 |
| SC-3 | Raw PDF in `raw/` is `ingest`ed (worker job) into typed Markdown memories with `sources:` provenance + LLM-built tags | `tests/e2e/sc03-ingest-pdf-to-memories.test.ts` | WP07 |
| SC-4 | Capture hooks record a Claude Code session's observations + failures into `staging/` per-session JSONL; privacy filter strips a planted secret (filter audit log records the strip) | `tests/e2e/sc04-capture-staging-privacy-filter.test.ts` | WP06 |
| SC-5 | A dreaming run distills staging into memories with `derived_from` backlinks, inserts links, re-weights importance, writes >=1 procedural memory from a failure pattern at `confidence: 0.3`; a second corroborating episode promotes it to active | `tests/e2e/sc05-dreaming-distill-procedural-gate.test.ts` | WP08 |
| SC-6 | Two-layer contradiction detection catches a planted contradiction and routes it to the review queue | `tests/e2e/sc06-contradiction-detection-queue.test.ts` | WP08 |
| SC-7 | `forget` moves a low-score memory to dormant after >=2 consecutive runs below threshold; still searchable on demand; never deleted | `tests/e2e/sc07-forget-dormant-not-deleted.test.ts` | WP04, WP08 |
| SC-8 | A concurrent write losing an OCC race is rejected, retried cleanly, and the conflict logged in AppLog | `tests/e2e/sc08-occ-conflict-retry-applog.test.ts` | WP01 |
| SC-9 | `memory.history <id>` shows per-field provenance with dream_run linkage | `tests/e2e/sc09-history-provenance-dream-link.test.ts` | WP01, WP08 |
| SC-10 | Access control: agent B cannot recall agent A's `private` memory; a planted cross-agent dreaming attempt is denied in v1 | `tests/e2e/sc10-access-control-private-cross-agent.test.ts` | WP05, WP08 |
| SC-11 | A dream worker crash leaves core daemon healthy; orphaned job transitions to `TIMED_OUT`; `engram dream resume` re-runs idempotently from checkpoint with no staging loss | `tests/e2e/sc11-worker-crash-recovery.test.ts` | WP08 |
| SC-12 | A planted prompt-injection in a memory body fails to mutate frontmatter because schema validation rejects the injected fields | `tests/e2e/sc12-prompt-injection-schema-rejected.test.ts` | WP08, WP09 |
| SC-13 | `dream.trigger` from an agent outside the dreaming-memory's scope returns `scope-denied` | `tests/e2e/sc13-dream-trigger-scope-denied.test.ts` | WP05, WP08 |
| SC-14 | Active-pool floor blocks a mass-archive dream run regardless of `merge_policy: always-auto` | `tests/e2e/sc14-active-pool-floor.test.ts` | WP08 |
| SC-15 | `engram doctor` detects a planted broken-frontmatter file and quarantines it without breaking daemon startup | `tests/e2e/sc15-doctor-quarantine-broken-fm.test.ts` | WP01 |
| SC-16 | `governance_delete --purge-history` purges a secret from working tree, git history, AppLog (tombstoned), QMD index, and graphify graph | `tests/e2e/sc16-governance-delete-purge-history.test.ts` | WP10 |
| SC-17 | Streamable HTTP MCP on `127.0.0.1` accepts a bearer token, denies missing/bad tokens, exposes 16 verbs + `engram://system/status` resource | `tests/e2e/sc17-mcp-bearer-16-verbs-resource.test.ts` | WP05 |
| SC-18 | `engram status` reports daemon healthy with plugin health for QMD, graphify, LLM | `tests/e2e/sc18-status-plugin-health.test.ts` | WP05 + WP07 (graphify plugin health is not live until WP07; the `system.status` resource scaffold is WP05) |

---

## Target Files

- `tests/e2e/sc01-init-idempotent.test.ts`
- `tests/e2e/sc02-remember-recall-scored.test.ts`
- `tests/e2e/sc03-ingest-pdf-to-memories.test.ts`
- `tests/e2e/sc04-capture-staging-privacy-filter.test.ts`
- `tests/e2e/sc05-dreaming-distill-procedural-gate.test.ts`
- `tests/e2e/sc06-contradiction-detection-queue.test.ts`
- `tests/e2e/sc07-forget-dormant-not-deleted.test.ts`
- `tests/e2e/sc08-occ-conflict-retry-applog.test.ts`
- `tests/e2e/sc09-history-provenance-dream-link.test.ts`
- `tests/e2e/sc10-access-control-private-cross-agent.test.ts`
- `tests/e2e/sc11-worker-crash-recovery.test.ts`
- `tests/e2e/sc12-prompt-injection-schema-rejected.test.ts`
- `tests/e2e/sc13-dream-trigger-scope-denied.test.ts`
- `tests/e2e/sc14-active-pool-floor.test.ts`
- `tests/e2e/sc15-doctor-quarantine-broken-fm.test.ts`
- `tests/e2e/sc16-governance-delete-purge-history.test.ts`
- `tests/e2e/sc17-mcp-bearer-16-verbs-resource.test.ts`
- `tests/e2e/sc18-status-plugin-health.test.ts`
- `tests/e2e/helpers/daemon.ts` — test harness: start/stop engramd against a temp store, mint test agent tokens

---

## Verified Evidence

- — (all criteria defined in SPEC §12.3; no prior test suite exists)

---

## Implementation Steps

| Step | File | State |
|------|------|-------|
| 1. Write `tests/e2e/helpers/daemon.ts`: `startDaemon(tmpDir)` starts engramd with `--store <tmpDir>`, waits for MCP port open, returns `{client, agentToken, stop()}`; `mintAgent(client, caps)` mints additional tokens for multi-agent tests | `tests/e2e/helpers/daemon.ts` | TODO |
| 2. SC-1: call `engram init --store <tmp>`; assert directory structure (`raw/`, `staging/`, `memories/`, `.engram/`); call again with same path; assert exit 0, no duplicate files | `tests/e2e/sc01-init-idempotent.test.ts` | TODO |
| 3. SC-2: `memory.remember({type:"semantic", title:"JWT fact", body:"...", confidence:0.8})`; in a new client session call `memory.recall({query:"JWT"})`; assert hit with `score.{importance,relevance,recency,m_v,total}` all non-null | `tests/e2e/sc02-remember-recall-scored.test.ts` | TODO |
| 4. SC-3: copy a small test PDF into `<store>/raw/test.pdf`; call `memory.ingest({rawPath:"<store>/raw/test.pdf"})`; poll `dream.status` until MERGED; assert >=1 file in `memories/` with `origin:ingested`, `sources:["raw/test.pdf"]`, `ingest_run_id` set, non-empty `tags` | `tests/e2e/sc03-ingest-pdf-to-memories.test.ts` | TODO |
| 5. SC-4: POST a synthetic capture hook payload containing a planted AWS key string (`AKIAIOSFODNN7EXAMPLE`) to `/capture-intake`; assert `staging/<agent>/<session>.jsonl` does not contain the key string; assert AppLog `capture-audit` row records the filter match | `tests/e2e/sc04-capture-staging-privacy-filter.test.ts` | TODO |
| 6. SC-5 (first run): seed staging with two synthetic observations including a failure trace; trigger dream; await MERGED; assert >=1 `procedural` memory with `confidence:0.3` and `lifecycle:dormant` | `tests/e2e/sc05-dreaming-distill-procedural-gate.test.ts` | TODO |
| 7. SC-5 (second run): add a second corroborating staging observation; trigger dream again; assert the same procedural memory promoted to `lifecycle:active` | `tests/e2e/sc05-dreaming-distill-procedural-gate.test.ts` | TODO |
| 8. SC-6: seed staging with two semantically similar but contradicting claims; trigger dream; assert manifest has `{kind:"contradiction", action:"queue_review"}`; assert neither memory file has a `contradicts` edge (v1: contradicts edge kind is v2) | `tests/e2e/sc06-contradiction-detection-queue.test.ts` | TODO |
| 9. SC-7: set a memory's `importance:0.01`; run two dream cycles (scoring below threshold each); assert `lifecycle:dormant`; call `memory.recall({query:"...", includeDormant:true})` and assert memory appears; assert `lifecycle` never equals `deleted` | `tests/e2e/sc07-forget-dormant-not-deleted.test.ts` | TODO |
| 10. SC-8: two concurrent clients both read the same memory at version N; client A writes first (version N+1 succeeds); client B writes with version N; assert `version-conflict`; assert AppLog has `occ_conflict` event; assert client B succeeds on retry with version N+1 | `tests/e2e/sc08-occ-conflict-retry-applog.test.ts` | TODO |
| 11. SC-9: `memory.remember` then trigger a dream that modifies `importance`; call `memory.history({id})`; assert events list includes write event (original) and a second event with `dream_run_id` set | `tests/e2e/sc09-history-provenance-dream-link.test.ts` | TODO |
| 12. SC-10 (private): agent A writes `memory.remember({visibility:"private",...})`; agent B calls `memory.recall({query:"..."})` with a different token; assert 0 hits for A's private memory | `tests/e2e/sc10-access-control-private-cross-agent.test.ts` | TODO |
| 13. SC-10 (cross-agent): agent B calls `dream.configure({name:"A-dreaming", config:{mode:"cross-agent"}})`; assert `invalid-config` (C11: v1 rejects cross-agent mode) | `tests/e2e/sc10-access-control-private-cross-agent.test.ts` | TODO |
| 14. SC-11: start a dream job; SIGKILL the worker process mid-run; wait for heartbeat watchdog to fire (5 min timeout or accelerated in test); assert job state = `TIMED_OUT`; call `dream.trigger({name, resumeJobId})`; assert new job reaches MERGED; assert staging watermark is unchanged (no observation loss) | `tests/e2e/sc11-worker-crash-recovery.test.ts` | TODO |
| 15. SC-12: write a memory whose body contains the string `---\nimportance: 1.0\n---`; trigger dream; assert merged memory file has `importance` at LLM-rated value, not 1.0; assert no frontmatter mutation | `tests/e2e/sc12-prompt-injection-schema-rejected.test.ts` | TODO |
| 16. SC-13: agent B sends `dream.trigger({name:"A-scoped-dreaming"})` where dreaming-memory `scope` excludes B; assert `scope-denied`; assert AppLog has `mcp_denied` event | `tests/e2e/sc13-dream-trigger-scope-denied.test.ts` | TODO |
| 17. SC-14: create store with 80 active memories; set `merge_policy: always-auto`; trigger dream that attempts to archive 70; assert merge validator blocks it; job -> FAILED; active count still >=min(100,20% total) floor | `tests/e2e/sc14-active-pool-floor.test.ts` | TODO |
| 18. SC-15: write a file to `memories/semantic/broken.md` with invalid YAML frontmatter; call `engram doctor` or start daemon; assert `broken.md` moved to `.engram/quarantine/`; assert daemon (re)starts successfully | `tests/e2e/sc15-doctor-quarantine-broken-fm.test.ts` | TODO |
| 19. SC-16: write memory containing `SECRET_TOKEN=abc123`; call `memory.governance_delete({id, justification:"test", purgeHistory:true})`; assert: file absent from working tree; `git log -S SECRET_TOKEN` returns no commits; AppLog has tombstone event; `memory.recall({query:"SECRET_TOKEN"})` returns 0 hits; `GraphPlugin.traverse` does not return the deleted node | `tests/e2e/sc16-governance-delete-purge-history.test.ts` | TODO |
| 20. SC-17: enumerate all 16 MCP verbs from §6.3; call each with valid token; assert 200 (or defined non-auth error); call each with missing token; assert 401; subscribe to `engram://system/status`; assert subscription update received | `tests/e2e/sc17-mcp-bearer-16-verbs-resource.test.ts` | TODO |
| 21. SC-18: call `engram status` CLI; assert JSON output contains `plugins` array with entries for `qmd`, `graphify`, `llm` each with `health.ok: true` | `tests/e2e/sc18-status-plugin-health.test.ts` | TODO |
| 22. Add `e2e` npm script (`vitest run tests/e2e/` or jest equivalent); add CI job that runs E2E suite as a gate after all WP build jobs pass | `package.json` | TODO |

---

## Verification

The gate is: **all 18 SC test files pass with 0 failures**.

### Build Gates

| Gate | Command | Expected |
|------|---------|----------|
| TypeScript compiles | `npm run build` | exit 0 |
| All E2E tests pass | `npm run e2e` | 18 files, 0 failures, 0 skips |
| No test is empty or skipped | `grep -rn "it.skip\|xit\|todo(" tests/e2e/` | 0 matches |

### Specific Tests

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| SC-1 | Init idempotent | Store dirs exist; second init exits 0, no duplicate | `sc01-init-idempotent.test.ts` |
| SC-2 | Remember + scored recall | Hit returned with `score.{importance,relevance,recency,m_v,total}` all present | `sc02-remember-recall-scored.test.ts` |
| SC-3 | PDF ingest -> typed memories | >=1 memory with `origin:ingested`, `sources:["raw/test.pdf"]`, non-empty `tags` | `sc03-ingest-pdf-to-memories.test.ts` |
| SC-4 | Capture + privacy filter | Planted AWS key absent from staging; AppLog filter strip recorded | `sc04-capture-staging-privacy-filter.test.ts` |
| SC-5 | Dreaming distill + procedural gate | Uncorroborated procedural at `confidence:0.3`, `lifecycle:dormant`; second run promotes to `lifecycle:active` | `sc05-dreaming-distill-procedural-gate.test.ts` |
| SC-6 | Contradiction -> review queue | Manifest `queue_review` entry; no `contradicts` edge in memory file | `sc06-contradiction-detection-queue.test.ts` |
| SC-7 | Forget -> dormant, never deleted | `lifecycle:dormant` after 2 runs below threshold; searchable with `includeDormant:true`; no `lifecycle:deleted` | `sc07-forget-dormant-not-deleted.test.ts` |
| SC-8 | OCC conflict retried + logged | `version-conflict` returned; `occ_conflict` in AppLog; retry succeeds | `sc08-occ-conflict-retry-applog.test.ts` |
| SC-9 | History with dream_run linkage | AppLog event for dream modification has `dream_run_id` set | `sc09-history-provenance-dream-link.test.ts` |
| SC-10 | Private memory invisible; cross-agent mode rejected | 0 recall hits for private memory from other agent; `invalid-config` for cross-agent mode | `sc10-access-control-private-cross-agent.test.ts` |
| SC-11 | Worker crash -> TIMED_OUT -> idempotent resume | Job TIMED_OUT after stale heartbeat; resume succeeds; staging watermark unchanged | `sc11-worker-crash-recovery.test.ts` |
| SC-12 | Prompt injection rejected by schema | Merged memory `importance` != injected value; no frontmatter mutation | `sc12-prompt-injection-schema-rejected.test.ts` |
| SC-13 | dream.trigger scope-denied | `scope-denied` returned; `mcp_denied` in AppLog | `sc13-dream-trigger-scope-denied.test.ts` |
| SC-14 | Active-pool floor gates always-auto | Merge blocked; active count at floor | `sc14-active-pool-floor.test.ts` |
| SC-15 | Doctor quarantines broken frontmatter | File moved to `.engram/quarantine/`; daemon starts OK | `sc15-doctor-quarantine-broken-fm.test.ts` |
| SC-16 | governance_delete purges all five stores | File absent; git clean; AppLog tombstoned; QMD 0 hits; graph node absent | `sc16-governance-delete-purge-history.test.ts` |
| SC-17 | 16 MCP verbs + bearer + resource subscription | All verbs callable with valid token; bad token -> 401; subscription event received | `sc17-mcp-bearer-16-verbs-resource.test.ts` |
| SC-18 | `engram status` all plugins healthy | `plugins[{qmd,graphify,llm}].health.ok === true` | `sc18-status-plugin-health.test.ts` |

---

## Recommended Agents

| Phase | Agent | Rationale |
|-------|-------|-----------|
| All | tdd-guide | E2E tests are written first as failing acceptance tests; each SC criterion has well-formed I/O |
| All | code-reviewer | Review test harness `helpers/daemon.ts` and verify each test exercises the real code path |

---

## Review

| Reviewer | Verdict | Notes |
|----------|---------|-------|
| — | — | — |

---

## Follow-up Issues

| Issue | Severity | Status |
|-------|----------|--------|
| — | — | — |
