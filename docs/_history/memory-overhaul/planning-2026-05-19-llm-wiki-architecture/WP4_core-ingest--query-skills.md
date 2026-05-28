# WP-4: Core Ingest + Query Skills

**Severity**: HIGH
**Status**: Pending
**Depends on**: WP1 (vault), WP2 (llm-memory schema), WP3 (memory-init scaffolds
the special files these skills maintain)

## Problem

The vault is structure without operations until it can ingest sources and answer
queries. This WP delivers the three core operational skills:

- **`memory-ingest`** — distill a source (file, directory, URL-saved doc, image)
  into one or more interconnected memory pages with full frontmatter and `[[wikilinks]]`.
- **`memory-query`** — answer a question against the compiled memory vault using the
  tiered retrieval protocol from `llm-memory` (WP-2).
- **`daily-update`** — the maintenance cycle: check source freshness, rebuild
  `index.md`, regenerate `hot.md`.

Source skills: `obsidian-wiki/.skills/{memory-ingest,memory-query,daily-update}/SKILL.md`
(390 / 188 / 198 lines). Each is **adapted**, not copied — same path-model and
retrieval-backend changes as WP-2 (scope discovery instead of `OBSIDIAN_VAULT_PATH`;
QMD as the index tier).

## Resolved design decision — OQ-10 (re-ingestion idempotency)

**Append-mode is the default.** When a `_raw/` source changes and is re-ingested:

- `memory-ingest` compares the source's current SHA-256 against `.manifest.json`.
- For each affected page, it also compares the *page's* stored hash against the
  page's current content.
  - **Page unchanged since last ingest** → regenerate the affected sections from
    the new source content (safe; no human edits to lose).
  - **Page human-edited since last ingest** (hash mismatch) → **merge**: add new
    source-derived content as new sections / bullet points; never overwrite
    human-edited prose. Mark merged-in sections so a later `memory-lint` can spot
    stale duplication.
- A `--full` flag opts into overwrite mode (regenerate the page wholesale) for
  the case where the human edit should be discarded. `--full` is never the
  autonomous-agent default.

`.manifest.json` therefore stores **two** hashes per entry: the source hash and
the last-ingest page hash. WP-3's `memory-init` creates `.manifest.json`; this WP
defines its schema.

## Target Files

- `<framework-repo>/.skills/memory-ingest/SKILL.md` (new)
- `<framework-repo>/.skills/memory-query/SKILL.md` (new)
- `<framework-repo>/.skills/daily-update/SKILL.md` (new)
- `<framework-repo>/scripts/verify/verify-ingest.sh` (new)
- `<framework-repo>/scripts/verify/verify-query.sh` (new)
- `<framework-repo>/scripts/verify/verify-daily-update.sh` (new)
- `<framework-repo>/templates/manifest.json` — two-hash schema (new)
- `<framework-repo>/scripts/graph/sync-graph.cjs` — Kuzu graph build/refresh from Markdown (new)
- `<framework-repo>/scripts/graph/graph-queries.cjs` — Cypher query library for structural traversal (new)

## Implementation Steps

### Step 1: `memory-ingest`

1. Scaffold via skill-creator. Triggers: "add this to the memory vault", "ingest this",
   "process these docs", plus raw-mode triggers ("promote my raw pages").
2. Adapt source → page distillation: read the source, classify into category
   (`concepts/`/`entities/`/…), write a page using the `templates/page.md`
   frontmatter from WP-2, set `provenance` (extracted for verbatim, inferred for
   interpretation, ambiguous when unsure), insert ≥ 2 `[[wikilinks]]` to related
   existing pages.
3. Implement the **two-hash manifest** logic and the append/overwrite rule above.
4. Image sources: read via the Read tool's vision support; skip with a logged
   message if the model lacks vision (per the obsidian-wiki source behavior).
5. After writing pages, update `index.md` and call `qmd-refresh.cjs --force` so
   new pages are searchable.
6. skill-creator review pass.

### Step 2: Kuzu Graph Integration

