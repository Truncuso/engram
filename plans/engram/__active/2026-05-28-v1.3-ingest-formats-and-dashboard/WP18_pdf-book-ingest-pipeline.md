---
name: wp18-pdf-book-ingest-pipeline
title: PDF/book ingest pipeline (pdfplumber + dream-worker structuring)
type: work-package
severity: MEDIUM
created: 2026-05-28
updated: 2026-05-28
plan: 2026-05-28-v1.3-ingest-formats-and-dashboard
tags: [ingest, pdf, book, python-seam, kbplugin]
relationships:
  blocked_by:
    - wp07-ingest-worker-graphify-graphplugin-ollama
    - wp13-multi-kb-registry-and-kbplugin-seam
  blocks:
    - wp22-read-only-dashboard-v1
    - wp23-v1-3-e2e-acceptance-gate
sources: [SRC-ADR-0012, SRC-SPEC-v2.3-SC-28]
status: TODO
---
<!-- Template: WP v2 (frontmatter-first) -->

# WP18: PDF/book ingest pipeline (pdfplumber + dream-worker structuring)

## Why

PDF/book ingestion is a first-class v1.3 scope item (SPEC v2.3 §4, SC-28).
ADR-0012 selects `pdfplumber` (Python subprocess, mirrors graphify seam from
ADR-0002) as the extractor; the existing dream-worker provides LLM structuring
without new mechanism. SC-28 recall-bridge invariant requires provenance
(file + page ranges) per chunk so expanded queries can surface PDF-sourced
memories. One book = one `book/<slug>` KbPlugin instance, reusing the ADR-0006
per-KB lifecycle (WP13) for isolated rebuild and deduplication.

## Deliverables

| # | Item | File | Status |
|---|------|------|--------|
| 1 | Python adapter: subprocess JSON-RPC, stdin→stdout, mirrors graphify shape | `src/plugins/kb/book/pdfplumber-adapter.py` | TODO |
| 2 | TS client: `execFile` wrapper, stdin/stdout JSON framing, error mapping | `src/plugins/kb/book/pdfplumber-client.ts` | TODO |
| 3 | Chapter-detection heuristic: font-size + bold flags + heading-pattern | `src/plugins/kb/book/chapter-detector.ts` | TODO |
| 4 | Chunker: ~500-token target, paragraph-aligned, page-range preserved | `src/plugins/kb/book/chunker.ts` | TODO |
| 5 | KbPlugin instance: implements KbPlugin contract (ADR-0005), registers as `book/<slug>` | `src/plugins/kb/book/index.ts` | TODO |
| 6 | MCP verb handler: `ingest.pdf { path, kb? }` → returns job id | `src/mcp/verbs/ingest-pdf.ts` | TODO |
| 7 | Dream-worker integration: chunks → staged → dreamer → `semantic` + `procedural` only (R-4: no episodic) | `src/worker/dream.ts` (extend) | TODO |
| 8 | Provenance: each memory `sources: [{kind:"pdf", path, pages:[start,end], chapter?, section?}]` | schema + worker | TODO |
| 9 | AppLog events: `ingest.pdf.{start,pages_extracted,chapters_detected,chunks_emitted,complete,failed}` | `src/worker/ingest-pdf.ts` | TODO |
| 10 | Failure path: no text layer → job `FAILED`, kind `ingest.no_text_layer`, user-facing msg | `src/worker/ingest-pdf.ts` | TODO |
| 11 | Tests: unit (chunker, chapter-detector, error mapping) + integration (fixture PDF → typed memories) | `tests/unit/book/`, `tests/integration/ingest-pdf.test.ts` | TODO |
| 12 | Tests: hook trace — AppLog events fire in order; fixture PDF in `tests/fixtures/` | `tests/fixtures/sample.pdf`, `tests/integration/` | TODO |

## Approach

1. **Python adapter** (`pdfplumber-adapter.py`): reads `{"path": "<abs>"}` from
   stdin; calls `pdfplumber.open(path)`; extracts `pages` array
   (`{n, text, bbox}`) + `metadata`; writes JSON to stdout; exits non-zero on
   any exception. No OCR, no fallback — scanned PDFs produce `text: ""`.

2. **TS client** (`pdfplumber-client.ts`): `execFile` with arg array (no
   `shell: true`); serialises request to stdin; parses stdout JSON; maps exit
   non-zero or empty-text to `PluginError` kinds (`ingest.no_text_layer` when
   all pages have empty text, `plugin-unavailable` when Python not found).

3. **Chapter detector** (`chapter-detector.ts`): scan page objects for
   font-size outliers and heading patterns (`^Chapter \d+`, `^\d+\.`, all-caps
   short lines ≤6 words); emit `ChapterBoundary[]` with page index + label;
   log detection count to AppLog counter for observability.

