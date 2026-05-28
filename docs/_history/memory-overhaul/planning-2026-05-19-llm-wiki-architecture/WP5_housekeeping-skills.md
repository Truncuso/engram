# WP-5: Housekeeping Skills

**Severity**: HIGH
**Status**: Pending
**Depends on**: WP1 (vault), WP2 (llm-memory schema), WP4 (pages to lint exist
once ingest works — WP-5 can be built in parallel with WP-4 but is verified after)

## Problem

An ingesting memory vault accumulates drift: orphan pages with no incoming links, broken
`[[wikilinks]]`, stale pages, undeclared contradictions, malformed frontmatter.
This WP delivers the three housekeeping skills that keep the graph healthy:

- **`memory-lint`** — audit the memory vault and report (or, with `--consolidate`, fix)
  structural issues. Report-only by default.
- **`memory-status`** — show memory vault state: what is ingested, what is pending, the
  source↔vault delta; plus an insights mode (hubs, bridges, orphan-adjacent pages).
- **`cross-linker`** — discover missing cross-references and **write** them into
  pages. The one write-heavy housekeeping skill.

Source skills: `obsidian-wiki/.skills/{memory-lint,memory-status,cross-linker}/SKILL.md`
(531 / 461 / 269 lines). Adapted, not copied — same path-model / QMD-backend
changes as WP-2 and WP-4.

## Target Files

- `<framework-repo>/.skills/memory-lint/SKILL.md` (new)
- `<framework-repo>/.skills/memory-status/SKILL.md` (new)
- `<framework-repo>/.skills/cross-linker/SKILL.md` (new)
- `<framework-repo>/scripts/verify/verify-lint.sh` (new)
- `<framework-repo>/scripts/verify/verify-status.sh` (new)
- `<framework-repo>/scripts/verify/verify-cross-link.sh` (new)

## Implementation Steps

### Step 1: `memory-lint`

1. Scaffold via skill-creator. Triggers: "check the memory vault for issues", "memory vault
   health check", "what needs fixing", "audit my notes".
2. Port the issue-type catalog from the source `memory-lint/SKILL.md`. The source
   defines the full set (orphaned pages, broken wikilinks, malformed frontmatter,
   stale content, contradictions, invalid lifecycle/confidence enums, invalid
   relationship `type:` values, tag-alias inconsistencies, …). **The porting step
   must enumerate every issue type from the source verbatim** — the plan's
   "13 issue types" count comes from that source; do not invent the list, copy it.
3. Two modes: **report-only** (default — never writes) and **`--consolidate`**
   (the "dream cycle": fixes broken links, adds orphan cross-refs, corrects
   lifecycle enums, demotes stale peripheral pages, normalizes tag aliases, adds
   contradiction callouts). `--consolidate` always does a dry-run preview and
   requires explicit user confirmation before any write.
4. Staleness is **computed at read time** (`is_stale = today − updated > 90d`),
   never stored. Stale + `lifecycle: verified` pages get a louder annotation.
5. `memory-lint --consolidate` is the WP-12 **weekly** cron target — so the
   dry-run-then-confirm gate must also have a non-interactive path (in
   automation: produce the report, apply only the safe mechanical fixes, log the
   rest for human review — never silently apply judgement-heavy fixes).
6. skill-creator review pass.

### Step 2: `memory-status`

1. Scaffold via skill-creator. Triggers: "what's the status", "show me the
   delta", "what changed since last ingest", "memory dashboard"; plus insights mode
   ("memory insights", "what's central", "show me the hubs").
2. Compute the source↔vault delta from `.manifest.json` (sources present but not
   ingested, sources changed since last ingest, pages with no source).
3. Report token footprint of the memory vault and of what a SessionStart load would inject.
4. Insights mode: analyze the link graph — top hubs (most incoming links),
   cross-category bridge pages, orphan-adjacent pages.
5. Read-only — `memory-status` never writes. skill-creator review pass.

### Step 3: `cross-linker`

1. Scaffold via skill-creator. Triggers: "link my pages", "find missing links",
   "connect my memory", "my memory vault feels disconnected"; also run after a large ingest.
2. Scan pages for unlinked mentions of other pages' titles/aliases; score
   affinity; insert `[[wikilinks]]` with the correct relationship `type:` from
   the `llm-memory` allowed set (`extends`, `implements`, `contradicts`,
   `derived_from`, `uses`, `replaces`, `related_to`).
3. **Write-heavy** — it modifies pages. Dry-run preview + confirmation before
   writes, same discipline as `memory-lint --consolidate`. After writing, call
   `qmd-refresh.cjs --force`.
4. skill-creator review pass.

### Step 4: Emergent Entity Detector

1. Create `detect-emergent-entities` mode in `cross-linker` skill (gated behind `--deep` flag).
2. After the existing unlinked-mention scan (Step 3), run a second LLM pass:
   - Extract named entities from each page body
   - Cross-reference against the page catalog (titles + aliases)
   - Flag terms appearing in ≥3 pages that lack a dedicated page file
3. Output: list of `[[suggested page titles]]` with source pages, ranked by occurrence count
4. Cost-gated: only runs with `--deep` flag or during weekly `memory-lint --consolidate` cron.
5. skill-creator review pass.

### Step 5: Verification scripts

Write the three `verify-*.sh` scripts. Each builds a small fixture vault with
known issues and asserts deterministic pass/fail.

## Recommended Agents

- `skill-creator` — scaffold + review (2 passes each, 6 total per the plan).
- `code-reviewer` — final review of all three SKILL.md files + verify scripts.

## Verification

See VERIFICATION.md WP-5 section:
- V5.1: `verify-lint.sh` — a fixture vault seeded with one instance of each
  issue type → `memory-lint` detects every type. The fixture and the detected-type
  list must both equal the source's enumerated catalog (no missing type).
- V5.2: `memory-lint` report-only mode writes nothing (assert vault byte-identical
  before/after).
- V5.3: `memory-lint --consolidate` dry-run lists fixes; applies only after
  confirmation; non-interactive path applies safe fixes only and logs the rest.
- V5.4: `verify-status.sh` — delta correctly computed against a fixture
  `.manifest.json`; insights mode surfaces the seeded hub page.
- V5.5: `verify-cross-link.sh` — a fixture with a known unlinked mention →
  `cross-linker` inserts the `[[wikilink]]` with a valid relationship type;
  dry-run gate respected; `qmd-refresh` called after write.
