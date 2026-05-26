---
name: phase-2-staging-jsonl-backpressure
title: Per-session JSONL append (atomic fsync), staging cap 500MB/10k files, capture-fallback 10MB/7d + drain on start
type: phase
phase_status: pending
wp: wp06-capture-captureintake-staging
goal: Staging observations are atomically appended to staging/<agent>/<session>.jsonl with fsync; backpressure drops new observations (+ logs + queues emergency dream) when cap is exceeded (500MB or 10k files); capture-fallback buffers up to 10MB/7d when engramd is unreachable and is drained + re-filtered on next daemon start.
verify: "npm test tests/integration/staging — 100 concurrent appends to the same session file produce a valid JSONL with 100 lines (no torn writes); a simulated cap-exceeded state drops the next observation and logs a backpressure event; after a daemon restart the fallback file is drained and each entry passes through the privacy filter."
---
<!-- Template: WP-folder PHASE v2 (frontmatter-first) -->

# Phase 2: Staging JSONL + backpressure + capture-fallback + drain

**Goal:** Filtered observations land in `staging/<agent-id>/<session-id>.jsonl`
(§3.3 store layout). Each append is atomic: write to a `.tmp` file, `fsync`,
`rename` — consistent with the memory-file write protocol (§7.4). The file is
append-only; the dreaming worker reads it via a watermark cursor (§5.2,
`staging_watermark`).

**Backpressure** (§6.1, failure-safety §4): staging cap defaults to 500MB total
/ 10 000 files. When exceeded: drop the incoming observation, log a `capture`
AppLog event with `{backpressure: true}`, and enqueue an emergency dream job
(if a dreaming-memory exists for this scope).

**capture-fallback** (§6.1, §3.3): when `engramd` is unreachable the capture
hook writes to `.engram/capture-fallback/` (10MB max / 7-day TTL). On daemon
start (§9.9 step 6) the fallback directory is drained: each entry is passed
through the full privacy filter before appending to staging (S-09 boundary —
fallback entries are not pre-trusted).

**Verify:** `npm test tests/integration/staging` — 100 concurrent appends
produce valid JSONL (no torn lines); cap-exceeded → drop + log; restart drains
fallback + re-filters.

## Steps

| Step | File | State |
|------|------|-------|
| `Staging.append(agentId, sessionId, entry)` — `.tmp`→fsync→rename; creates `staging/<agent>/<session>.jsonl` | `src/core/staging.ts` | TODO |
| Staging size/count check before append; backpressure: drop + AppLog + emergency dream enqueue | `src/core/staging.ts` | TODO |
| `Staging.drain(fallbackDir)` — reads `.engram/capture-fallback/`, re-filters each entry via `PrivacyFilter`, appends to staging; skips entries older than 7d | `src/core/staging.ts` | TODO |
| Fallback write path: `Staging.writeFallback(entry)` — used by hooks when engramd is unreachable; enforces 10MB cap | `src/core/staging.ts` | TODO |
| Wire `drain()` into daemon startup sequence (§9.9 step 6) | `src/core/startup.ts` | TODO |
| Integration tests: concurrent appends, cap-exceeded, fallback drain with re-filter | `tests/integration/staging.test.ts` | TODO |

## Notes

The JSONL line format matches the `RawObservation` type produced by
`CapturePlugin.normalise()` (§2.3), with an additional `mac` field for S-02
attestation and `captured_at` ISO timestamp. The dreaming worker's
`staging_watermark` records the byte offset read up to — not a line count — so
the append-only JSONL must never be truncated mid-session (truncation happens
only on `MERGED`, §5.4). The fallback path (`capture-fallback/`) is per-observation
files (one JSON per file) not a single JSONL, to allow partial drain without
file-level atomicity issues.
