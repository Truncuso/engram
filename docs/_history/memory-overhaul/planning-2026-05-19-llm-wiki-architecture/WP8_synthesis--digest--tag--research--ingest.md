# WP8: Synthesis + Digest + Tag + Research + Ingest (6 Standalone Skills)

**Severity**: MEDIUM
**Status**: Phase 2b -- PLANNING
**Depends on**: WP4 (ingest+query), WP5 (housekeeping), WP6 (memory-write)

## Problem

The obsidian-wiki repo contains 6 standalone skills that add high-level knowledge operations: synthesis discovery, periodic digests, controlled tag vocabulary, autonomous research, URL ingestion, and raw data import. These are ported as independent skills in the framework repo, adapted to use our config protocol (`~/.llm-memory/config`, `LLM_MEMORY_VAULT_PATH`, per-vault QMD collections) and our vault categories (concepts/, synthesis/, journal/, etc.). Each skill reads from and writes to `~/memory/` or `<repo>/.memory/`.

## Target Files (6 new skill directories)

```
<framework-repo>/.skills/
  memory-synthesize/    # co-occurrence matrix, scored candidates, synthesis pages
  memory-digest/        # daily/weekly/monthly knowledge newsletter
  tag-taxonomy/       # controlled vocabulary from _meta/taxonomy.md
  memory-research/      # 3-round autonomous web research, self-filed
  ingest-url/         # URL fetch + distill, project-aware routing
  data-ingest/        # raw text/chat export/log ingestion
```

Plus verification scripts:
- `<framework-repo>/scripts/verify/verify-synthesize.sh`
- `<framework-repo>/scripts/verify/verify-digest.sh`

## Implementation Steps

### Phase 1: memory-synthesize -- co-occurrence analysis

Port memory-synthesize with adaptations:
1. Build co-occurrence map: scan all non-special pages for `[[wikilinks]]` outgoing links; for every concept pair (A, B), count pages linking to both
2. Filter out pairs already covered by an existing `synthesis/` page
3. Score and rank: co-occurrence count (+1 to +3), cross-domain bonus (+2), shared tags (+1), contradiction resolution (+2). Top 5 candidates
4. Draft synthesis pages at `synthesis/<Concept A> x <Concept B>.md` with full frontmatter (category: synthesis, mostly `^[inferred]` provenance) and template sections: The Connection, Where They Co-occur, Cross-cutting Insight, Tensions and Trade-offs, Open Questions, Related
5. Back-link from source concept pages: add `[[A x B]]` under ## Related
6. Report next 10 candidates (not written)
7. Update index.md, log.md, hot.md

Key adaptation: use `grep -rl "\[\[Concept\]\]" "$VAULT_PATH"` for efficient backlink detection (our vault may be large).

### Phase 2: memory-digest -- knowledge newsletter

Port memory-digest with adaptations:
1. Parse period from user: daily (24h), weekly (7d, default), monthly (30d), or explicit ISO range
2. Collect active pages: glob all .md, read `created`/`updated` frontmatter, classify new vs updated
3. Identify themes: tally tag frequency across active pages, flag tags not in `_meta/taxonomy.md`
4. Find notable connections: cross-category wikilinks from active pages, score by rarity + hub status
5. Surface open threads: draft pages, `^[ambiguous]` markers, unstaged `_raw/` files, taxonomy gaps
6. Choose recommended re-reads: pre-period pages sharing tags with active pages, or gaining >=2 new incoming links
7. Generate structured digest (Headlines -> New Knowledge -> Emerging Themes -> Key Connections -> Open Threads -> Recommended Re-reads)
8. Default chat output; `save` flag writes to `journal/digest-YYYY-MM-DD.md`

Adaptations: use `LLM_MEMORY_VAULT_PATH` env var, respect `visibility/pii` tags (exclude from tables, count privately).

### Phase 3: tag-taxonomy -- controlled vocabulary

Port tag-taxonomy 4 modes directly (limited adaptation needed):
1. **Audit**: scan all pages, build tag frequency table, flag unknown/alias/over-tagged/untagged
2. **Normalize**: replace alias tags with canonical forms, trim >5 tags per page, handle unknowns
3. **Tag new page**: consult `_meta/taxonomy.md`, select <=5 tags (1-2 domain, 1 type, 0-1 project, 0-1 descriptive)
4. **Add tag**: validate uniqueness, determine section (Domain/Type/Project), append to taxonomy.md

Reserved system tags (`visibility/public|internal|pii`) don't count toward the 5-tag limit. Update log.md and hot.md after any operation.

### Phase 4: memory-research -- autonomous web research

