---
name: wp02-plugin-host-llmplugin-vercel-ai-sdk
title: Plugin host + LlmPlugin (Vercel AI SDK)
type: work-package
stage: ready
severity: HIGH
created: 2026-05-26
updated: 2026-05-26
plan: 2026-05-26-v1-build
tags: [plugin-host, llm, vercel-ai-sdk, anthropic, openai, ollama, transports]
relationships:
  - depends_on: wp01-core-scaffold-store-schemas-applog-jobs-occ-coreservice-doctor-init
  - blocks: wp04-scoring-engine-recall-degradation-chain
  - blocks: wp07-ingest-worker-graphify-graphplugin-ollama
  - blocks: wp08-dreaming-worker-orchestrator
sources: [SPEC-v2.1-§2.3, SPEC-v2.1-§2.4, SPEC-v2.1-§9.9, SPEC-v2.1-§11.1, ADR-0003, Spike-1b]
---
<!-- Template: WP v2 (frontmatter-first) -->

# WP02: Plugin host + LlmPlugin (Vercel AI SDK)

## Problem

The kernel needs a Plugin Host that loads, validates, and supervises plugins via
two transports — in-process TS calls and subprocess JSON-RPC over stdio — behind a
single contract (SPEC §2.3, §2.4). Without this seam, LLM providers, retrieval,
and graph plugins cannot be wired or swapped. The LLM seam must expose
`complete`/`embed` via Vercel AI SDK's `generateObject`/`embed` for all three
providers, with `ClaudeLlmPlugin` using raw `@anthropic-ai/sdk` directly so
`cache_control` headers are reachable (a deliberate, documented abstraction leak
confirmed by Spike 1b; ADR-0003). Per C13/§9.9 step 4, LLM init failure must
demote the daemon to degraded — not fatal — so `memory.get/list/history/recall`
(grep fallback) remain available; dreaming and ingest are suspended until the LLM
plugin is healthy.

---

## Target Files

- `src/core/plugin-host/manifest.ts` — `PluginManifest`, `PluginKind`, `Transport`, `PluginError`, `PLUGIN_CONTRACT_VERSION = "1.0"` types and constants
- `src/core/plugin-host/lifecycle.ts` — `PluginLifecycle` interface; `PluginHost` class: `register`, `init`, `shutdown`, `contractVersion` check (mismatch → `fatal` error)
- `src/core/plugin-host/transport-inproc.ts` — In-process transport adapter: holds a direct TS module reference, forwards calls synchronously
- `src/core/plugin-host/transport-subproc.ts` — Subprocess JSON-RPC transport: `child_process.spawn` with args array (never shell strings — S-16); framed newline-delimited JSON over stdio; restart-on-crash logic
- `src/core/plugin-host/health.ts` — Scheduled health prober: polls each registered plugin's `health()` on interval; marks in-proc plugins degraded; restarts subprocess plugins on `unavailable`
- `src/plugins/llm/index.ts` — `LlmPlugin` interface (`complete`, `embed`, `init`, `health`, `shutdown`); `LlmOpts`, `LlmResult`, `Prompt` types; plugin registry/factory
- `src/plugins/llm/claude.ts` — `ClaudeLlmPlugin`: uses raw `@anthropic-ai/sdk` (not Vercel AI SDK) for `complete` so `cache_control` is available; uses `ai` SDK `embed` for embeddings; documented leak
- `src/plugins/llm/openai.ts` — `OpenAiLlmPlugin`: Vercel AI SDK `generateObject`/`embed` via `@ai-sdk/openai`; structured-output-capable model required
- `src/plugins/llm/ollama.ts` — `OllamaLlmPlugin`: Vercel AI SDK via `ollama-ai-provider-v2`; structured-output-capable model required; worker validates output vs Zod schema, violation → job FAILED (C6)

---

## Verified Evidence

- `Spike-1b:confirmed` — `ai@6.0.191` `generateObject`/`embed`/`streamObject` verified live; providers coexist (`@ai-sdk/openai`, `ollama-ai-provider-v2`); SDK correctly rejects schema-noncompliant model output (e.g. `importance:"high"` string instead of float)
- `Spike-1b:confirmed` — `@anthropic-ai/sdk@0.98` has `cache_control` in types; `ClaudeLlmPlugin` raw-SDK path is reachable and confirmed
- `Spike-1b:confirmed` — OpenAI path end-to-end (failed only on billing quota, not code); Ollama transport works
- `ADR-0003:accepted` — Vercel AI SDK as substrate; raw Anthropic SDK inside ClaudeLlmPlugin; no agent framework
- `SPEC-§2.3:specified` — `PluginManifest`/`PluginLifecycle`/`PluginError` types; `contractVersion` mismatch = `fatal`; health probed on schedule
- `SPEC-§2.4:specified` — two transports (in-process TS, subprocess JSON-RPC stdio); same contract for both
- `SPEC-§9.9-C13:specified` — LLM init failure = degraded (not fatal); dreaming/ingest return `plugin-unavailable`; recall continues via grep fallback

