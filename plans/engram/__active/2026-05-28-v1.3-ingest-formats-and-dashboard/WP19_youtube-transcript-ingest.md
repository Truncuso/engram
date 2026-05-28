---
name: wp19-youtube-transcript-ingest
title: YouTube transcript ingest (youtube-transcript-api primary, yt-dlp+whisper fallback)
type: work-package
stage: spec
severity: MEDIUM
created: 2026-05-28
updated: 2026-05-28
plan: 2026-05-28-v1.3-ingest-formats-and-dashboard
tags: [ingest, youtube, transcript, python-seam, kbplugin, egress]
relationships:
  - blocked_by: [[wp18-pdf-book-ingest-pipeline]]
  - blocks: [[wp23-v1-3-e2e-acceptance-gate]]
sources: [SRC-ADR-0011, SRC-SPEC-v2.3-SC-29, SRC-SPEC-v2.3-SC-34]
---
<!-- Template: WP v2 (frontmatter-first) -->

# WP19: YouTube transcript ingest (youtube-transcript-api primary, yt-dlp+whisper fallback)

## Why

SPEC SC-29 requires YouTube URLs to produce typed memories with timestamp
provenance. SC-34 mandates that any media-downloading fallback is gated by an
explicit opt-in config flag — no accidental bandwidth, disk, or CPU consumption.
ADR-0011 resolves the implementation: `youtube-transcript-api` (caption fetch,
no download) is the primary path; `yt-dlp` + `whisper` is the fallback, off by
default. Both paths are Python subprocess seams (ADR-0002 shape), orchestrated
by a TS client that follows the same async dream-worker pattern as PDF ingest
(WP18). This WP delivers the full pipeline from MCP verb to typed memories.

---

## Deliverables

| # | Artifact | Path | Status |
|---|----------|------|--------|
| 1 | Python adapter — primary | `src/plugins/kb/transcript/youtube-adapter.py` | TODO |
| 2 | Python adapter — fallback | `src/plugins/kb/transcript/yt-dlp-whisper-adapter.py` | TODO |
| 3 | TS orchestration client | `src/plugins/kb/transcript/youtube-client.ts` | TODO |
| 4 | KbPlugin instance | `src/plugins/kb/transcript/index.ts` | TODO |
| 5 | MCP verb handler | `src/mcp/verbs/ingest-youtube.ts` | TODO |
| 6 | Chunker extension | `src/plugins/kb/book/chunker.ts` (extend to accept transcript segments) | TODO |
| 7 | Config keys | `ingest.youtube.{fallback_enabled,whisper_model,proxy}` in config schema | TODO |
| 8 | AppLog events | `ingest.youtube.{start,captions_found,fallback_used,complete,failed}` | TODO |
| 9 | Unit tests | URL parsing, error mapping, primary→fallback gating | TODO |
| 10 | Integration tests | mock youtube-transcript-api → typed memories; fallback skipped in CI | TODO |
| 11 | Hook trace tests | same shape as WP18 | TODO |

---

## Approach

**Primary path** (`youtube-adapter.py`): invoke `youtube-transcript-api` with
the parsed `video_id`; emit JSON-RPC result over stdout — `{segments: [{text,
start, duration}]}`. No network I/O beyond the caption API; no media download.
Subprocess called via `execFile` with arg array (S-16).

**Fallback path** (`yt-dlp-whisper-adapter.py`): only invoked when primary
returns `no_captions` and `ingest.youtube.fallback_enabled = true`. Downloads
audio via `yt-dlp --extract-audio`, runs `whisper` with the configured model,
emits the same JSON-RPC segment shape. Conditional execution is enforced in the
TS client, not in the Python adapter, so the adapter itself is always
unconditional and testable in isolation.

**TS client** (`youtube-client.ts`): parses URL → `video_id`; spawns primary
adapter; on `no_captions` error checks config flag; conditionally spawns
fallback; maps all Python error kinds to `PluginError`. Logs every fetch to
AppLog with URL + resolved domain (egress threat model).

**KbPlugin** (`transcript/index.ts`): registers KB key `transcript/youtube/<video_id>`.
One KB per video for per-video lifecycle — same isolation pattern as book KBs
from WP18. Implements `PluginLifecycle`; delegates chunking to the shared
chunker extended with a `TranscriptSegment[]` overload.

**Chunker extension**: add `chunkTranscript(segments: TranscriptSegment[],
opts)` to `src/plugins/kb/book/chunker.ts`. Target ~500 tokens per chunk;
never split mid-segment. Reuses existing token-budget logic; no new chunking
strategy introduced.

