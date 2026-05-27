# ADR-0005: KbPlugin as the fifth seam — KB type is a plugin, KB instance is a registry row

- **Status:** Accepted
- **Date:** 2026-05-27
- **Related:** SPEC v2.2 §15.1, §15.2; ADR-0001 (microkernel), ADR-0004 (plugins are rebuildable derived indexes)

## Context

SPEC v2.1's microkernel has four plugin seams: Retrieval, Graph, LLM, Capture.
The store itself is fixed: one Markdown root under `~/.engram/` (global) or
`<repo>/.engram/` (project), with the four type folders inside. Every memory
goes there, regardless of where it came from.

The v2.2 extension introduces multiple knowledge bases of different *shapes*:

- **project memories** — engram's existing markdown store (current default).
- **llm-wiki** — wiki pages generated from session transcripts (Pratiyush/llm-wiki layout: `raw/sessions/`, `wiki/sources/`, `wiki/entities/`, `wiki/concepts/`, …).
- **obsidian-vault** — a user's existing Obsidian vault, including its `.obsidian/` workspace and Dataview metadata.
- **raw-sources** — a read-mostly pool of ingest material (PDFs, blog dumps, transcripts) that is consumed by the worker but not directly recalled.
- **agent-self** — a per-agent scratch KB owning Contextual + early-Episodic memory.

These shapes differ on: directory layout, frontmatter dialect, write policy
(read-write vs read-only vs append-only), ingest pipeline, and which lifecycle
jobs make sense (a `raw-sources` KB has no recall rollup; an `obsidian-vault`
has Dataview-aware retrieval).

A naive "add 5 hardcoded KB types to core" would bloat the kernel, couple it to
Obsidian's quirks, and violate Simplicity-First. A federation of independent
engramd instances doubles deployment cost and breaks dreaming (D-3 deferral).

## Decision

Add **`KbPlugin`** as a fifth plugin seam.

- A **KB type** is a `KbPlugin`. The kernel ships five built-in types
  (`markdown-store`, `llm-wiki`, `obsidian-vault`, `raw-sources`, `agent-self`),
  each implementing the same lifecycle contract. New KB types are added by
  dropping in a new plugin.
- A **KB instance** is a row in the registry — `{id, name, type, root, config}`.
  Multiple instances of the same type are allowed (e.g., two `obsidian-vault`
  instances pointing at two different vaults).
- The kernel owns: the registry, the routing layer (`kb.list/register/connect/
  disconnect/route`), the per-KB QMD index and graphify graph (provisioned on
  register, rebuilt on demand), the per-KB job schedule.
- The plugin owns: directory layout contract, frontmatter dialect, ingest
  pipeline shape, default lifecycle job set.
- Security-critical logic — privacy filter, access control, safe/gated
  classification — stays in core. A KbPlugin cannot turn it off.

The `KbPlugin` contract is small, mirroring the existing seams:

```ts
interface KbPlugin {
  manifest: { id: string; version: string; capabilities: KbCapability[] };
  init(ctx: KbContext): Promise<void>;
  validate(root: string): Promise<ValidationResult>;   // is this a valid KB of this type?
  layout(): KbLayout;                                  // dir layout + frontmatter dialect
  ingest?(input: IngestInput): Promise<IngestResult>;  // type-specific write path
  lifecycleJobs(): JobSpec[];                          // default daily/weekly schedule
}
```

Recall stays unified: the scoring engine fuses per-KB results behind the
existing `(I × Relevance × Recency) × m_v` formula. Cross-KB expansion is a
bridges-graph concern (ADR-0007), not a scoring-engine concern.

## Consequences

- The kernel grows by one contract (~150 lines + a registry). No new fixed-core
  modules.
- KB shape changes (e.g., Obsidian Dataview integration) are plugin work, not
  kernel work.
- "Files are truth" invariant holds — each KB instance has its own Markdown
  root; QMD index and graphify graph remain derived and rebuildable.
- Auto-discovery (ADR-0008) becomes a small kernel scanner that asks each
  registered KbPlugin's `validate()` whether a candidate directory belongs to
  it.

## Alternatives considered

- **Hardcode 5 KB types in core.** Smaller seam, faster v1.2. Rejected:
  violates Simplicity-First once Obsidian quirks creep in; new KB types would
  force core edits.
- **KBs as folders inside the single existing store.** Simplest; matches today's
  layout. Rejected: loses per-KB retrieval config, per-KB write policy, and the
  ability to point at an existing external vault.
- **Federation of independent engramd instances per KB.** Cleanest isolation.
  Rejected: doubles deployment cost; cross-KB consolidation gets harder; D-3
  cross-agent deferral already covers the federation case.
