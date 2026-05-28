# ADR-0012: PDF/book ingest pipeline via pdfplumber and per-book KB lifecycle

- **Status:** Accepted
- **Date:** 2026-05-28
- **Related:** SPEC v2.3 §4 (info flow), §6 (MCP contract), §11 (tech decisions),
  §12.3 SC-28; ADR-0002 (Python seam — pdfplumber reuses the graphify subprocess
  pattern), ADR-0005 (KbPlugin), ADR-0006 (per-KB lifecycle), ADR-0011
  (YouTube — sibling ingest pipeline, shared chunking approach);
  WP18 (milestone v1.3-ingest-formats-and-dashboard)
- **Supersedes:** none

## Context

PDFs and technical books are a primary user-facing ingest format: lecture notes,
papers, O'Reilly books, personal reading. The v1.3 milestone makes them a
first-class ingest source.

Three design questions drive this ADR:

**Why `pdfplumber` and not a TypeScript-native library?**  
`pdfplumber` is best-of-breed for layout-preserving text extraction: it exposes
per-character bounding boxes, detects tables and multi-column layouts, and
returns page coordinates that feed provenance metadata. `pdf.js` (the obvious TS
candidate) is a renderer, not an extractor — its text-layer heuristics lose
column order and table structure. Both `nashsu/llm_wiki` and `Pratiyush/llm-wiki`
independently chose `pdfplumber` for the same pipeline (PDF → chunked text →
LLM structuring), making it the established reference implementation for this
pattern.  
Running `pdfplumber` from TypeScript via `execFile` is zero new mechanism: the
graphify extractor (ADR-0002) established the subprocess / JSON-RPC-shaped IPC
pattern. `pdfplumber` piggybacks on that Python seam — it is not a third
language boundary, it is a second script in the existing one.

**Why is a book a `book` KbPlugin subtype with its own KB instance?**  
A single PDF is one coherent corpus with chapters, cross-references, and a
shared conceptual vocabulary. Merging multiple books into a single KB conflates
distinct corpora, forces global rebuild when one book changes, and prevents
per-book dream tuning. One book = one KB instance (per ADR-0006 lifecycle) gives
isolated rebuild, dream, and deduplication scope. The `book` type is a KbPlugin
subtype (ADR-0005) so the registry, recall fan-out, and cross-KB bridges remain
unmodified.

**Why not OCR in v1.3?**  
Scanned PDFs without a text layer require Tesseract or a deep-learning OCR model
(docTR). Both add large runtime dependencies and increase the job surface area.
The v1.3 happy path is text-layer PDFs — the dominant case for technical books
and papers. OCR is explicitly deferred to v2 and fails loudly at the boundary
so the user knows exactly why.

## Decision

PDF/book ingest uses `pdfplumber` (Python subprocess) for text and page-range
extraction, a chapter/heading heuristic chunker, and the existing dream worker
for typed-memory structuring. Books are a `book` KbPlugin subtype; one book is
one KB instance, reusing the ADR-0006 lifecycle without modification.

### Extraction

`pdfplumber` is invoked from the ingest worker via `execFile`, mirroring the
graphify subprocess shape (ADR-0002): the Python script receives a JSON payload
on stdin and writes a JSON result to stdout. Output: `{ pages: [{ n, text,
bbox }], metadata: { title?, author?, page_count } }`.

Failure branch: if `pdfplumber` returns no text (scanned PDF without text layer),
the job terminates with status `FAILED`, error kind `ingest.no_text_layer`, and
a user-facing suggestion to enable OCR in v2.

### Chunking heuristic

1. Detect chapter/heading boundaries via font-size + bold/style flags + heading
   patterns (`^Chapter \d`, `^\d+\.`, all-caps short lines).
2. Within each chapter, target ~500-token chunks aligned to paragraph
   boundaries (no mid-paragraph splits).
3. Preserve `pages: [start, end]` metadata per chunk; propagate `chapter` and
   `section` labels when detected.

Chapter-detection rate is logged as an AppLog counter so chunking quality is
observable and tuneable without code changes.

### Dream-worker structuring

Chunks flow through the standard dream pipeline unchanged: chunks → LLM →
typed memories. Memory types produced:

- `semantic` — facts, concepts, definitions.
- `procedural` — how-to passages, algorithms, recipes.
- `episodic` — **not produced**. R-4 invariant: episodic memories are
  session-derived; documents are not sessions.

### Provenance

Every memory carries:

```
sources:
  - kind: pdf
    path: <scope>/.engram/kb/book/<slug>/raw/<filename>.pdf
    pages: [start, end]
    chapter: <string | null>
    section: <string | null>
```

### Storage layout

```
<scope>/.engram/kb/book/<slug>/
  raw/<filename>.pdf          # git-ignored; immutable source
  memories/*.md               # typed memories, one file each
  manifest.json               # ADR-0006 lifecycle manifest
```

`<slug>` is derived from the filename (lowercase, hyphens, no extension).

### MCP verb

`ingest.pdf { path, kb? }` — `kb` defaults to `book/<slug-derived-from-path>`.
The verb is a subverb of the existing `ingest.*` namespace (SPEC §6); no new
MCP surface beyond the verb registration.

### AppLog events

| Event | Payload |
|---|---|
| `ingest.pdf.start` | `{ path, kb }` |
| `ingest.pdf.pages_extracted` | `{ count }` |
| `ingest.pdf.chapters_detected` | `{ count, detection_rate }` |
| `ingest.pdf.chunks_emitted` | `{ count }` |
| `ingest.pdf.complete` | `{ memories_written }` |
| `ingest.pdf.failed` | `{ kind, message }` |

## Consequences

**Positive**

- Ships PDF/book ingest for the common case (text-layer PDFs) with no new
  runtime mechanisms — Python seam and dream worker are already in the build.
- Per-book KB lifecycle isolates rebuild, dream, and deduplication; one broken
  PDF cannot force a global rebuild.
- Same chunking principles as the YouTube pipeline (ADR-0011) — consistent
  recall surface across ingest formats.
- Provenance (file + page ranges) satisfies SC-28.

**Negative / mitigations**

- OCR is explicitly out of scope — scanned PDFs fail loud with a clear error
  kind; tradeoff is accepted to avoid scope creep in v1.3.
- Chunking is heuristic — some books will chunk poorly. Mitigated by the
  `ingest.pdf.chapters_detected` counter, which makes detection quality
  observable and enables tuning without a code change.
- Per-book KB instances multiply registry entries. Mitigated by the KB registry
  introduced in WP13 (v1.2 milestone).

## Alternatives considered

- **Single shared "documents" KB for all PDFs** — rejected: one bad PDF forces
  a global rebuild; no per-book dream or deduplication isolation.
- **pdf.js (TypeScript)** — rejected: layout extraction materially worse than
  `pdfplumber`; same engineering cost, lower output quality.
- **Marker (ML-based PDF → Markdown)** — rejected: large model dependency; adds
  infra complexity the v1.3 happy path does not need.
- **Full OCR pipeline (Tesseract / docTR) in v1.3** — rejected: scope cut;
  deferred to v2 with an explicit failure path that communicates the gap.
- **One chapter = one KB instance** — rejected: breaks the "one book = one
  coherent corpus" abstraction; cross-chapter recall and deduplication would
  require cross-KB bridges for what is a single document.
