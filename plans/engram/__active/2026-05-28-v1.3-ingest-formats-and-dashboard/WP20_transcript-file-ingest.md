---
name: wp20-transcript-file-ingest
title: Standalone transcript file ingest (.vtt / .srt / .txt)
type: work-package
stage: spec
severity: MEDIUM
created: 2026-05-28
updated: 2026-05-28
plan: 2026-05-28-v1.3-ingest-formats-and-dashboard
tags: [ingest, transcript, vtt, srt, kbplugin]
relationships:
  - blocked_by: [[wp19-youtube-transcript-ingest]]
  - blocks: [[wp23-v1-3-e2e-acceptance-gate]]
sources: [SRC-SPEC-v2.3-SC-30]
---
<!-- Template: WP v2 (frontmatter-first) -->

# WP20: Standalone transcript file ingest (.vtt / .srt / .txt)

## Problem

SC-30 (SPEC v2.3 §17): a `.vtt` or `.srt` file must be ingested by
`ingest.transcript` and produce typed memories with per-cue timestamps in
`sources:`. Plain `.txt` transcripts (no timing markup) must also be accepted
and chunked by paragraph.

WP19 delivers the YouTube KbPlugin shape and chunker; WP20 reuses both for
local files. No new Python seam is needed — vtt/srt parsing is compact and
well-served by the `subtitle` npm package (TS-native).

## Deliverables

| # | Artifact | File | State |
|---|----------|------|-------|
| 1 | Transcript file parser | `src/plugins/kb/transcript/file-parser.ts` | TODO |
| 2 | KbPlugin instance (file variant) | `src/plugins/kb/transcript/file-plugin.ts` | TODO |
| 3 | MCP verb handler | `src/mcp/verbs/ingest-transcript.ts` | TODO |
| 4 | Fixture files (vtt, srt, txt) | `tests/fixtures/transcript/` | TODO |
| 5 | Unit tests — parser, all three formats | `tests/unit/transcript/file-parser.test.ts` | TODO |
| 6 | Integration test — full round-trip | `tests/integration/transcript/file-ingest.test.ts` | TODO |
| 7 | Hook trace test | `tests/integration/transcript/hook-trace.test.ts` | TODO |

## Design Notes

**Parser (`file-parser.ts`):**
- Detects format from file extension (`.vtt` → WebVTT, `.srt` → SubRip,
  `.txt` → plain).
- `.vtt` / `.srt`: uses the `subtitle` npm package to parse; emits per-cue
  records `{ start_ts: number, end_ts: number, text: string, cue_id: string }`.
  Timestamps are in milliseconds (millisecond granularity preserved).
- `.txt`: splits on double-newline (`\n\n`) paragraph boundaries; emits records
  with `start_ts`/`end_ts` absent and `cue_id` as sequential paragraph index.
- Malformed file (parse throws) → re-throws as `TranscriptParseError`; caller
  logs `ingest.transcript.parse_failed`.

**KbPlugin (file variant):**
- Reuses the KbPlugin interface from WP19; registers namespace as
  `transcript/file/<slug>` where `<slug>` is the filename stem, lowercased,
  spaces replaced with `-`.
- Chunker: groups cues into ~500-token chunks (same budget as WP19/PDF). For
  `.txt` each paragraph is its own chunk unless very short (< 50 tokens → merge
  with next).

**Output shape:**
```typescript
sources: [{
  kind:    "transcript",
  path:    string,          // absolute path of the ingested file
  format:  "vtt" | "srt" | "txt",
  cue_id?: string,          // absent for .txt paragraph chunks
  start_ts?: number,        // milliseconds; absent for .txt
  end_ts?:   number,        // milliseconds; absent for .txt
}]
```

**MCP verb:** `ingest.transcript { path, kb?, tags? }` — returns job id
immediately; worker handles pipeline async; status via `dream.list_jobs`.

**AppLog events:**

| Event | When |
|---|---|
| `ingest.transcript.start` | MCP verb received, job enqueued |
| `ingest.transcript.cues_parsed` | Parser completed; includes cue count |
| `ingest.transcript.chunks_emitted` | Chunker finished; includes chunk count |
| `ingest.transcript.complete` | Memories written, job done |
| `ingest.transcript.parse_failed` | Malformed file; includes parse error |
| `ingest.transcript.failed` | Unrecoverable non-parse error |

---

## Verification Matrix

| ID | Criterion | Setup | Expected | Method |
|----|-----------|-------|----------|--------|
| W20-1 | SC-30 .vtt round-trip | Ingest fixture `.vtt` with 3 cues | 3 memories; each has `start_ts`, `end_ts`, `cue_id` in `sources:` | Integration test |
| W20-2 | .srt round-trip — same memory shape | Ingest equivalent `.srt` fixture | Memory shape identical to W20-1; `format: "srt"` | Integration test |
| W20-3 | .txt round-trip — paragraph chunking | Ingest 3-paragraph `.txt` fixture | 3 memories; `start_ts`/`end_ts` absent; `format: "txt"` | Integration test |
| W20-4 | Malformed file fails loud | Pass truncated/invalid `.vtt` | AppLog contains `ingest.transcript.parse_failed`; job state `FAILED`; no memory written | Integration test |
| W20-5 | Timestamp millisecond precision | `.vtt` cue with `00:00:01.750 --> 00:00:03.250` | `start_ts: 1750`, `end_ts: 3250` in sources | Unit test (parser) |
| W20-6 | Hook trace fires | Run full ingest | `events.jsonl` contains start + complete entries for this job | Hook trace test |

---

## Quality Gates

| Gate | Command | Pass Criterion |
|------|---------|----------------|
| TypeScript compiles | `npm run build` | exit 0, no type errors |
| Unit tests | `npx vitest run tests/unit/transcript/` | All pass |
| Integration tests | `npx vitest run tests/integration/transcript/` | All pass (three-format) |
| Lint | `npm run lint` | exit 0 |

---

## Recommended Agents

| Phase | Agent | Rationale |
|-------|-------|-----------|
| impl | typescript-pro | Parser plumbing + KbPlugin wiring; pure TS, no subprocess |
| Review | code-reviewer | Verify sources shape matches WP19 contract; check .txt edge cases |
