# ADR-0009: ADRs as a first-class engram memory type via KbPlugin

- **Status:** Accepted
- **Date:** 2026-05-28
- **Related:** ADR-0005 (KbPlugin seam), ADR-0006 (per-KB lifecycle),
  SPEC v2.3 §3, §15.1; WP21 (milestone v1.3-ingest-formats-and-dashboard)
- **Supersedes:** none

## Context

Users write Architecture Decision Records per project — typically under
`adr/`, `docs/adr/`, or `decisions/`. These files capture *why* choices were
made, rejected alternatives, and the status of each decision. When a project
is loaded into engram, that reasoning should be recallable alongside code
knowledge — "why did we reject JWT?" should surface ADR-0003, not require
the agent to grep the repo.

Two tempting but wrong approaches surface immediately:

**Why not a new top-level memory type `decision`?**  
SPEC §3.1 defines four types — `semantic`, `episodic`, `procedural`,
`contextual` — that cover the full spectrum: what happened, what is known,
how to do things, current state. Adding a fifth breaks the type-taxonomy
invariant (§3 is load-bearing: scoring, retrieval fans, the dream-job
pipeline, and AppLog all branch on these four). ADRs are naturally
*procedural* — they encode hard-won patterns ("always validate at system
boundaries"), rejected paths, and settled decisions. Storing them there
keeps the taxonomy closed.

**Why not a tag on ordinary procedural memories?**  
Tags are discovery hints, not schema contracts. ADRs have a fixed structure
(Decision/Context/Consequences/Status/Alternatives) that must be preserved
for reliable extraction. A free-form tag gives no enforcement and no
structured recall path.

**Why a KbPlugin?**  
ADR-0005's KB seam exists precisely for this: adding a new knowledge-base
*shape* without touching core. An ADR directory is a well-defined shape —
known file-naming convention, fixed frontmatter, read-only after acceptance,
per-project scope. Implementing it as a `KbPlugin` reuses the registration,
validation, lifecycle, and recall plumbing from ADR-0005 and ADR-0006 at
zero cost to the microkernel.

## Decision

ADRs are a first-class engram memory type, implemented as a dedicated
`KbPlugin` instance — not a new top-level memory type, not a special-case
in core.

### ADR schema (frontmatter contract)

Each ingested ADR is stored as a procedural memory with this additional
frontmatter:

```yaml
kind: adr                       # discriminator on the procedural track
adr_id: "ADR-0005"              # canonical ID from the filename
adr_status: accepted            # proposed | accepted | rejected | superseded
decision: "…"                   # one-liner decision string
supersedes: "ADR-0003"          # optional; null if none
related: ["mem_01HX…", "ADR-0006"]  # memory IDs or ADR IDs
```

The markdown body preserves the original Context / Decision / Consequences /
Alternatives sections. The KbPlugin sets `type: procedural`, `origin:
ingested`, and `confidence: 0.9` (accepted) / `0.5` (proposed).

### Storage

```
<repo>/.engram/kb/adr/          # KbPlugin working store
  NNNN-slug.md                  # one file per ADR; frontmatter + body
```

Memories are also written into the standard store under `memories/procedural/`
so they participate in global scoring and dreaming.

### Auto-detection

The plugin matches any of:

- `<repo>/adr/NNNN-*.md`
- `<repo>/docs/adr/NNNN-*.md`
- `<repo>/decisions/NNNN-*.md`

The pattern is configurable via the KB registry row (`adr_glob`). On match,
the KbPlugin validates frontmatter, rejects files that fail schema validation
(quarantine, not crash), and queues an ingest job via the standard dream-job
state machine (ADR-0006). Rebuild is per-file on change; no incremental
partial edits — the full file is re-ingested.

### Recall surface

ADR memories are returned on `recall(track: "procedural")` with a `kind:
"adr"` discriminator so callers can filter or render them distinctly.
Cross-links declared in `related:` are wired through the bridge layer
(ADR-0007) — `engram memory why <id>` traverses them.

## Consequences

**Positive**

- Every project with an `adr/` directory gets architecture-decision recall
  for free — no manual capture step.
- The §3 type taxonomy stays at four; no invariant break.
- Rejected alternatives and supersession chains are recallable, not just the
  current decision.
- Per-KB lifecycle (ADR-0006) handles rebuild on file change; no new
  scheduling machinery.

**Negative**

- The `KbPlugin` contract must support per-file frontmatter validation
  (`validate(file) → ok | quarantine`). This is a small extension to the
  interface defined in ADR-0005 (currently `validate(dir)`).
- ADR naming conventions vary across projects; the glob pattern must be
  configurable per KB registry row, adding one config knob.
- Frontmatter in existing project ADRs is often absent or inconsistent;
  the plugin must degrade gracefully (parse body-only, infer `adr_status`
  from heading text, flag as `confidence: 0.5`).

## Alternatives considered

- **New top-level memory type `decision`** — rejected: breaks the 4-type
  taxonomy invariant; §3 scoring, retrieval, and dream pipelines all branch
  on the closed type set.
- **Tag `kind=adr` on ordinary procedural memories** — rejected: loses
  schema enforcement; `adr_status`, `supersedes`, and `related` fields
  would be free-form strings with no validation path.
- **Inject ADRs at session start only (no recall index)** — rejected: defeats
  relevance-ranked recall; injects all ADRs regardless of query relevance;
  wastes context on unrelated decisions.
