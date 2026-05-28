# WP-1: Directory Migration + Vault Setup

**Severity**: HIGH
**Status**: Pending
**Depends on**: WP0 (framework repo scaffold)
**Risk**: HIGH — moves a verified-working memory store. A botched run breaks the
Phase 1 system (37/37 green). This WP is structured around a backup-first,
rollback-always discipline.

## Problem

Phase 1's memory lives at `~/.claude/.memory/` (global). Phase 2 makes `~/memory/`
the canonical vault — accessible to non-Claude agents and openable directly in
Obsidian — and converts `~/.claude/.memory/` into a symlink. The vault must also
gain the full memory vault category structure (`concepts/`, `entities/`, …) and the
`_raw/` bridge. QMD collections must be re-registered against the new paths, and
the hook scripts must resolve memory through the new location.

The migration touches: the global memory tree, QMD's registered collections, the
SessionStart/SessionEnd hook scripts, and the CLAUDE.md override block. Every one
of those is load-bearing for the working Phase 1 system, so the WP is
backup-gated and fully reversible.

## Target Files

- `~/memory/` — new canonical vault (created)
- `~/.claude/memory` — becomes a symlink → `~/memory/` (was a real directory)
- `~/.claude/scripts/hooks/session-start-memory.cjs` — path resolution updated
- `~/.claude/scripts/hooks/qmd-refresh.cjs` — collection names updated
- `~/.claude/CLAUDE.md` — memory override block points to `~/memory/`
- `~/.llm-memory/config` — new: `LLM_MEMORY_VAULT_PATH`, `LLM_MEMORY_REPO`
- `~/.cache/qmd/` — QMD collections re-registered
- `~/.memory-migration-backup-<timestamp>/` — full pre-migration backup (temp)

## Implementation Steps

### Phase 0: Pre-flight + backup (MANDATORY — gate for everything else)

1. **Assert clean baseline.** Run `verify-memory.cjs` — abort if not 37/37 green.
   Migrating a broken store is never correct.
2. **Snapshot QMD state.** `qmd collection list > backup/qmd-collections.txt`.
3. **Full backup.** `cp -a ~/.claude/.memory/ ~/.memory-migration-backup-<ts>/memory/`
   and copy the three hook scripts, `hooks.json`, and `~/.claude/CLAUDE.md` into
   the same backup dir. Record the backup path in `_meta/migration-log.md`.
4. **Resolve the symlink fact.** `~/.claude` is itself a symlink into the
   dotfiles repo (`/home/cunger/dotfiles/claude`). `~/.claude/.memory/` therefore
   currently resolves to `dotfiles/claude/memory/`. The migration must move the
   *real directory* out of the dotfiles repo to `~/memory/` (a non-repo path) and
   leave a symlink behind. Record both the literal and resolved paths in the log.
   **Decision point:** `~/memory/` is outside the dotfiles repo — confirm with the
   user whether the vault should be its own git repo or stay un-git-tracked
   (OQ-11, see OPEN_QUESTIONS.md).

### Phase 1: Create the `~/memory/` vault structure

```
~/memory/
├── index.md            ├── log.md            ├── hot.md
├── .manifest.json      ├── USER.md           ├── GLOSSARY.md
├── concepts/   entities/   skills/   references/   synthesis/   journal/
├── projects/           ├── _raw/             ├── _meta/        ├── _archive/
└── .obsidian/          (graph config, default workspace)
```

`_raw/`, `_archive/`, and `.obsidian/workspace*` are gitignored if the vault is
git-tracked. The category set matches the `llm-memory` schema (WP-2).

### Phase 2: Migrate Phase 1 content (direct mapping — copy, then verify, then cut)

Per OQ-2 (resolved): map the Phase 1 typed files into memory vault categories.

