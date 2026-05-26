---
name: phase-1-schemas-and-store
title: Schemas (Zod + JSON-schema) + store layout + atomic write
type: phase
phase_status: pending
wp: wp01-core-scaffold-store-schemas-applog-jobs-occ-coreservice-doctor-init
goal: Frontmatter/manifest/dream-output Zod schemas exist as the single source of truth, with JSON-schema export; the store layout (§3.3) is created with atomic file writes and ULID/slug identity.
verify: "npm test tests/unit/schemas — round-trips a valid memory and rejects an injected field; a written memory file matches §3.3 layout and parses back identically."
---
<!-- Template: WP-folder PHASE v2 (frontmatter-first) -->

# Phase 1: Schemas + store layout + atomic write

**Goal:** Frontmatter / manifest / dream-output **Zod schemas** are the single
source of truth (TS types derived; JSON-schema exported for the worker). The
store layout (§3.3) is created; file writes are atomic; identity is ULID + the
per-origin filename rule (§3.5).

**Verify:** `npm test tests/unit/schemas` round-trips a valid memory and rejects
an injected/unknown frontmatter field; a written file lands at the §3.3 path and
parses back byte-identically.

## Steps

| Step | File | State |
|------|------|-------|
| Frontmatter Zod schema (all §3.4 fields) + TS type | `src/schemas/frontmatter.ts` | TODO |
| Manifest + dream-output Zod schemas + JSON-schema export | `src/schemas/{manifest,dream-output}.ts` | TODO |
| Store paths/layout (raw, staging, memories/{type}, .engram) | `src/core/store/layout.ts` | TODO |
| Atomic write (`.tmp`→fsync→rename), `O_NOFOLLOW` | `src/core/store/write.ts` | TODO |
| ULID id + filename rule (slug vs id by origin) | `src/core/store/identity.ts` | TODO |
| YAML frontmatter parse/serialize (round-trip stable) | `src/core/store/frontmatter-io.ts` | TODO |
| Unit tests | `tests/unit/schemas/*.test.ts` | TODO |

## Notes

Schemas are imported everywhere; never re-declare these shapes (engram-typescript
rule). `dream-output.schema.json` is the artifact WP08/WP09 validate against
(C6 / S-05). `exactOptionalPropertyTypes` is on — model optional frontmatter
fields precisely.
