---
name: phase-3-capture-plugin-cc-hooks
title: CapturePlugin install/uninstall/normalise; CC hook scripts fire-and-forget 200ms; PreCompact request/response (G1); MAC attestation from 0600 file (G2)
type: phase
phase_status: pending
wp: wp06-capture-captureintake-staging
goal: The Claude Code CapturePlugin implements the CapturePlugin contract (install/uninstall/normalise); generated hook scripts for PostToolUse/PostToolUseFailure/Stop/SessionEnd/UserPromptSubmit/SubagentStart/SubagentStop fire-and-forget with 200ms hard timeout and exit 0 on any error; PreCompact is a separate request/response hook that calls memory.recall and returns context; MAC attestation reads ~/.engram/agent-secrets/<id>.mac at invocation (G2).
verify: "npm test tests/integration/cc-hooks — install() writes hook scripts to a temp .claude/hooks/ dir; a simulated PostToolUse invocation posts to /capture-intake within 200ms and exits 0 even if engramd returns 500; the MAC header in the posted payload verifies against the known secret; pre-compact.sh returns recall output (not empty when engramd is up)."
---
<!-- Template: WP-folder PHASE v2 (frontmatter-first) -->

# Phase 3: CapturePlugin (Claude Code) + hook scripts + PreCompact + MAC attestation

**Goal:** The `CapturePlugin` for Claude Code implements the `CapturePlugin`
interface from §2.3:
- `install(target)` — writes executable hook scripts into the CC hooks directory.
- `uninstall(target)` — removes the hook scripts.
- `normalise(raw)` — maps the CC hook payload (tool name, args, result, error) to
  `RawObservation`.

**Two hook classes (G1):**

1. **Fire-and-forget hooks** (`PostToolUse`, `PostToolUseFailure`, `Stop`,
   `SessionEnd`, `UserPromptSubmit`, `SubagentStart`, `SubagentStop`): each
   script normalises the CC payload, computes the MAC, and POSTs to
   `POST /capture-intake` with a **200ms hard timeout**. On any error (timeout,
   connection refused, non-2xx) the script exits 0 silently — the host session
   must never stall. If engramd is unreachable the script writes to
   `capture-fallback/` instead.

2. **Re-inject hook** (`PreCompact`) — **NOT fire-and-forget** (G1). This script
   calls `memory.recall` (MCP tool call with 1s bounded timeout) and returns the
   result as hook output for re-injection into the context. If engramd is slow or
   down, it returns empty JSON and exits 0 — the host proceeds with normal
   compaction.

**MAC attestation (G2 / S-02):** Each fire-and-forget script reads the
per-agent MAC secret from `~/.engram/agent-secrets/<agent-id>.mac` (0600,
daemon-written) at invocation, computes `HMAC-SHA256(secret, normalised_payload)`,
includes the result in the `X-Engram-MAC` header, then zeroes the secret from
memory (shell: `unset` + `secret=""`) before exit.

**Verify:** `npm test tests/integration/cc-hooks` — `install()` writes scripts
to temp dir; simulated `PostToolUse` invocation posts to `/capture-intake` within
200ms, exits 0 on HTTP 500; MAC header verifies; `pre-compact.sh` returns recall
output when engramd is up, empty when down.

## Steps

| Step | File | State |
|------|------|-------|
| `CapturePlugin` class: `install`, `uninstall`, `normalise`, `health` per §2.3 contract | `src/plugins/capture/cc/index.ts` | TODO |
| `normalise(raw)` → `RawObservation` type (maps tool/args/result/error/session/ts) | `src/plugins/capture/cc/index.ts` | TODO |
| Fire-and-forget hook script template (Bash): read MAC file, normalise via embedded JS snippet, POST with 200ms timeout, fallback on error, exit 0 | `src/plugins/capture/cc/hooks/fire-and-forget.sh.tmpl` | TODO |
| `install()` renders and writes hook scripts for 7 fire-and-forget events; `chmod +x` | `src/plugins/capture/cc/index.ts` | TODO |
| `pre-compact.sh`: calls `memory.recall` via MCP HTTP with 1s timeout, returns JSON output; empty on timeout/error | `src/plugins/capture/cc/hooks/pre-compact.sh.tmpl` | TODO |
| MAC module: `readMacSecret(agentId)` reads `~/.engram/agent-secrets/<id>.mac` (0600); `computeMac(secret, payload)`; wipe secret after use | `src/plugins/capture/cc/mac.ts` | TODO |
| Integration tests: install/uninstall; fire-and-forget 200ms timeout; MAC verify; PreCompact recall path | `tests/integration/cc-hooks.test.ts` | TODO |

## Notes

The hook scripts are Bash (not TypeScript) because CC hooks are external
processes. The normalise-and-POST logic is a minimal embedded shell pipeline
using `curl` (always present) — no Node.js required in the hook script. The MAC
secret file must exist before hooks can fire; `engram agent add` (WP05 phase 2)
creates it. The daemon writes the MAC secret to `~/.engram/agent-secrets/<id>.mac`
alongside the bearer token (same provisioning step). Session tokens expire at
`SessionEnd` per §8.3; the MAC secret is rotated with `engram agent rotate <id>`
and revoked with `engram agent revoke <id>`.