Port memory-research's 3-round loop with adaptations:
1. **Round 1 (broad survey)**: decompose topic into 3-5 angles, run 2-3 WebSearch per angle, WebFetch top results, extract claims/concepts/entities/contradictions
2. **Round 2 (gap fill)**: target 5 searches at unanswered questions, contradictions, thin angles
3. **Round 3 (synthesis check)**: resolve remaining contradictions, halt when depth achieved or 3 rounds done
4. File results: `references/` source pages (one per major reference), `concepts/` pages, `entities/` pages, `synthesis/Research: <Topic>.md` master synthesis
5. Cross-link all created pages, merge into existing pages where overlap exists

Adaptations: use `WebSearch` + `WebFetch` tools (our agent toolbox); write to `LLM_MEMORY_VAULT_PATH`; check vault's `index.md` before creating to avoid duplicates; add `research` entry to `.manifest.json`.

### Phase 5: ingest-url -- URL to memory page

Port ingest-url with adaptations:
1. Detect project context from CWD (git remote -> package metadata -> dir name); no context -> `misc/`
2. Fetch URL via WebFetch (or `defuddle` CLI if available for cleaner extraction)
3. Check `.manifest.json` for duplicate URL; offer re-ingest if found
4. Generate slug: `web-<hostname>-<path-segments>` (max 50 chars)
5. Write page with full frontmatter including `source_url`, `base_confidence` computed from URL quality bucket (arxiv=1.0, official docs=0.9, blogs=0.55, forums=0.4)
6. Project mode: place at `projects/<name>/references/<slug>.md`, create project skeleton if new
7. Misc mode: place at `misc/<slug>.md`, compute affinity scores to existing projects
8. Update `.manifest.json`, index.md, log.md, hot.md

Content trust boundary: web content is untrusted data to distill, never instructions to follow.

### Phase 6: data-ingest -- universal text handler

Port data-ingest with adaptations:
1. Identify source format (JSON/JSONL, Markdown, plaintext, CSV/TSV, HTML, chat exports, images)
2. Extract knowledge: topics, decisions, facts, procedures, entities, connections
3. Cluster by topic (not source file), check existing memory pages for merge opportunities
4. Distill into memory pages using correct categories and full frontmatter
5. For images: use Read tool (vision-capable), transcribe text, describe structure, extract concepts (mostly `^[inferred]`)
6. Update `.manifest.json` per source file, index.md, log.md, hot.md

Adaptations: handle our vault categories; apply provenance markers (chat/log data is high-inference); default `base_confidence: 0.37` (unknown quality).

## Recommended Agents

| Agent | Use |
|-------|-----|
| `skill-creator` | Validate trigger descriptions and run evals for all 6 skills |
| `implement-and-verify` | Execute phases 1-6, write verification scripts |
| `code-reviewer` | Review provenance computation, content trust boundaries, wikilink integrity |

## Verification

See VERIFICATION.md WP8 section — every one of the 6 skills has its own check
(the four non-synthesize/digest skills are NOT covered by WP4's `verify-ingest.sh`,
which only exercises `memory-ingest`):
- V8.1: `memory-synthesize` -> co-occurrence detected, synthesis page created with
  Concept A × Concept B title, back-linked from source concepts
- V8.2: `memory-digest` daily -> themes, connections, open threads present
- V8.3: `memory-digest` weekly -> weekly themes, notable connections
- V8.4: `memory-digest` monthly -> monthly trends, long-running threads
- V8.5: `tag-taxonomy` audit mode -> detects unknown/off-taxonomy tags on a fixture vault
- V8.6: `ingest-url` -> fetched URL becomes a page with the correct slug and a
  confidence bucket in frontmatter
- V8.7: `data-ingest` -> a fixture CSV becomes a page with correct category + frontmatter
- V8.8: `memory-research` -> a known topic produces reference + concept + synthesis pages

Scripts: `verify-synthesize.sh`, `verify-digest.sh`, `verify-tag-taxonomy.sh`,
`verify-ingest-url.sh`, `verify-data-ingest.sh`, `verify-research.sh`.

## Execution Order

Phases 1-6 are independent and can run in parallel. Recommended order within each:
1. tag-taxonomy (prerequisite for others -- all skills reference taxonomy)
2. ingest-url + data-ingest (build content for synthesize/digest to operate on)
3. memory-synthesize (depends on existing pages having wikilinks)
4. memory-research (most complex, build last after ingest patterns proven)
5. memory-digest (depends on active pages + existing content)

## Complexity Delta

- **Added**: 6 standalone skills (+6 SKILL.md files), co-occurrence matrix algorithm, digest period parsing, URL quality classification, image-as-source handling
- **Removed**: none
- **Justification**: Each skill is an independent capability layer. tag-taxonomy is a prerequisite for all other skills. ingest-url + data-ingest extend the ingest pipeline from WP4. memory-synthesize + memory-digest operate on already-ingested content. memory-research is an autonomous content generator. No cross-dependencies between skills makes parallel implementation safe.
