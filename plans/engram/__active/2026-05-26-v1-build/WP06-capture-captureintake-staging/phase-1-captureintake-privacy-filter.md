---
name: phase-1-captureintake-privacy-filter
title: Kernel CaptureIntake — 4-layer privacy filter, fail-closed, filter.audit_log, /capture-intake endpoint
type: phase
phase_status: pending
wp: wp06-capture-captureintake-staging
goal: CaptureIntake receives POST /capture-intake (bearer-gated), runs the 4-layer privacy filter (regex/entropy/path-blocklist/optional-LLM) in fail-closed mode, appends a stripped observation to the staging path, and records every suppression in filter.audit_log with the matched rule but no payload content.
verify: "npm test tests/integration/capture-intake — a payload containing a planted AWS_SECRET_ACCESS_KEY has the key REDACTED (replaced with ‹redacted:aws-key›) and the rest of the observation IS written to staging (OQ-01); filter.audit_log records {rule:'aws-key', layer, ts, agent_id} with no payload content; a forced filter-layer exception DROPS the whole observation (fail-closed); an unfiltered payload passes through unchanged."
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

**Two paths by trigger (OQ-01, resolved 2026-05-26):**
- **Filter MATCH** (a layer detects a secret in an otherwise-fine observation) →
  **redact the matched span** in place (replace with a `‹redacted:<rule>›`
  marker) and write the *rest* of the observation to staging. The audit log
  records the rule + layer (never the matched content). This is what SC-4
  asserts: the secret is stripped, the observation still reaches staging.
- **Filter ERROR/timeout** (a layer throws/hangs) → **fail-closed: drop the whole
  observation** — partial filtering can't be trusted when the filter itself
  failed. Log at ERROR **without payload content**.

In both cases the error is never propagated to the hook process (hook always sees
200 / exit 0). The §6.1 "fail-closed ⇒ drop" rule governs the *error* path, not
the *match* path.

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
| `PrivacyFilter` class: regex layer (AWS/GitHub/JWT/PEM patterns), entropy layer, path blocklist; each layer returns matched spans for redaction | `src/core/privacy-filter.ts` | TODO |
| Match path: redact matched spans in place (`‹redacted:<rule>›`), pass the rest (OQ-01) | `src/core/privacy-filter.ts` | TODO |
| Entropy layer: configurable `entropy_min_len`(default 40)/`entropy_threshold`; allowlist benign high-entropy shapes (git SHA, UUID, base64 data-URI, paths); calibrate default FP rate against a corpus of real captured payloads before locking it (OQ-05) | `src/core/privacy-filter.ts` | TODO |
| Optional LLM sweep stub (off by default; config flag `capture.llm_sweep: false`) | `src/core/privacy-filter.ts` | TODO |
| Fail-closed wrapper: any layer **throws/timeouts** → drop whole observation + log ERROR (no payload). Distinct from the match path (OQ-01) | `src/core/privacy-filter.ts` | TODO |
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
