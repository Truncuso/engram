---
name: phase-4-capture-observability
title: Capture-audit AppLog channel + engram capture status CLI
type: phase
phase_status: pending
wp: wp06-capture-captureintake-staging
goal: Every hook invocation (including suppressions and filter strips) is recorded in AppLog as a capture event with no payload content; engram capture status reports active session, staging backlog count+bytes, filter suppressions in the last hour by rule, capture-fallback size, and hook failure count.
verify: "engram capture status exits 0 and prints structured output with all five fields; a simulated filter suppression appears in 'filter suppressions (last hour)' output; AppLog has a capture event row for each hook invocation."
---
<!-- Template: WP-folder PHASE v2 (frontmatter-first) -->

# Phase 4: Capture-audit AppLog channel + engram capture status

**Goal:** Complete capture observability per §10.5:

1. **capture-audit AppLog channel** — every hook invocation (success, suppressed,
   fallback, error) appends an AppLog event with `event_type: 'capture'` and a
   structured `data_json` payload:
   `{hook_type, agent_id, session_id, passed: bool, suppression_rule?, layer?,
   fallback: bool, hook_error: bool}`. No payload content, no secrets.

2. **`engram capture status`** CLI command (§10.5) reports:
   - Active session (session_id + agent_id).
   - Staging backlog (file count + bytes).
   - Filter suppressions last hour (count per rule).
   - `capture-fallback/` size (bytes + file count).
   - Hook failure count (exit non-0 or timeout, last hour).

**Verify:** `engram capture status` exits 0 and prints all five fields; a
simulated filter suppression appears in suppressions output; AppLog has a
`capture` event row for the invocation.

## Steps

| Step | File | State |
|------|------|-------|
| Extend AppLog `event_type` enum to include `'capture'` (already reserved in §10.2; wire here) | `src/core/applog.ts` | TODO |
| `CaptureAuditLogger.log(event)` — writes capture AppLog row with `data_json` (no payload) | `src/core/capture-audit.ts` | TODO |
| Wire `CaptureAuditLogger` into `CaptureIntake`: log on pass, on suppression, on fallback write, on filter error | `src/core/capture-intake.ts` | TODO |
| `engram capture status` CLI command: queries AppLog + staging dir stats + fallback dir size | `src/cli/commands/capture-status.ts` | TODO |
| Unit/integration tests: audit log row on suppression; capture status output fields | `tests/integration/capture-observability.test.ts` | TODO |

## Notes

This phase completes the capture pipeline and SC-4. The `capture-audit` channel
is distinct from the main AppLog mutation stream (different `event_type`); `engram
log --type capture` surfaces it. Suppression entries contain `suppression_rule`
(the regex name or "entropy" or "path-blocklist") and the `layer` (1–4) but
never the matched string or surrounding context — this is the §10.2 principle:
"no silent capture." Hook failure count is derived from AppLog rows where
`data_json.hook_error: true`; it does not require a separate counter. The
`capture status` command is the CLI surface for SC-4 / §10.5 verification.