4. **Chunker** (`chunker.ts`): split text within chapter spans at paragraph
   boundaries; target ~500 tokens (character proxy: 2000 chars); never split
   mid-paragraph; annotate each chunk with `{pages: [start, end], chapter?,
   section?}`.

5. **KbPlugin** (`book/index.ts`): implement `PluginLifecycle` + KbPlugin
   contract (ADR-0005); `init` derives `<slug>` from filename; registers KB
   instance in the multi-KB registry (WP13); `rebuild` re-runs pdfplumber +
   chunker + dreamer for the book only.

6. **MCP verb** (`ingest-pdf.ts`): validate `path` under store root (path-jail,
   S-06); resolve `kb` to `book/<slug>` when absent; enqueue job; return job id.

7. **Dream integration**: hand `Chunk[]` to dreamer unchanged; assert output
   types in `{semantic, procedural}` only; write memories with full provenance
   frontmatter to `kb/book/<slug>/memories/`.

8. **Tests**: small fixture PDF (text-layer, ≥2 chapters) in
   `tests/fixtures/sample.pdf`; unit tests for chunker edge cases (single-page,
   no chapters) and chapter-detector (pattern matching); integration test:
   `ingest.pdf` → job MERGED → recall returns memory with correct page range.

## Verified Evidence

_(empty — to be filled during implementation)_

## Quality Gates

| Gate | Command | Expected |
|------|---------|---------|
| TypeScript compiles | `npm run build` | exit 0, no type errors |
| Unit tests pass | `npx vitest run tests/unit/book/` | all green |
| Integration test | `npx vitest run tests/integration/ingest-pdf.test.ts` | all green |
| Lint | `npm run lint` | no errors |
| Full PDF round-trip | integration test with `tests/fixtures/sample.pdf` | typed memories in `kb/book/<slug>/memories/`, page ranges present |

## Verification Matrix

| ID | Test | Expected Result | Method |
|----|------|----------------|--------|
| W18-1 | SC-28 round-trip: fixture PDF → `ingest.pdf` → dreamer → `recall` | Job state = MERGED; ≥1 typed memory returned by recall with `sources[0].kind = "pdf"` | Integration test |
| W18-2 | Page-range provenance: inspect memory frontmatter after ingest | Every memory has `sources[0].pages = [start, end]` with valid page numbers | Integration test assertion |
| W18-3 | Chapter detection rate ≥ 80% on fixture PDF with known chapter count | AppLog `chapters_detected.count` / known chapters ≥ 0.80 | Integration test + AppLog assertion |
| W18-4 | Scanned PDF (no text layer): supply fixture with empty text layers | Job status = FAILED, err kind = `ingest.no_text_layer`, user-facing message present | Unit test with mock adapter response |
| W18-5 | Hook chain: AppLog events fire in declared order | Events `start → pages_extracted → chapters_detected → chunks_emitted → complete` present in AppLog, in order | Integration test capturing AppLog stream |
| W18-6 | Per-book isolation: ingest same PDF twice | Second ingest triggers rebuild of `book/<slug>` only; no other KBs touched; no duplicate memories (dedup by content hash) | Integration test |
| W18-7 | R-4 invariant: no episodic memories emitted | After full ingest, no memory file in `kb/book/<slug>/memories/` has `type: episodic` | Integration test assertion on output files |

## Risk & Mitigation

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| pdfplumber missing from env | Low | High | `health` check at plugin init; clear install instructions in README; `plugin-unavailable` err kind |
| Poor chapter detection on unusual layouts | Medium | Medium | Detection rate logged via AppLog counter; tuneable patterns without code change; heuristic is best-effort |
| Large PDFs exhaust memory in Python adapter | Low | Medium | Stream pages in batches of 50; log page count at start; job-level timeout |
| Per-book KB registry bloat (many PDFs) | Low | Low | ADR-0006 lifecycle + WP13 registry already handles this; no mitigation needed beyond what WP13 provides |
| Path-jail bypass via symlink | Low | High | Resolve to real path before jail check (`fs.realpathSync`); reject if outside store root |

## Recommended Agents

| Role | Agent | Scope |
|------|-------|-------|
| Python adapter impl | python-pro | `pdfplumber-adapter.py`, subprocess framing, page extraction, error output |
| TS plumbing impl | typescript-pro | `pdfplumber-client.ts`, `chapter-detector.ts`, `chunker.ts`, `ingest-pdf.ts` verb handler |
| Final pass | code-reviewer | Path-jail logic, subprocess arg arrays (S-06/S-16), R-4 invariant, provenance schema |

## Review

| Reviewer | Verdict | Notes |
|----------|---------|-------|
| — | — | — |

---

## Follow-up Issues

| Issue | Severity | Status |
|-------|----------|--------|
| — | — | — |
