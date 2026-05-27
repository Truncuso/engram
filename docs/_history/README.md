---
title: "docs/_history — superseded engram design artifacts"
status: archived
note: These files are kept for audit/traceability only. They are NOT authoritative. The current spec is `docs/engram-SPEC.md` v2.1+.
---

# docs/_history — superseded engram design artifacts

This folder preserves engram's pre-v2.1 design audit trail. **Nothing here is authoritative.** The current authoritative spec is `docs/engram-SPEC.md`.

| Folder | Contents | Superseded by |
|---|---|---|
| `spec-v2/SPEC.md` | SPEC v2 (2026-05-22) — Round 1+2 merge | `docs/engram-SPEC.md` v2.1+ |
| `round-1-2/architecture-review.md` | Round-1 architecture review | folded → SPEC v2.1 §2, §3, §5, §11 |
| `round-1-2/failure-safety-review.md` | Round-1 failure/safety review | folded → SPEC v2.1 §8, §9 |
| `round-1-2/observability-review.md` | Round-1 observability review | folded → SPEC v2.1 §10 |
| `round-1-2/security-review.md` | Round-1 security review | folded → SPEC v2.1 §8 (threat model + 6 CRITICAL mitigations) |
| `round-1-2/SYNTHESIS.md` | Round-1/2 synthesis with D-1…D-7, R-1…R-5 author decisions | landed in SPEC v2.1 — see crosswalk below |

## Finding → SPEC v2.1 crosswalk

This table is the canonical record of where each Round-1/2 finding landed in the current spec. If you are tempted to read a `_history/` file to answer a design question, read the v2.1 section in the right column instead — it is current.

### Author decisions (D-1 … D-7)

| ID | Decision | Landed in |
|---|---|---|
| D-1 | Adopt the 4 architecture contracts (data-flow, plugin, MCP-16-verb, job-queue) | SPEC v2.1 §2 (architecture), §6 (plugin contract), §7 (MCP 16 verbs), §5.4 (job queue) |
| D-2 | Adopt failure + observability sections wholesale, X-1/X-2 reconciled | SPEC v2.1 §9 (failure), §10 (observability) |
| D-3 | Cross-agent dreaming → v2 (per-agent only in v1, keep schema field) | SPEC v2.1 §5 (dreaming scope), §12.1 (v1 scope) |
| D-4 | App-log = SQLite-only for v1; hash-chained JSONL deferred | SPEC v2.1 §10.2 (AppLog) |
| D-5 | Health = CLI + unix socket for v1 (no TCP port). **Note:** R-3 later reversed transport to Streamable HTTP on 127.0.0.1 + bearer; the unix-socket plan applies to health/metrics only. | SPEC v2.1 §10.3, §10.6 |
| D-6 | Threat model + 6 CRITICAL mitigations are spec requirements | SPEC v2.1 §8.3 (CRITICAL mitigations), §8.4 (HIGH) |
| D-7 | Trim v1: 2 relation kinds, `confidence` stored-not-consumed. **Note:** R-1 later reversed `confidence` to a v1 retrieval gate. | SPEC v2.1 §3 (relations), §3.6 (scoring uses confidence) |

### Round-2 reversals (R-1 … R-5)

| ID | Reversal | Landed in |
|---|---|---|
| R-1 | `confidence` is a v1 multiplicative gate (not stored-not-consumed). `verification_state` enum + `engram memory confirm <id>`. | SPEC v2.1 §3.6 (scoring formula), §7 (MCP `memory.confirm`) |
| R-2 | Per-type decay rates (not one formula across all memory types) | SPEC v2.1 §3.6 (per-type decay constants table) |
| R-3 | Transport = Streamable HTTP on `127.0.0.1` + bearer token (not stdio/unix). | SPEC v2.1 §7.1 (MCP transport), §8 (X-1 mitigation) |
| R-4 | Episodic immutable during dreaming; `derived_from` backlinks mandatory; counterfactual gate for procedural promotion. | SPEC v2.1 §3 (memory types), §5.2 (dreaming), ADR-0001 |
| R-5 | Active-pool floor `min(100, 20% of total)` on top of 5% per-run rate limit. | SPEC v2.1 §9.5 (forgetting safety rails) |

### Open-question resolutions (OQ-A … OQ-M)

| ID | Topic | Resolution | Landed in |
|---|---|---|---|
| OQ-A | Plugin host (TS-only vs FFI) | TS in-process or subprocess; no FFI | SPEC v2.1 §6, ADR-0002, ADR-0003 |
| OQ-B | LLM substrate | Vercel AI SDK behind `LlmPlugin`; raw `@anthropic-ai/sdk` for cache_control | ADR-0003 |
| OQ-D | Store paths | `~/.engram/` global, `<repo>/.engram/` project | SPEC v2.1 §3.2 |
| OQ-E | Agent identity | per-agent bearer token (SO_PEERCRED dropped by R-3) | SPEC v2.1 §7.1, §8 (S-03) |
| OQ-F | QMD index scope | indexes `memories/` only; worker reads staging directly | SPEC v2.1 §6.1 |
| OQ-G | Raw sources storage | git-ignored default; LFS opt-in | SPEC v2.1 §3.2 |
| OQ-H | Language | TypeScript / Node ≥22, ESM, strict | ADR-0002 |
| OQ-I | Job queue mechanism | SQLite `jobs` table | SPEC v2.1 §5.4, §10.2 |
| OQ-J | Dreaming worker — Agent SDK vs raw API | Vercel AI SDK + cache_control via `@anthropic-ai/sdk` | ADR-0003 |
| OQ-K | App-log format | SQLite (see D-4) | SPEC v2.1 §10.2 |
| OQ-L | AppLog hash-chaining | not in v1 (deferred) | SPEC v2.1 §10.2 |
| OQ-M | Health surface | unix socket for v1 (see D-5) | SPEC v2.1 §10.3 |

## Why this folder exists

The Round-1/2 reviews introduced specific, traceable corrections (D-1…D-7, R-1…R-5) and answered specific open questions (OQ-A…OQ-M). v2.1 absorbs all of them; the table above is the contract that says so. Keeping the source documents — not just the git log — makes the audit trail browsable without `git show`.

Round-3 (the v2 → v2.1 consistency review) is documented inline in `docs/engram-SPEC.md` §14, not here.