1. Install `kuzu` npm package in the framework repo (embedded graph DB, no server).
2. Create `scripts/graph/sync-graph.cjs`:
   - Input: vault path (global `~/memory/` or project `<repo>/.memory/`)
   - Parse all `.md` files → extract Page nodes from frontmatter (title, category, tags, lifecycle, scope)
   - Extract LINK edges from `[[wikilinks]]` in body + `relationships:` in frontmatter
   - Upsert into Kuzu DB at `<vault>/.kuzu/` using MERGE semantics (idempotent)
   - `.kuzu/` directory is git-ignored (derived index, rebuildable from Markdown)
3. Create `scripts/graph/graph-queries.cjs`:
   - Export 6 Cypher query functions: multiHop, reverseLookup, neighborhood, pathExists, bridgeDetection, centrality
   - Each function takes Kuzu connection + parameters, returns JSON results
4. Integration points:
   - SessionStart hook: call `sync-graph.cjs` after QMD refresh (same staleness gate)
   - SessionEnd hook: call `sync-graph.cjs --force` after writes (same pattern as QMD)
5. Verification: `verify-graph.cjs` — fixture vault → sync → 6 query types return correct results

### Step 3: `memory-query`

1. Scaffold via skill-creator. Triggers: "what do I know about X", "find
   everything related to Y", "search my memory". Include the index-only fast mode
   ("quick answer", "just scan").
2. Implement the tiered retrieval from `llm-memory` WP-2: Kuzu graph (structural patterns) → `qmd search` index pass →
   `qmd vector_search` semantic pass → section grep → full read. Fast mode stops
   after the index pass (summaries + frontmatter only).
3. Answers cite the source pages (`[[wikilink]]` + the page's `sources:`).
4. Works from any scope — resolve global vs project via the scope rule.
5. skill-creator review pass.

### Step 4: `daily-update`

1. Scaffold via skill-creator. Triggers: "/daily-update", "morning sync",
   "refresh the memory vault index"; also the WP-12 daily cron target.
2. Steps: walk `_raw/` and each source, recompute hashes, flag stale sources
   (source newer than its manifest entry); rebuild `index.md` from page
   frontmatter; regenerate `hot.md` (recent + active-focus pages, ≤ 500 words).
3. Does NOT itself ingest — it reports what is stale and leaves ingestion to
   `memory-ingest` / the WP-14 ingestion agent. (Keeps `daily-update` cheap and
   non-LLM-heavy so it is safe as a cron job.)
4. skill-creator review pass.

### Step 5: Verification scripts

Write the four `verify-*.sh` scripts (see Verification below). Script-first:
each is deterministic pass/fail, committed alongside the skill, run by `make verify`.

## Recommended Agents

- `skill-creator` — scaffold + review (2 passes each, 6 total per the plan).
- `code-reviewer` — final review of all three SKILL.md files + verify scripts.

## Verification

See VERIFICATION.md WP-4 section:
- V4.1: `verify-ingest.sh` — a known test source → page with correct category,
  full frontmatter, ≥ 2 wikilinks; `.manifest.json` updated with both hashes.
- V4.2: idempotent re-ingest of an **unchanged** source → no spurious page churn.
- V4.3: re-ingest of a **changed** source whose page was **human-edited** →
  append-mode merge; the human-edited prose is preserved verbatim; new content
  added as new sections.
- V4.4: `--full` flag → overwrite confirmed (human edit discarded).
- V4.5: `verify-query.sh` — a known fact is retrievable via the tiered pipeline;
  fast mode returns an answer from summaries without reading page bodies.
- V4.6: `verify-daily-update.sh` — `hot.md` regenerated, ≤ 500 words; `index.md`
  reflects current pages; stale sources flagged but not auto-ingested.
- V4.7: image source skipped with a logged message on a non-vision model.
- V4.8: `verify-graph-sync.sh` — fixture vault synced to Kuzu; node + edge counts match.
- V4.9: multi-hop query returns correct path on seeded fixture.
- V4.10: reverse-lookup returns all inbound pages with correct rel_type.
- V4.11: neighborhood query returns pages within specified depth.
- V4.12: path existence returns true/false correctly.
- V4.13: bridge detection returns pages connecting two categories.
