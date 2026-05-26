---
name: phase-1-captureintake-privacy-filter
title: Kernel CaptureIntake — 4-layer privacy filter, fail-closed, filter.audit_log, /capture-intake endpoint
type: phase
phase_status: pending
wp: wp06-capture-captureintake-staging
goal: CaptureIntake receives POST /capture-intake (bearer-gated), runs the 4-layer privacy filter (regex/entropy/path-blocklist/optional-LLM) in fail-closed mode, appends a stripped observation to the staging path, and records every suppression in filter.audit_log with the matched rule but no payload content.
verify: "npm test tests/integration/capture-intake — a payload containing a planted AWS_SECRET_ACCESS_KEY is fully dropped (observation not written to staging); filter.audit_log records {rule:'aws-key', ts, agent_id}; an unfiltered payload passes through and reaches the staging append stub."
---
<!-- Template: WP-folder PHASE v2 (frontmatter-first) -->

# Phase 1: CaptureIntake — 4-layer privacy filter + fail-closed + audit_log + endpoint

**Goal:** `CaptureIntake` is a fixed kernel component (not swappable — §2.2).
It exposes `POST /capture-intake` on the existing MCP Express app (bearer-gated
— same token used for MCP verbs). On every request it runs the four-layer
privacy filter (§6.1, S-01):

1. **Regex layer** — known-pattern rules: AWS (`AKIA…`), GitHub (`ghp_…`),
   JWT (`eyJ…`), PEM (`-----BEGIN`), etc.
2. **Entropy layer** — high-entropy strings ≥ N chars (configurable; default 40).
3. **Path blocklist** — strings matching `~/.ssh/`, `~/.aws/`, `/run/secrets/`,
   `~/.gnupg/`, etc.
4. **Optional LLM sweep** — opt-in; off by default (§6.1).

**Fail-closed:** any filter layer error (exception, timeout) → drop the
observation. Log at ERROR level **without payload content**. Never propagate the
error to the hook process (hook always sees 200 / exit 0).

`filter.audit_log` is an AppLog channel recording suppressions: `{event_type:
'capture', ts, agent_id, session_id, rule_matched, layer}` — no payload, no
secret content.

**Verify:** `npm test tests/integration/capture-intake` — AWS key planted in
payload → not written to staging, `filter.audit_log` has a row with `rule:'aws-key'`;
unfiltered payload passes through to staging stub (tested empty write here; phase
2 adds real JSONL).

## Steps

| Step | File | State |
|------|------|-------|
| `PrivacyFilter` class: regex layer (AWS/GitHub/JWT/PEM patterns), entropy layer, path blocklist | `src/core/privacy-filter.ts` | TODO |
| Optional LLM sweep stub (off by default; config flag `capture.llm_sweep: false`) | `src/core/privacy-filter.ts` | TODO |
| Fail-closed wrapper: any layer throws → drop + log ERROR (no payload) | `src/core/privacy-filter.ts` | TODO |
| `filter.audit_log` → AppLog `event_type: 'capture'` with `{rule_matched, layer}` (no payload content) | `src/core/privacy-filter.ts` | TODO |
| `POST /capture-intake` Express route: bearer gate → MAC read → `PrivacyFilter.run()` → staging stub | `src/core/capture-intake.ts` | TODO |
| Integration tests: planted AWS key dropped + audit logged; unfiltered payload passes; filter exception → drop not throw | `tests/integration/capture-intake.test.ts` | TODO |

## Notes

CaptureIntake is registered as a route on the same Express app as the MCP server
(WP05 phase 2). The bearer token is the same per-agent token — the capture hook
presents it exactly as the MCP client does. MAC verification (S-02/G2) is read
in this endpoint but the MAC secret provisioning lives in WP05 (agent add) and
the MAC computation lives in phase 3 (capture plugin). Here, the endpoint reads
the MAC header and passes it to staging metadata; the worker verifies it later.
The LLM sweep is architecture-complete (config flag) but disabled by default —
it requires the LLM plugin (WP02) which is not a dependency of this WP.
