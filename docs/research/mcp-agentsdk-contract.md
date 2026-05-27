---
title: "engram — MCP Contract & Dreaming-Worker Research"
project: engram
created: 2026-05-22
agent: research-analyst
purpose: validate engram's 16-verb MCP contract; resolve OQ-J (Agent SDK vs raw API)
---

# Research Report: MCP Contract Validation and Dreaming Worker Architecture for engram

**Date:** 2026-05-22
**Scope:** Validates engram's 16-verb MCP contract, 5-capability model, and {ok,data,error} envelope against the MCP specification (2025-06-18). Resolves OQ-J (Claude Agent SDK headless vs raw API loop). Covers MCP server design, capability-gating, tool conventions, resources vs tools, Agent SDK for the dreaming worker, and provider abstraction.

**Primary sources:** modelcontextprotocol.io spec 2025-06-18 · code.claude.com/docs/en/agent-sdk · ai-sdk.dev · platform.claude.com prompt-caching docs.

---

## 1. MCP Server Design Best Practice

### 1.1 Protocol & message structure
MCP is JSON-RPC 2.0, UTF-8 newline-delimited. Spec (2025-06-18) defines two standard transports: **stdio** and **Streamable HTTP** (the latter replaced legacy HTTP+SSE). The TS SDK (`@modelcontextprotocol/sdk`) provides `McpServer`, tool registration with Standard Schema (Zod/Valibot/ArkType), and transport adapters.

### 1.2 Tool I/O schemas
Spec defines `inputSchema` (JSON Schema, required) and `outputSchema` (optional, 2025-06-18). With an `outputSchema`, the server MUST return `structuredContent` conforming to it AND (for back-compat) serialize it into a `TextContent` block. Tool response shape: `{ content: [...], isError: false, structuredContent: {...} }`.

### 1.3 Error semantics — the two-level model
- **Level 1 — Protocol errors** (JSON-RPC `error` field): unknown method, malformed request, server crash. Codes: `-32602` (unknown tool/bad args), `-32603` (internal), `-32002` (resource not found). Aborts the RPC.
- **Level 2 — Tool execution errors** (`result.isError: true` with human-readable `content`): business-logic failures within a dispatched call. The JSON-RPC layer sees success; `isError` tells the LLM the tool's work failed, so it can reason about retrying/degrading.

### 1.4 engram's {ok,data,error} envelope — alignment
engram's envelope is a **layer conflict with MCP wire conventions**. Resolution: keep `{ok,data,error}` as an **internal `CoreService` type**; the MCP adapter translates:
- `ok:true` → `{ content:[...], isError:false, structuredContent: data }`
- `ok:false, protocol error` → JSON-RPC `error` response
- `ok:false, tool-execution error` → `{ content:[{type:"text",text:error.message}], isError:true }`

This preserves engram's internal uniformity while being MCP-idiomatic at the wire level. `structuredContent` + `outputSchema` is the right home for engram's typed responses (e.g. the `recall` hit array).

### 1.5 Transport recommendation: **Streamable HTTP over `127.0.0.1`**
engram's daemon is always-up and serves multiple per-session agents — this is the multi-client Streamable HTTP use case, not single-client stdio. Bearer-token auth (`Authorization: Bearer <agentToken>`) is natural on HTTP; the spec recommends localhost binding (DNS-rebinding protection); `Mcp-Session-Id` maps to engram's session-scoped memory. **Do not use unix domain sockets as an MCP transport** — no SDK support; the SO_PEERCRED plan would require significant custom-transport code. Streamable HTTP + bearer token is simpler, portable, spec-aligned.

## 2. Capability-Gating
**MCP has no built-in per-tool/per-caller access control.** The `initialize` capability negotiation is about protocol features, not caller permissions. Tool-level access control is a server-side implementation concern.

engram's per-agent token (minted at `memory.init`, mapped server-side to `(agentId, capabilitySet)`) is sound. On Streamable HTTP, the standard mechanism is `Authorization: Bearer <agentToken>` — engram acts as its own authorization server (issuing opaque tokens) and resource server (validating them), valid for a local-only tool. **Drop SO_PEERCRED** in favour of bearer tokens. Prior art: the Sentry MCP server uses bearer tokens on HTTP transport. No MCP spec/SDK provides a richer per-tool gating mechanism — this is the state of the art.

## 3. Tool Contract Conventions
- **16-verb set validated as sound.** Dot-notation namespacing (`memory.*`, `dream.*`, `system.*`) is idiomatic. 16 verbs is larger than typical (most servers 3–8) but appropriate for a domain-specific server with a rich contract.
- **Declare `outputSchema` for every verb** — enables client-side validation and type-safe consumption. The architecture review's typed returns map directly.
- **Async verbs (`ingest`, `dream.trigger` → `jobId`)** — MCP has no native async job model (experimental "Tasks" still experimental). The `trigger → jobId / status poll / result fetch` triad is current best practice. Correct as designed.
- **Tool annotations** — use the `annotations` field (`destructive: true`, `requiresHumanApproval: true`) on `memory.governance_delete` so clients prompt the user.

## 4. Resources vs Tools vs Prompts
- **Tools** = model-controlled executable functions.
- **Resources** = application-controlled read-only data sources with URIs, subscribable for change notifications.
- **Recommendation:** expose `system.status` as **both** a resource (`engram://system/status`, with subscription so a monitor learns of plugin state changes) **and** a tool (LLM-invoked introspection). Expose memories as URI-template resources (`engram://memories/{id}`) for monitoring — v2. Keep dreaming-memory config as tools (`dream.list`/`dream.configure`), not resources, in v1.

