---
name: 2026-05-28-v1.3-ingest-formats-and-dashboard
title: engram v1.3 — multi-format ingest + read-only dashboard
status: active
feature: engram
created: 2026-05-28
updated: 2026-05-28
tags: [engram, memory, ingest, pdf, youtube, transcript, adr, dashboard, sdd]
work_packages:
  - wp18-pdf-book-ingest-pipeline
  - wp19-youtube-transcript-ingest
  - wp20-transcript-file-ingest
  - wp21-adr-as-first-class-memory-type
  - wp22-read-only-dashboard-v1
  - wp23-v1-3-e2e-acceptance-gate
relationships:
  - extends: [[2026-05-26-v1-build]]
  - parallel-to: [[2026-05-27-v1.2-multi-kb-and-skills]]
sources: [SRC-SPEC-v2.3, SRC-ADR-0009, SRC-ADR-0010, SRC-ADR-0011, SRC-ADR-0012, SRC-HISTORY-memory-overhaul]
---
<!-- Template: OVERVIEW v2 (frontmatter-first) -->

# engram v1.3 — multi-format ingest + read-only dashboard

## Executive Summary

v1.3 lands the user-facing surface that turns engram from a memory daemon into a usable memory system: **read-only dashboard** for browsing + semantic search + knowledge-graph view (ADR-0010), **multi-format ingest** for PDFs/books (ADR-0012), YouTube videos (ADR-0011), and standalone transcript files (.vtt/.srt/.txt), **ADRs as a first-class memory type** via the existing KbPlugin seam (ADR-0009), and the **explicit hook test gate** that closes a load-bearing v1 invariant gap (§6.1: never silent on filter drops).

The authoritative design is `docs/engram-SPEC.md` v2.3 (2026-05-28); this milestone operationalises the v2.2 → v2.3 amendment. **v1.3 is additive — no v1 or v1.2 invariants change, no existing WPs are restructured.** Two existing WPs (WP07, WP11) are extended with hook-test mandates and cross-references to this milestone's E2E gate (WP23).

**Three "it works" milestones for v1.3:**

- **M6** after WP18 — one ingest format end-to-end (PDF → typed memory → recall).
- **M7** after WP19 + WP20 + WP21 — full multi-format surface (PDF, YouTube, transcript file, ADR) all flow through the same dream worker and recall as a unified collection.
- **M8** after WP22 + WP23 — read-only dashboard browses the result + every SPEC v2.3 success criterion SC-27…SC-35 passes the automated acceptance gate.

## Active Work Packages

> Derived from WP frontmatter `stage:` by `update_plan.py --sync`. Do not edit the Stage column by hand.

| WP | Title | Severity | Stage | Impact |
|----|-------|----------|-------|--------|
| WP18 | PDF/book ingest pipeline (pdfplumber + dream-worker structuring) | HIGH | spec | **M6** first ingest format end-to-end |
| WP19 | YouTube transcript ingest (youtube-transcript-api + opt-in yt-dlp/whisper fallback) | HIGH | spec | YouTube videos as KB |
| WP20 | Standalone transcript file ingest (.vtt / .srt / .txt) | MEDIUM | spec | thin TS-only subset of WP19 |
| WP21 | ADR-as-first-class-memory-type (KbPlugin + per-project `adr/` + procedural recall) | HIGH | spec | per-project architectural memory |
| WP22 | Read-only dashboard v1 (React + sigma.js, loopback + bearer-gated) | HIGH | spec | **M8a** user-facing surface |
| WP23 | v1.3 E2E acceptance gate (SC-27 → SC-35 + hook test suite) | HIGH | spec | **M8** acceptance gate |

WP07 (existing, v1-build milestone) and WP11 (existing, v1-build milestone) are extended in place with hook-test mandates and v1.3 cross-references — see those WP files for the edits.

**SC ownership note.** The v2.3 success criteria map to WPs as: SC-27 → WP22, SC-28 → WP18, SC-29 → WP19, SC-30 → WP20, SC-31 → WP21, SC-32 → WP07 (hook chain) + WP23 (E2E), SC-33 → WP22, SC-34 → WP19. **SC-35** (`engram install --global/--project` idempotency) has no dedicated WP — its mechanism is owned by the existing WP00/WP01 (`engram init` + daemon registration) and WP22 (`engram dashboard login` bearer handshake); WP23 only *verifies* it via `sc35-install.spec.ts`. If WP00/WP01 do not fully cover the `install` verb surface, a follow-up WP24 should be opened (tracked in OPEN_QUESTIONS).

## Execution Strategy

Strict bottom-up. An edge `A → B` means B depends on A (B cannot reach `ready` until A is `verified`). All v1.3 WPs are gated on v1 M2 (WP05 verified) and on WP07 (ingest worker) being implemented.

