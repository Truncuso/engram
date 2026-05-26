# ADR-0006: CaptureIntake is fixed core — the privacy filter must not be a plugin

- **Status:** Accepted
- **Date:** 2026-05-26
- **Related:** SPEC §2.2, §6.1, §8.3 (S-01/S-09), §8.5, `.claude/rules/engram-architecture.md`

## Context

engram is a microkernel: a fixed core plus four plugin seams
(retrieval, graph, llm, capture). The capture seam is per-harness — the Claude
Code `CapturePlugin` installs hook scripts and normalises wire payloads. A
natural-looking but wrong design would put the **privacy filter** (which strips
secrets, API keys, PII before anything is persisted) inside that capture plugin,
alongside `install`/`normalise`.

That is dangerous. Capture hooks POST raw session observations to the daemon. If
the privacy filter lived in a swappable plugin, then a missing, misconfigured, or
buggy plugin would let raw observation data — including secrets — flow into
staging and from there into durable memory and git history. The filter is
**security-critical** and must be **fail-closed**: a filter error drops the
observation rather than passing it. Fail-closed semantics cannot be guaranteed by
a component the kernel does not own.

The same logic applies to the other security-critical decisions (access control,
safe/gated classification) — they are core, not plugin — and is already stated as
an architecture invariant (`engram-architecture.md`) and in SPEC §2.2. This ADR
records the decision for the capture path explicitly so it is not re-litigated.

## Decision

- **`CaptureIntake` is a fixed core component**, not a plugin. It owns the
  multi-layer privacy filter (regex / entropy / path-blocklist / optional LLM
  sweep), the fail-closed wrapper, and the `filter.audit_log` channel.
- **The capture *plugin* seam handles only harness adaptation:** `install` /
  `uninstall` (write hook scripts) and `normalise` (wire payload → typed
  `RawObservation`). It performs **no filtering**.
- All persisted-data security gates — privacy filter, access control, safe/gated
  classification — live in the kernel, never in a swappable plugin.
- The filter is **fail-closed**: any layer error (exception/timeout) drops the
  observation and logs at ERROR without payload content; the hook still sees a
  fast success (it must never stall the host session).

## Consequences

- A broken, missing, or hostile capture plugin **cannot** leak unfiltered data
  into the store — the kernel filters everything that reaches staging,
  independent of which harness produced it.
- Adding a new harness (a new editor/agent) means writing only an
  `install`/`normalise` plugin; the privacy guarantees come for free from the
  core and apply uniformly.
- The privacy filter is testable and auditable in one place (`src/core/`), and is
  the home of the S-01/S-09 mitigations and the capture audit trail (§10.5).
- One open behavioural question remains for the filter and is tracked separately
  (OQ-01: on a successful *match*, strip-the-secret-and-pass vs drop-the-whole
  -observation) — orthogonal to *where* the filter lives, which this ADR fixes.

## Alternatives considered

- **Privacy filter inside the CapturePlugin** — rejected: a plugin failure or
  misconfiguration becomes a data-leak path; fail-closed cannot be guaranteed for
  a component the kernel does not own; the guarantee would have to be re-proven
  per harness.
- **A dedicated "privacy" plugin seam** — rejected: adds a fifth seam for logic
  that must never be swapped or disabled, contradicting the point of a seam
  (swappability). Security-critical logic does not belong behind a swap boundary.
</content>