| Phase 1 source | → | Phase 2 destination |
|----------------|---|---------------------|
| `memory/user/*.md` | → | `~/memory/entities/` (frontmatter `category: entity`) |
| `memory/feedback/*.md` | → | `~/memory/references/` |
| `memory/reference/*.md` | → | `~/memory/references/` |
| `memory/MEMORY.md` | → | `~/memory/index.md` (reformatted to memory index) |
| `memory/USER.md` | → | `~/memory/USER.md` (kept as-is) |
| `memory/GLOSSARY.md` | → | `~/memory/GLOSSARY.md` (kept as-is) |

Each migrated file gets memory frontmatter added (title, category, tags, sources,
`lifecycle: verified` — these are human-authored, not freshly ingested). Copy
first; do not delete the Phase 1 tree until Phase 5 verification passes.

### Phase 3: Symlink + path swap

1. Move the now-migrated real directory aside, create `~/memory/` as the real
   vault, and replace `~/.claude/memory` with a symlink → `~/memory/`.
2. Create `~/memory/_raw/plans/` → symlink to `~/.claude/plans/` (the `_raw/`
   bridge — design rationale becomes QMD-searchable; OQ-5 resolved).
3. Verify `readlink ~/.claude/memory` resolves and `ls ~/.claude/.memory/` shows
   the vault.

### Phase 4: Update hooks + QMD registration

1. `session-start-memory.cjs` / `qmd-refresh.cjs`: update the global memory path
   constant from `~/.claude/memory` to `~/memory` (the symlink keeps the old path
   working, but the scripts should resolve canonically). Apply the deferred
   Phase 1 code-review fix here — the `startsWith` path-boundary check (see G4 in
   OVERVIEW.md).
2. Re-register the QMD collection: `qmd collection add ~/memory` as
   `memory-global`; drop the stale `~/.claude/memory` registration.
3. Run `qmd-refresh.cjs --force` and confirm `qmd search` returns a hit from a
   migrated page.

### Phase 5: Write config + verify

1. Write `~/.llm-memory/config`: `LLM_MEMORY_VAULT_PATH=~/memory`,
   `LLM_MEMORY_REPO=<framework-repo-path>` (WP0's repo).
2. Update the `~/.claude/CLAUDE.md` memory override block → `~/memory/`.
3. Run `verify-memory.cjs` (WP-15 extends it; for now the Phase 1 checks must
   still pass against the new paths) — expect ALL GREEN.
4. Run the SessionStart hook — valid framed JSON, ≤ 2,500 tokens, migrated
   content present.
5. **Only after 1–4 pass:** the Phase 1 `memory/` directory in the dotfiles repo
   is removed. Until then it stays as a fallback.

### Rollback (if any phase fails)

1. `rm` the partial `~/memory/` and the `~/.claude/memory` symlink.
2. `cp -a ~/.memory-migration-backup-<ts>/memory/` back to its original path.
3. Restore the three hook scripts, `hooks.json`, `CLAUDE.md` from the backup.
4. `qmd collection add` the original `~/.claude/memory` path; `qmd-refresh --force`.
5. Run `verify-memory.cjs` — must return to 37/37. Log the failure cause.

The backup directory is kept until the user confirms a clean migration, then
removed.

## Recommended Agents

- `code-reviewer` — post-migration review of the updated hook scripts.

## Verification

See VERIFICATION.md WP-1 section:
- V1.1: `verify-memory.cjs` 37/37 green BEFORE migration (pre-flight gate).
- V1.2: backup directory exists and contains memory tree + hooks + CLAUDE.md.
- V1.3: `~/memory/` has all 7 categories + special files + `_raw/`/`_meta/`/`_archive/`.
- V1.4: `readlink ~/.claude/memory` → `~/memory`; old path still resolves.
- V1.5: every Phase 1 fact present in `~/memory/` under the mapped category with
  memory frontmatter.
- V1.6: `qmd collection list` shows `memory-global` at `~/memory`; no stale
  `~/.claude/memory` registration; `qmd search` returns a migrated-page hit.
- V1.7: `verify-memory.cjs` 37/37 green AFTER migration.
- V1.8: rollback rehearsal — on a copy, force a mid-migration failure and confirm
  the rollback restores 37/37.