```
v1 M2 (WP05 verified) + WP07 (ingest worker, hook-test-extended)
      │
      ├──► WP18 (PDF) ────────┐
      │                       │
      │      ┌──► WP19 (YouTube) ──┐
      │      │                     │
      │      │   ┌──► WP20 (transcript file) ──┐
      │      │   │                             │
      │      │   │   ┌──► WP21 (ADR, parallel)─┤
      │      │   │   │                         │
      │      │   │   │   ┌──► WP22 (dashboard) ─┤
      │      │   │   │   │                     │
      │      │   │   │   │                     ▼
      └──────┴───┴───┴───┴─────────────────► WP23 (E2E gate) ─► M8
                                  M7 = after WP19+WP20+WP21
                                  M6 = after WP18
                                  M8a = after WP22
```

- **WP18** is the foundation — Python adapter pattern (pdfplumber subprocess, mirrors ADR-0002 graphify shape) + chapter detector + chunker. Unlocks M6.
- **WP19** reuses WP18's chunker, adds the second Python seam (justified in ADR-0011: youtube-transcript-api has no TS equivalent; seam exists regardless).
- **WP20** is a thin TS-only subset — vtt/srt/txt parser, no new Python.
- **WP21** is **parallel-able** with WP18-20. It depends only on WP13 (KbPlugin seam from v1.2) being in place; it adds no ingest plumbing, just an ADR-aware KbPlugin instance.
- **WP22** needs WP05 (MCP) up and at least one ingest format usable. Pure frontend — React + sigma.js, loopback + bearer-gated.
- **WP23** is the acceptance gate. Every SC-27 → SC-35 has at least one automated E2E test plus the explicit hook test suite verifying SC-32 (capture → staging → dream → recall + filter-drop never silent).

## Relationship to v1.2 (parallel milestone)

v1.3 and v1.2 run in parallel. They share **WP13** (multi-KB registry + KbPlugin seam) as a common dependency — WP21 (ADR) and the per-format KbPlugin instances in WP18-20 all register against the WP13 registry. They do NOT share other state; v1.2's WP14 (per-KB lifecycle workers) and WP15 (cross-KB bridges) are layered separately and gated differently.

If v1.2 ships first: v1.3's KbPlugin registrations land cleanly.
If v1.3 ships first: v1.2's lifecycle workers wrap the already-registered KBs without retrofit.
Either order is supported.

## Constraints and Invariants

- **Files are truth** (SPEC §3): every ingest format writes Markdown + frontmatter to the store; derived indexes (QMD, graphify, AppLog) reflect that.
- **R-4 invariant** (SPEC §15.9, ADR-0009): ingest pipelines produce semantic + procedural memories. **Episodic memories are NOT produced from documents** — they are session-derived only.
- **4-type taxonomy preserved** (SPEC §3): ADRs are NOT a new top-level type. They are a KbPlugin instance (ADR-0009).
- **No new IPC mechanism** (ADR-0002): Python adapters reuse the graphify subprocess JSON-RPC shape. WP18 and WP19 add Python seams that follow this contract; no new mechanism is introduced.
- **No RRF, no fusion** (ADR-0007, SPEC §11.2): cross-KB ranking uses bridge scores (v1.2 WP15); v1.3 itself does not introduce any cross-KB ranking.
- **Episodics immutable during dreaming** (R-4): the dream worker structures ingest chunks into NEW typed memories, never mutates existing episodics.
- **Capture failures are never silent** (SPEC §6.1): if the privacy filter blocks an observation, AppLog records the drop with reason. WP23 hook test suite verifies this with failure injection.
- **Dashboard is read-only in v1** (ADR-0010): no editing, no ingest UI, no review queue UI. Those remain v2 scope per SPEC §1.2.

## Out of Scope (v1.3)

So implementers do not drift:

- ❌ OCR for scanned PDFs without text layer — deferred to v2 (ADR-0012)
- ❌ Whisper-only YouTube ingest as default — opt-in only, off by default (ADR-0011)
- ❌ Dashboard editing, ingest UI, dreaming visualisation, review queue UI — all v2 (ADR-0010)
- ❌ Cross-agent dreaming, hash-chained AppLog tamper-evidence, capture plugins beyond Claude Code — v2 carries (SPEC §12.1)
- ❌ Adding a new top-level memory type — taxonomy stays at 4 (ADR-0009, SPEC §3)
- ❌ Live cross-KB edges — bridges remain batch-derived (ADR-0007)
- ❌ Third-party transcript / OCR / book-summarisation APIs — local-first stays the default; integrations are v2

## Cross-References

- SPEC §0, §1, §3, §4, §6, §12.3 (SC-27 … SC-35) — `docs/engram-SPEC.md` (v2.3)
- ADR-0009 (ADR-as-memory-type) — `docs/adr/0009-*.md`
- ADR-0010 (dashboard) — `docs/adr/0010-*.md`
- ADR-0011 (YouTube) — `docs/adr/0011-*.md`
- ADR-0012 (PDF/book) — `docs/adr/0012-*.md`
- Historical context — `docs/_history/memory-overhaul/ARCHIVE.md`
- Parent v1 milestone — `plans/engram/__active/2026-05-26-v1-build/OVERVIEW.md`
- Parallel v1.2 milestone — `plans/engram/__active/2026-05-27-v1.2-multi-kb-and-skills/OVERVIEW.md`
