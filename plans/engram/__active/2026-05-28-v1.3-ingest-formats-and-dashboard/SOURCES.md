---
plan: 2026-05-28-v1.3-ingest-formats-and-dashboard
created: 2026-05-28
updated: 2026-05-28
---

# Sources — v1.3

## Internal sources (this repo)

| ID | Path | Purpose |
|----|------|---------|
| SRC-SPEC-v2.3 | `docs/engram-SPEC.md` (v2.3, 2026-05-28) | Authoritative design. SC-27 … SC-35 derive from §12.3. |
| SRC-ADR-0009 | `docs/adr/0009-adr-as-first-class-memory-type.md` | Implements ADR-as-memory via KbPlugin. WP21 source-of-decision. |
| SRC-ADR-0010 | `docs/adr/0010-dashboard-v1-read-only-react-sigma.md` | Implements read-only dashboard. WP22 source-of-decision. |
| SRC-ADR-0011 | `docs/adr/0011-youtube-transcript-extraction-pipeline.md` | Implements YouTube ingest. WP19 source-of-decision. |
| SRC-ADR-0012 | `docs/adr/0012-pdf-book-ingest-pipeline.md` | Implements PDF/book ingest. WP18 source-of-decision. |
| SRC-HISTORY-memory-overhaul | `docs/_history/memory-overhaul/ARCHIVE.md` | Predecessor planning trail (unified-memory + llm-wiki-architecture). Context, not action. |

## External sources (cited in SPEC §11 / ADRs / README)

| Source | Role | Cited in |
|--------|------|----------|
| `rohitg00/agentmemory` | Pattern study (hooks, KV+vector, sentinels, signals, checkpoints) — NOT a code dependency. | SPEC §11.2, README Acknowledgements, `docs/research/agentic-memory-survey-2026-05-27.md` |
| `nashsu/llm_wiki` | Pattern study (compile-once wiki, LanceDB, sigma.js graph viz, format-agnostic ingest, Chrome ext capture) — user-cited; cross-referenced with Pratiyush/llm-wiki. | README Acknowledgements (v2.3 addition), `docs/research/agentic-memory-survey-2026-05-27.md` |
| `Pratiyush/llm-wiki` | Pattern study (same family as nashsu/llm_wiki; engram's prior attribution). | README Acknowledgements, survey doc |
| Hermes article ("I Rebuilt Hermes in Claude Code") | Framing inspiration — engram = the memory pillar of an "identity + memory + self-learning loop" agentic OS. | SPEC §1.1 (v2.3 framing paragraph), this milestone's OVERVIEW |
| User's Obsidian MOC `00_Memory_Moc_And_Notes.md` and siblings | User's prior beliefs about decoupled memory, dreaming, event-driven triggers — confirms architectural shape. | `docs/_history/memory-overhaul/` (and the user's vault, not duplicated here) |

## Library choices

| Library | Purpose | ADR / WP |
|---------|---------|----------|
| `pdfplumber` | PDF text + layout extraction (Python subprocess) | ADR-0012, WP18 |
| `youtube-transcript-api` | YouTube caption fetch (Python subprocess) | ADR-0011, WP19 |
| `yt-dlp` + `whisper` | Opt-in fallback for videos without captions (Python subprocess, config-gated) | ADR-0011, WP19 |
| `subtitle` (npm) or hand-rolled parser | .vtt / .srt parsing (pure TS) | WP20 |
| `react` (18) + `vite` + `typescript` | Dashboard SPA | ADR-0010, WP22 |
| `sigma` (sigma.js) | Knowledge-graph rendering in the dashboard | ADR-0010, WP22 |
| `@tanstack/react-query` | Dashboard data fetching | ADR-0010, WP22 |

All libraries are either MIT or BSD; no GPL/AGPL is introduced.