---

## Implementation Steps

| Step | File | State |
|------|------|-------|
| 1. Define plugin contract types and `PLUGIN_CONTRACT_VERSION = "1.0"` | `src/core/plugin-host/manifest.ts` | TODO |
| 2. Implement `PluginLifecycle` interface and `PluginHost` class with `contractVersion` check (mismatch → throw `PluginError{kind:"fatal"}`) | `src/core/plugin-host/lifecycle.ts` | TODO |
| 3. Implement in-process transport adapter (direct TS module ref, no IPC) | `src/core/plugin-host/transport-inproc.ts` | TODO |
| 4. Implement subprocess JSON-RPC transport (`spawn` args array; newline-delimited JSON framing; restart-on-crash) | `src/core/plugin-host/transport-subproc.ts` | TODO |
| 5. Implement scheduled health prober; degrade in-proc on `ok:false`; restart subproc on `unavailable` | `src/core/plugin-host/health.ts` | TODO |
| 6. Define `LlmPlugin` interface, `Prompt`/`LlmOpts`/`LlmResult` types, factory | `src/plugins/llm/index.ts` | TODO |
| 7. Implement `ClaudeLlmPlugin` with raw `@anthropic-ai/sdk` for `complete` (cache_control reachable); use `ai` SDK for `embed` | `src/plugins/llm/claude.ts` | TODO |
| 8. Implement `OpenAiLlmPlugin` using Vercel AI SDK `generateObject`/`embed` | `src/plugins/llm/openai.ts` | TODO |
| 9. Implement `OllamaLlmPlugin` using Vercel AI SDK + `ollama-ai-provider-v2`; validate Zod schema; violation → reject (C6) | `src/plugins/llm/ollama.ts` | TODO |
| 10. Wire plugin init into daemon startup (§9.9 step 4): LLM init failure → degraded, log, set `dreaming: unavailable` | `src/core/plugin-host/lifecycle.ts` | TODO |

---

## Verification

### Build Gates

| Gate | Command | Expected |
|------|---------|----------|
| TypeScript compiles | `tsc --noEmit` | 0 errors |
| Unit tests pass | `vitest run src/core/plugin-host/ src/plugins/llm/` | All green |
| Lint | `eslint src/core/plugin-host/ src/plugins/llm/` | 0 errors |

### Specific Tests

| ID | Test | Expected Result | Method |
|----|------|-----------------|--------|
| T-WP02-01 | Register a mock in-proc plugin; call `health()` | Returns `{ok: true}` | Unit test |
| T-WP02-02 | Register plugin with `contractVersion: "0.9"` (mismatched) | `init()` throws `PluginError{kind:"fatal"}`; host unloads the plugin | Unit test |
| T-WP02-03 | Mock in-proc plugin returns `health: {ok: false}`; assert host marks it degraded | Host sets plugin state to degraded; does not throw | Unit test |
| T-WP02-04 | `ClaudeLlmPlugin.complete()` with a prompt that includes `cache_control` in message blocks | `cache_control` field passes through to the raw Anthropic SDK call (inspect constructed messages) | Unit test with `@anthropic-ai/sdk` mock |
| T-WP02-05 | `OllamaLlmPlugin.complete()` with a Zod schema; model returns schema-noncompliant output | Plugin throws / returns error; does not return partial structured data | Unit test with mock that returns `{importance: "high"}` |
| T-WP02-06 | `generateObject` round-trip against structured-output-capable model (Ollama local, if available in CI) | Returns valid typed object matching the Zod schema | Integration test (skip if no Ollama) |
| T-WP02-07 | Daemon startup with LLM plugin init throwing | Daemon starts in degraded mode; `memory.get` succeeds; `dream.trigger` returns `plugin-unavailable` | Integration test (SPEC §9.9 C13, SC-implicit) |
| T-WP02-08 | Subprocess transport: spawn a simple echo JSON-RPC responder; call `health()` via transport | Returns the echo response; framing correct | Unit test |

---

## Recommended Agents

| Phase | Agent | Rationale |
|-------|-------|-----------|
| Implementation | typescript-pro | Core plugin host typings, discriminated union errors, generics |
| Review | typescript-reviewer | Verify no `any` leaks across transport boundary; subprocess arg array (S-16) |
| Review | code-reviewer | Transport restart logic; degradation state machine; documented ClaudeLlmPlugin leak |
| Review | security-reviewer | S-16 subprocess args never shell strings; no credential logging in LLM opts |

---

## Review

| Reviewer | Verdict | Notes |
|-----------|---------|-------|
| — | — | — |

---

## Follow-up Issues

| Issue | Severity | Status |
|-------|----------|--------|
| — | — | — |