## 5. Claude Agent SDK for the Dreaming Worker — CRITICAL

### 5.1 What the Agent SDK provides
`@anthropic-ai/claude-agent-sdk` (TS) — builds agents running the Claude Code tool loop as a library. `query({prompt, options})` async generator; built-in tools; `allowedTools`/`disallowedTools`; `permissionMode` (incl. `dontAsk`); `maxTurns`; `maxBudgetUsd`; `outputFormat: {type:"json_schema", schema}`; `systemPrompt`; session resume/fork; hooks; env-var timeouts; `AbortController`.

### 5.2 Headless capability
The SDK runs fully headless: `permissionMode:"dontAsk"` + `allowedTools` = locked-down, zero human prompts. `maxBudgetUsd` enforces a per-dream budget; `outputFormat:json_schema` captures the dream manifest as typed output; the `result` message's `subtype` (`success`/`error_max_turns`/`error_max_budget_usd`/`error_during_execution`) distinguishes terminal states for the job state machine; `AbortController` kills a runaway dream. Designed for CI/CD automation. Recommended worker pattern: a **separate Node process** (not worker_threads) — preserves process isolation.

### 5.3 Raw Anthropic API loop — comparison
A raw `@anthropic-ai/sdk` loop gives full control: the tool-use loop, budget tracking, structured-output parsing, retry, and — critically — **prompt caching** (`cache_control` on system prompt + tool defs, ~90% input-cost reduction on cache hits, 5-min default / 1-hour TTL). The Agent SDK does **not expose prompt-caching settings** at the `query()` level.

### 5.4 DECISION: raw Anthropic API loop (via Vercel AI SDK / LlmPlugin), NOT the Agent SDK
The decisive factor is **provider abstraction**. The Agent SDK is Anthropic-only (Bedrock/Vertex backends are still Claude). engram's `LlmPlugin` must support Claude + OpenAI + Ollama. Hardcoding the Agent SDK in the dreaming worker means a user configuring `llm: openai`/`ollama` cannot dream — violating the LLM-provider seam. Correct architecture: the dreaming worker calls `llmPlugin.complete(...)`, implemented by `ClaudeLlmPlugin`/`OpenAiLlmPlugin`/`OllamaLlmPlugin`. The Agent SDK remains a valid **v2 opt-in** for an explicitly Anthropic-native dreaming mode.

## 6. Provider Abstraction for LlmPlugin
**Use the Vercel AI SDK (`ai` package)** as the substrate behind `LlmPlugin`. Unified `generateText`/`generateObject`/`embed`/`embedMany`; official `@ai-sdk/openai`, `@ai-sdk/anthropic`; community Ollama providers. Map: `complete` → `generateText`/`generateObject`; `embed` → `embedMany`. Three impls — `ClaudeLlmPlugin`, `OpenAiLlmPlugin`, `OllamaLlmPlugin` — all expose the same interface.
- **Ollama caveat:** the basic community provider has tool-calling reliability issues; use the `ai-sdk-ollama` (jagreehal) variant with reliable tool calling.
- **Prompt caching:** for `ClaudeLlmPlugin`, use raw `@anthropic-ai/sdk` internally to get clean `cache_control` placement (1-hour cache on the dreaming system prompt → ~90% cost reduction on turns 2+); the `LlmPlugin` interface hides this choice.
- Rolling your own for 3 providers is ~300 lines of boilerplate with real edge cases; the AI SDK is battle-tested, MIT, TS-native.

## 7. Open Question Resolution
- **OQ-J** → **RESOLVED:** dreaming worker = raw Anthropic API loop (via Vercel AI SDK / `LlmPlugin`), not the Agent SDK. Agent SDK is Anthropic-only and bypasses the provider seam.
- **OQ-E** → per-agent bearer token + Streamable HTTP. `memory.init` mints an opaque token → server lookup → `(agentId, capabilities, scope)`.
- **OQ-B** → `memory.*`/`dream.*`/`system.*` namespacing confirmed idiomatic.

## 12-Line Summary
**MCP contract verdict:** engram's 16-verb contract is validated as sound and complete. The 5-capability model, verb set, and async `jobId` triad are MCP-idiomatic. The `{ok,data,error}` envelope is a layer conflict — keep it internal to `CoreService`, translate to MCP's native `isError`/`structuredContent` at the wire level. Declare `outputSchema` on all verbs. **Switch transport from stdio to Streamable HTTP over `127.0.0.1`** (multi-client daemon, bearer-token auth, spec-aligned). **Replace the SO_PEERCRED plan with `Authorization: Bearer <agentToken>`** — per-agent tokens minted at `memory.init` are sound and consistent with production MCP servers. **Dreaming worker: raw Anthropic API loop via the Vercel AI SDK / `LlmPlugin`, NOT the Claude Agent SDK** — the Agent SDK is feature-complete for headless execution but Anthropic-only, which breaks OpenAI+Ollama support on day one and bypasses the provider seam. For `ClaudeLlmPlugin`, use raw `@anthropic-ai/sdk` internally for 1-hour `cache_control` on the dreaming system prompt (~90% cost reduction on multi-turn loops). The Agent SDK is a valid v2 opt-in for an Anthropic-native dreaming mode but must not be the default.