**MCP verb** (`ingest-youtube.ts`): verb `ingest.youtube { url, kb?, tags? }`;
enqueues dream job; returns job id immediately. Status via `dream.list_jobs`.
Matches async pattern of `ingest.pdf` and all other v1.3 ingest verbs.

**Provenance**: every chunk carries
`sources: [{kind:"youtube", url, video_id, segment_id, start_ts, end_ts}]`.

**Config defaults**: `fallback_enabled: false`, `whisper_model: "base"`,
`proxy: null`.

**Egress policy**: every subprocess invocation logs URL + ip-domain to AppLog
before network contact; optional proxy threaded through both adapters via env
var `HTTPS_PROXY` (matches system convention).

---

## Verified Evidence

Each item must be demonstrably true before this WP is marked DONE.

| Claim | Verification command / test |
|-------|-----------------------------|
| TypeScript compiles | `npm run build` → exit 0, no type errors |
| Unit tests pass | `npm test -- --grep wp19` → all green |
| Integration tests pass | `npm run test:integration -- --grep wp19` (primary mock) |
| Fallback skipped in CI | CI env has `fallback_enabled=false`; fallback tests tagged `@manual` |
| Provenance shape correct | Snapshot test of `sources[]` against ADR-0011 fixture |
| AppLog events emitted | Integration log assertion — all five event kinds present in order |

---

## Quality Gates

- No `shell: true` or string concatenation in subprocess calls (S-16).
- Fallback adapter never invoked unless `ingest.youtube.fallback_enabled = true`.
- `no_captions` is a surfaced error when fallback is off, not a silent swallow.
- Every network fetch (primary or fallback) produces an AppLog egress entry.
- Chunker extension introduces no regression on existing PDF chunker unit tests.
- TypeScript strict mode; no `any` casts in new files.

---

## Verification Matrix

| ID | Scenario | Expected | Test kind |
|----|----------|----------|-----------|
| W19-1 | SC-29 round-trip: mock URL → typed memories | memories carry `sources[].start_ts / end_ts` | Integration (primary mock) |
| W19-2 | Fallback off by default: primary returns `no_captions` | surfaces `ingest.youtube.no_captions`; no fallback invoked | Unit |
| W19-3 | Fallback opt-in: `fallback_enabled=true`, primary fails | fallback adapter spawned; memories produced | Integration (manual / @manual tag) |
| W19-4 | SC-34 config gate: `fallback_enabled=false` with yt-dlp installed | no media download; `no_captions` error only | Unit + env assertion |
| W19-5 | AppLog egress: every fetch | AppLog contains URL + domain entry before subprocess returns | Integration log assertion |
| W19-6 | Memory shape parity with WP18 PDF path | dream worker output schema identical (minus `page` → `segment_id`) | Snapshot / schema test |
| W19-7 | Hook trace: same events as WP18 | `ingest.youtube.start` and `ingest.youtube.complete` in trace JSONL | Integration trace test |

---

## Risk & Mitigation

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| YouTube rate-limits caption API | Medium | High | Fallback path exists; exponential backoff in adapter; `no_captions` error surfaces cleanly |
| Whisper CPU cost on low-spec machines | High | Medium | Off by default; model configurable (`base` = fastest); user must opt in explicitly |
| Egress to YouTube / CDN | Certain | Low | AppLog entry for every fetch; proxy config threaded through both adapters |
| Caption removal post-ingest | Low | Low | Memories preserve cached transcript; URL field becomes informational; no re-fetch on read |
| Second Python seam adds maintenance surface | Medium | Low | Seam mirrors ADR-0002 shape exactly; bounded, auditable; no new IPC mechanism |

---

## Recommended Agents

| Phase | Agent | Rationale |
|-------|-------|-----------|
| impl | python-pro | Both adapters: subprocess shape, JSON-RPC stdout, error surface |
| impl | typescript-pro | TS client orchestration, KbPlugin, MCP verb handler |
| impl | tdd-guide | Primary/fallback gating has crisp I/O — strong TDD candidate |
| security gate | security-reviewer | Egress threat model: URL logging, proxy threading, subprocess arg arrays (S-16) |
| pre-merge | code-reviewer | Chunker extension regression risk; config-gate correctness |

---

## Review

| Reviewer | Verdict | Notes |
|----------|---------|-------|
| — | — | — |

---

## Follow-up Issues

| Issue | Severity | Status |
|-------|----------|--------|
| Install docs: add `youtube-transcript-api` (always) and `yt-dlp`+`whisper` (fallback) to setup guide | LOW | — |
| Consider per-video AppLog retention policy (egress log grows with usage) | LOW | — |
