# ADR-0003: LLM substrate — Vercel AI SDK behind LlmPlugin, thin (no agent framework)

- **Status:** Accepted
- **Date:** 2026-05-26
- **Related:** ADR-0002, SPEC §11.1, §2.3, plan Part 1c, Spike 1b

## Context

The worker needs an LLM substrate that is multi-provider (Claude, OpenAI, local
**Ollama**), extensible, and flexible enough to later support an autonomous
curation/dreaming-orchestration agent. The question conflated two layers: the
*substrate* (send prompt → structured output) and an *agent framework* (ADK,
LangChain, CrewAI — tool loops, planning). The dreaming worker is a
**deterministic pipeline** (read staging → one structured-output call → validate
→ write manifest), not a multi-step agent — matching the SPEC's existing
rejection of the Agent SDK as worker engine (OQ-J).

Spike 1b verified: `ai@6` exposes `generateObject`/`generateText`/`embed`;
providers coexist (`@ai-sdk/openai`, `ollama-ai-provider-v2`); `@anthropic-ai/sdk`
exposes `cache_control`. It also surfaced that structured-output quality is
model-dependent — the SDK correctly *rejects* non-compliant model output.

## Decision

- **Substrate = thin.** Worker calls the LLM directly with JSON-schema/Zod
  structured output. **No agent framework** in the core product.
- **Implementation = Vercel AI SDK (`ai`)** behind engram's own `LlmPlugin`
  interface (multi-provider routing + structured output + retries). `cache_control`
  uses raw `@anthropic-ai/sdk` inside `ClaudeLlmPlugin` (a contained, documented
  leak).
- **`LlmPlugin` is the stable seam** for a future agentic sub-project, keeping
  substrate (provider swap) and orchestration (agent loop) as separate layers.
- **Operational requirement:** pin a structured-output-capable model per provider;
  the worker validates output against the dream-output schema and treats
  violations as job FAILED (this is also the S-05 prompt-injection enforcement
  path — see SPEC C6).

## Consequences

- Least code for strong multi-provider flexibility; Ollama gives a local/free
  path for cost control.
- Strict schema validation is a feature (injection defense), not a limitation —
  but model selection matters; some cloud-routed models won't honor strict
  structured output.

## Alternatives considered

- **Agent framework (ADK/LangChain/CrewAI) in the worker** — solves problems the
  worker doesn't have; rejected as bloat (Simplicity-First). Reserved for the
  future agentic sub-project.
- **Hand-rolled provider router** — max control/learning, but reimplements
  structured-output parsing + retries the SDK gives free; rejected for v1.
