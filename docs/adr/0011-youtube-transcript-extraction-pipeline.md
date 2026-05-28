# ADR-0011: YouTube transcript extraction pipeline

- **Status:** Accepted
- **Date:** 2026-05-28
- **Related:** SPEC v2.3 §4 (info flow), §6 (MCP contract), §8 (threat model —
  egress), §11 (tech decisions), §12.3 SC-29/SC-34; ADR-0002 (Python seam
  invariant); ADR-0005 (KbPlugin shape); WP19 (milestone v1.3-ingest-formats-and-dashboard)

## Context

SPEC v2.3 adds YouTube transcript ingest as a v1.3 deliverable (SC-29, SC-34).
The pipeline must produce typed memories with per-segment timestamps and
provenance, callable via the `ingest.youtube` MCP verb, with an optional
media-download+transcription fallback gated by config.

**ADR-0002 invariant:** engram is a single TypeScript package with exactly one
Python seam — the graphify adapter (`src/plugins/graph/py/`), consumed via
subprocess over a JSON-RPC/stdio wire boundary. Any second Python seam requires
explicit justification.

**Why a second Python seam is warranted here:**

- `youtube-transcript-api` has no maintained TypeScript equivalent with feature
  parity (timestamped captions, language selection, auto-generated track
  support).
- The optional fallback tools (`yt-dlp`, `whisper`) are Python-native; no
  production-grade TS ports exist.
- Forcing TS would still shell out to the same Python binaries — the process
  boundary exists either way. Making the seam explicit and shaped identically to
  the graphify subprocess pattern (JSON-RPC/stdio, no shared types) is strictly
  better than an ad-hoc shell invocation buried in worker code.
- The seam is isolated to the ingest worker; it does not touch the MCP core,
  QMD, or the dreaming pipeline.

## Decision

**Primary path:** `youtube-transcript-api` (Python), invoked from the ingest
worker via `execFile` as a subprocess. The wire boundary mirrors ADR-0002's
graphify pattern: JSON-RPC over stdio, no shared types across the process
boundary. Returns transcript text and per-segment timestamps.

**Fallback path:** when caption fetch fails AND
`config.ingest.youtube.fallback_enabled === true`, the worker invokes `yt-dlp`
to download the audio stream, then `whisper` to transcribe. Config governs
model name, language hint, and compute device. Fallback is **off by default**
(see Consequences). Produces an identical output shape to the primary path.

**Output shape:** chunked typed memories, ~500 tokens per chunk (consistent
with PDF ingest, ADR-0012), with sources:

```typescript
sources: [{
  kind: "youtube",
  url: string,        // canonical watch URL
  video_id: string,
  segment_id: string, // sequential index within the chunk
  start_ts: number,   // seconds
  end_ts: number,
}]
```

**Chunking:** by transcript segment groups targeting ~500 tokens per memory.
Segment boundaries are preserved; a segment is never split mid-chunk.

**MCP verb:** `ingest.youtube { url, kb?, tags? }` — returns a job id
immediately; worker handles the pipeline asynchronously; status via
`dream.list_jobs`. Matches the async job pattern of all other ingest verbs.

**AppLog events:**

| Event | When |
|---|---|
| `ingest.youtube.start` | MCP verb received, job enqueued |
| `ingest.youtube.captions_found` | Primary path succeeded |
| `ingest.youtube.fallback_used` | Caption fetch failed, fallback running |
| `ingest.youtube.complete` | Memories written, job done |
| `ingest.youtube.failed` | Unrecoverable error |

**Config keys** (all under `ingest.youtube`):

| Key | Default | Notes |
|---|---|---|
| `fallback_enabled` | `false` | Must be `true` to allow yt-dlp + whisper |
| `whisper_model` | `"base"` | Any whisper model name |
| `whisper_language` | `null` | null = auto-detect |
| `whisper_device` | `"cpu"` | `"cpu"` or `"cuda"` |
| `proxy` | `null` | HTTP/HTTPS proxy for all YouTube fetches |

**Threat model (SPEC §8):** YouTube fetch is egress to a third-party host.
Mitigations:

- Every fetch is logged to AppLog (`ingest.youtube.start` with the URL).
- The `proxy` config key allows routing through a local proxy for auditing or
  network-policy compliance.
- Fallback is opt-in; media download (yt-dlp) is therefore never triggered
  without explicit user consent via config change.
- No credentials are stored; no YouTube auth is required for public videos.

**Python adapter location:** `src/ingest/youtube/py/` — a sibling to
`src/plugins/graph/py/` in structure. Install docs (README + `engram install`)
must list: `pip install youtube-transcript-api` (always); `pip install yt-dlp
openai-whisper` (if fallback enabled).

## Consequences

**Positive:**

- Ships YouTube ingest cheaply for the common case (videos with captions) — no
  media download, no transcription cost.
- Opt-in fallback avoids accidental bandwidth, disk, and CPU consumption.
- Reuses the ADR-0002 subprocess pattern — no new IPC mechanism introduced.
- Uniform async job shape (`dream.list_jobs`) across all ingest formats.

**Negative:**

- A second Python seam now exists. Mitigated by reusing the graphify subprocess
  shape identically; the seam is explicit, bounded, and auditable.
- Adds Python deps: `youtube-transcript-api` is always required; `yt-dlp` and
  `whisper` are required only when fallback is enabled. Install docs must be
  updated.
- Primary path depends on YouTube's caption surface, which is rate-limited and
  occasionally absent or removed. The fallback exists specifically for this
  failure mode.

## Alternatives considered

- **Pure TS via a JS scraper** — no maintained option with timestamped-caption
  feature parity; rejected.
- **Whisper-only (always download media)** — expensive default; egress + disk +
  CPU cost with no user opt-in; rejected.
- **Defer YouTube to v2** — explicitly a v1.3 user requirement (SC-29); rejected.
- **Third-party transcript API (AssemblyAI, Deepgram, etc.)** — adds external
  dependency, recurring cost, auth surface, and data-egress risk inconsistent
  with the local-first design goal; rejected.
