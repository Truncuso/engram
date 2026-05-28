---
plan: 2026-05-28-v1.3-ingest-formats-and-dashboard
created: 2026-05-28
updated: 2026-05-28
---

# Open Questions — v1.3

No open questions block milestone start. Items below are noted for resolution **during** the milestone, not before.

## To resolve during the milestone

### OQ-V1.3-1 — chapter-detection heuristic strictness (WP18)
The pdfplumber chapter detector (font-size + heading-pattern) will not catch every PDF style. Question: should the chapter-detection rate be a hard gate (≥ 80% on a benchmark set, per Verification Matrix W18-3), or a metric we track without gating? Resolution path: ship the heuristic, instrument the metric, gate on it in WP23 acceptance only if it exceeds 80% on the fixture suite.

### OQ-V1.3-2 — YouTube fallback default for headless installs (WP19)
ADR-0011 makes the `yt-dlp + whisper` fallback opt-in. Question: for headless server installs where the user is unlikely to monitor bandwidth, should the fallback default be even stricter (e.g. require an explicit interactive confirmation)? Resolution: keep config-flag default off in v1.3; revisit if usage telemetry shows accidental enables.

### OQ-V1.3-3 — ADR matcher path precedence (WP21)
`<repo>/adr/`, `<repo>/docs/adr/`, `<repo>/decisions/` are all common conventions. When multiple are present in one repo, which wins? Resolution: matcher reads ALL and dedupes by `id` field in frontmatter; conflicting IDs → schema-failed event, neither is ingested. Confirm in WP21 implementation.

### OQ-V1.3-4 — dashboard port conflict handling (WP22)
ADR-0010 puts the dashboard on a separate port from MCP. Question: what happens when the chosen port is taken? Resolution: engramd selects the next free port in a small range (e.g. 7050…7060) and logs the actual port; `engram dashboard login` reads it from the daemon status endpoint. Confirm in WP22.

### OQ-V1.3-5 — degraded-mode last-known status freshness (WP22)
SC-33 mandates a "last-known status" cache. Question: how stale is acceptable? Resolution: dashboard refreshes a small status cache (`~/.engram/dashboard-cache.json`) every 60s while engramd is healthy; when down, it serves whatever is there with a "last refreshed N seconds ago" indicator. No TTL — staleness is shown, not hidden.

### OQ-V1.3-6 — SC-35 install-verb ownership (WP00/WP01 vs new WP24)
SC-35 requires `engram install --global` / `--project <path>` to be documented and idempotent, producing a working daemon registration. The mechanism overlaps WP00/WP01 (`engram init` + daemon registration) and WP22 (`engram dashboard login` bearer handshake), but no single WP owns the `install` verb surface end-to-end. Resolution path: audit WP00/WP01 coverage during v1.3 kickoff; if the `install` verb (as distinct from `init`) is not fully covered, open WP24 for install docs + idempotency. WP23 verifies SC-35 regardless via `sc35-install.spec.ts`.

## Closed / not applicable

- ADR-as-memory taxonomy question (4-type vs 5-type) — closed by ADR-0009 (KbPlugin, not new type).
- Dashboard editing scope — closed by ADR-0010 (read-only in v1).
- Python seam expansion — closed by ADR-0011 invoking ADR-0002 explicitly.
- OCR support — closed by ADR-0012 (deferred to v2).
