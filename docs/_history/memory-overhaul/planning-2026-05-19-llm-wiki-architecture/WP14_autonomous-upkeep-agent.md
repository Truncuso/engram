# WP14: Autonomous Upkeep Agent

**Severity**: MEDIUM
**Status**: Phase 2b — PLANNING (unblocked; OQ-8 and OQ-9 resolved 2026-05-19)
**Depends on**: WP1 (vault migration), WP2 (llm-memory skill), WP3 (memory-init overhaul),
WP4 (ingest + query skills), WP5 (housekeeping skills), WP8 (synthesis + digest skills),
WP12 (cron framework ready).
Execution: Phase 6 in OVERVIEW.md — final phase alongside WP15.

## Problem

The memory vault operates manually: a user runs `/memory-ingest` after adding sources, `/memory-lint`
when they remember, `/memory-digest` when curious. This is fragile. Sources accumulate in
`_raw/` unprocessed. QMD indexes grow stale. Dead pages linger. The user is the
scheduler, and the user forgets.

The Autonomous Upkeep Agent replaces the user as scheduler for all repetitive memory
operations. It is a three-layer system that runs in the background:

- **Layer 1 — QMD Index Daemon**: keeps the QMD search index fresh (no LLM calls).
- **Layer 2 — Ingestion Agent**: scans `_raw/` for new/changed content, auto-ingests it.
- **Layer 3 — Curation Agent**: weekly lint + synthesize + staleness scan; monthly
  digest + dedup.

The agent is **configurable** (LLM backend, budget caps, and schedules set in
`<vault>/_meta/config.yaml`), **budget-aware** (daily token and monthly cost limits
enforced per API response or char/4 estimate), and **non-destructive** (ingestion is
append-only; curation is report-first with explicit flags required for writes).

## Resolved Design Decisions

These were OQ-8 and OQ-9; both resolved 2026-05-19 (see OPEN_QUESTIONS.md). They are
firm decisions, baked into the spec below — not open options.

- **OQ-8a — Detection: periodic scan.** A timer-driven script walks `_raw/`, computes
  SHA-256 of each source, compares against `_meta/.manifest.json`, enqueues changed
  sources. No resident inotify daemon — matches the session-cron model, works on all
  filesystems. Scan latency = the timer interval (10 min).
- **OQ-8b — Process model: two separate cron jobs.** Ingestion and curation run on
  independent schedules with independent failure domains and budgets. An ingestion
  failure never cancels curation and vice versa.
- **OQ-8c — Fallback: queue with exponential backoff.** When the LLM backend is offline
  or a token/cost budget is exceeded: log the reason to `_meta/agent-errors.log`,
  enqueue the work item in `_meta/agent-queue.json`, and retry on the next run with
  `delay = scan_interval * 2^(attempt-1)`, capped at `retry_limit` (default 5). Items
  past the cap are dropped from the queue and a warning is appended to `hot.md`.
  **Layer 1 (the QMD daemon) keeps running regardless** — it makes no LLM calls.
- **OQ-9 — Budget / token counting: trust API counts, estimate local.** API backends
  (Anthropic, OpenAI): read exact token counts from the response `usage` field. Local
  backends (Ollama): estimate via `floor(len(prompt + completion) / 4)`.
  `max_monthly_cost_usd` applies only to paid API backends — Ollama is cost-free, so
  only `max_daily_tokens` gates it.

## Target Files

Daemon scripts live in the framework repo and are **copied** (not symlinked) into the
vault at install time — daemon scripts must not auto-update mid-execution.

```
<framework-repo>/scripts/daemons/
├── qmd-index-daemon.sh        # Layer 1: QMD update + embed loop (no LLM)
├── ingest-agent.sh            # Layer 2: _raw/ scan, manifest diff, memory-ingest dispatch
├── curate-agent.sh            # Layer 3: weekly lint+synthesize+staleness; monthly digest+dedup
└── budget.cjs                 # Shared budget tracking helper (Node.js CommonJS)

<framework-repo>/config/
├── config.yaml.example        # Full agent-config template, all keys documented
└── systemd/                   # 4 service+timer unit pairs (qmd-daemon, ingest,
                                #   curate-weekly, curate-monthly)

<framework-repo>/scripts/verify/
└── verify-daemon.sh           # V14.1–V14.7 verification script

<vault>/_meta/                  # Runtime state (auto-created by memory-init, WP3)
├── config.yaml                # Agent config (copied from config.yaml.example on first run)
├── .manifest.json             # SHA-256 per source file in _raw/ (created by WP3/WP4)
├── .qmd-last-refresh          # Unix timestamp: last successful qmd update (atomic write)
├── .qmd-last-embed            # Unix timestamp: last successful qmd embed
├── agent-queue.json           # Pending work items with retry metadata
├── agent-state.json           # Per-layer last-run timestamps and status
├── agent-errors.log           # Failure log (backend unreachable, budget exceeded, …)
├── budget.json                # Rolling token + cost accumulator
└── daemons/                   # Runtime copies of the daemon scripts (installed by WP3)
```

**Config-path note:** `~/.llm-memory/config` (no extension, written by WP1/WP3) is the
machine-level *path pointer* — it holds `LLM_MEMORY_VAULT_PATH` and `LLM_MEMORY_REPO`.
The agent's richer settings (backend, budget, schedules) live in the vault at
`<vault>/_meta/config.yaml`. Two distinct files, two distinct purposes — the daemon
scripts read `~/.llm-memory/config` to find the vault, then read
`<vault>/_meta/config.yaml` for their own settings.

## Implementation Steps

### Step 1: Agent config schema (`<vault>/_meta/config.yaml`)

`config.yaml.example` in the framework repo is the template; `memory-init` (WP3) copies
it to `<vault>/_meta/config.yaml` on first install if absent. Re-running `memory-init`
never overwrites an existing config (idempotent). Keys:

```yaml
backend:
  provider: anthropic              # anthropic | openai | ollama | openrouter
  model: claude-sonnet-4-6
  api_key_env: ANTHROPIC_API_KEY   # env var name holding the key — never the key itself
  base_url: null                   # override for Ollama / OpenRouter
budget:
  max_daily_tokens: 200000         # 0 = unlimited
  max_monthly_cost_usd: 20         # 0 = unlimited; ignored for local (Ollama) backends
qmd_daemon:
  enabled: true
  update_interval_minutes: 10
  embed_interval_hours: 1          # debounced: at most one embed per hour
ingestion:
  enabled: true
  scan_interval_minutes: 10
  max_files_per_batch: 5
  retry_limit: 5
  append_mode: true                # hard constraint — never overwrite human edits
curation:
  enabled: true
  weekly:  { day: saturday, lint_hour: 10, synthesize_hour: 11, staleness_hour: 12,
             auto_consolidate: false }
  monthly: { digest_day: 1, digest_hour: 9, dedup_day: 2, dedup_hour: 9 }
```

### Step 2: Layer 1 — QMD Index Daemon (`qmd-index-daemon.sh`)

Installed as a **systemd user timer** (Linux), firing every 10 minutes. No LLM calls;
runs regardless of budget or backend state.

1. Source `~/.llm-memory/config` to locate `LLM_MEMORY_VAULT_PATH`.
2. Read `_meta/.qmd-last-refresh` (missing = 0). If `now - last >= update_interval`,
   for each collection in `_meta/qmd-collections.json` (AF-4 multi-collection model):
   `qmd update --name <collection>`; then atomically write the timestamp
   (`.qmd-last-refresh.tmp` → rename `.qmd-last-refresh`).
3. Read `_meta/.qmd-last-embed`. If `now - last >= embed_interval`, `qmd embed --name
   <collection>` per collection; atomically update `.qmd-last-embed`.
4. Never calls `qmd index` (full re-index — memory-init only) or `qmd update --force`.

### Step 3: Layer 2 — Ingestion Agent (`ingest-agent.sh`)

Installed as a **CronCreate job** (WP12), every 10 min, offset 2 min from the QMD daemon.

1. `budget.cjs check --type daily_tokens` → if exceeded, log to `agent-errors.log`, exit 0.
2. Walk `_raw/` recursively, **excluding `_raw/projects/`** (project-vault symlinks).
   SHA-256 each file; compare against `_meta/.manifest.json`; changed/new → work list.
3. Add queued items from `agent-queue.json` whose `next_retry_at <= now`.
4. Truncate the work list to `max_files_per_batch`.
5. Per item: invoke `memory-ingest <file> --mode append` (append is unconditional — the
   agent never passes `--full`). On success: upsert the manifest entry, record budget,
   dequeue if queued, append a line to `log.md`. On failure: append to
   `agent-errors.log`, upsert into `agent-queue.json` with `attempt+1` and
   `next_retry_at = now + scan_interval * 2^attempt`; if `attempt >= retry_limit`,
   drop from queue and append a warning to `hot.md`.
6. After the batch: `qmd-refresh.cjs --force` on affected collections.

### Step 4: Layer 3 — Curation Agent (`curate-agent.sh`)

Installed as **two CronCreate jobs** (WP12): weekly (Saturday) and monthly. Both invoke
`curate-agent.sh --mode {weekly|monthly}`.

- **Weekly:** `memory-lint --report-only` → report file; if `auto_consolidate: true`,
  apply only the safe mechanical fixes (broken-wikilink resolution, frontmatter enum
  normalization, tag-alias canonicalization) — never merge/synthesize/archive. Then
  `memory-synthesize` (writes co-occurrence *candidates*, never auto-creates pages); then
  a no-LLM staleness scan (`updated` > 90 days).
- **Monthly:** `memory-digest --period monthly` → `journal/digest-<YYYY-MM>.md`; then
  `memory-curate --mode DEDUP` (writes dedup *candidates*, never auto-merges).
- The curation agent is **read-by-default, write-on-request**. All judgement-heavy
  operations (merge, archive, synthesize, contradiction resolution) are permanently
  human-gated.

**Relationship to WP-12's CronCreate jobs:** WP-12 registers session-context CronCreate
jobs (`/daily-update`, `/memory-lint --consolidate`, `/memory-digest monthly save`) that
fire inside an interactive Claude session. WP-14's daemon scripts run autonomously
*outside* any session. They **coexist** and serve different contexts — the CronCreate
job is the user-facing interactive maintenance; the daemon is the unattended backstop.
Where both target the same operation (weekly lint), the daemon runs report-only
(`auto_consolidate: false` by default) and the CronCreate job runs `--consolidate`
interactively — no conflict, the daemon never writes what the interactive job would.
WP-12 and WP-14 each state this explicitly.

### Step 5: Budget tracking (`budget.cjs`)

Shared Node CommonJS module: `node <vault>/_meta/daemons/budget.cjs <subcommand>`.
Reads `_meta/config.yaml`; reads/writes `_meta/budget.json`.

- `check --type daily_tokens` / `check --type monthly_cost` → exit 0 allowed / 1
  exceeded; `monthly_cost` always allowed for local providers.
- `record --tokens N --cost-usd F --model M --provider P --operation OP` → appends to a
  rolling op-log, accumulates daily/monthly counters, auto-resets on date rollover.
- **Security:** `budget.cjs` reads `backend.api_key_env` to know *which* env var holds
  the key — it never reads, logs, or stores the key value.

### Step 6: SessionStart hook interaction (AF-5)

No locking: the hook is read-only on `_meta/.qmd-last-refresh`, the daemon is the sole
writer (atomic rename). `session-start-memory.cjs` reads the timestamp; if a daemon
refresh happened within the last 10 min, the hook skips its own `qmd update`. No race.

### Step 7: Install via `memory-init` (WP-3 extension)

WP-3 Phase 10 already calls the systemd-timer install. This WP supplies the scripts and
unit files Phase 10 references. On `memory-init --global`: copy the four daemon scripts
to `<vault>/_meta/daemons/` (runtime copies, executable bit set); copy the systemd unit
files to `~/.config/systemd/user/`; `systemctl --user daemon-reload`; enable the
QMD-daemon timer; initialise `config.yaml`, `budget.json`, `agent-state.json`,
`agent-queue.json` if absent. All steps existence-guarded — idempotent.

### Step 8: Verification script (`verify-daemon.sh`)

See Verification below — covers V14.1–V14.7.

## Recommended Agents

- `code-reviewer` — all three daemon scripts, `budget.cjs`, the config schema, and
  `verify-daemon.sh` (functions < 50 lines, explicit error handling, no hardcoded values).
- `security-reviewer` — confirm `budget.cjs` never logs API-key values; confirm
  `agent-errors.log` and `hot.md` warnings contain no credentials; review env-var
  handling in the shell scripts.
- `tdd-guide` — write V14.4 (scan detection) and V14.5 (queue/backoff) before
  implementing the corresponding agent logic.

## Verification

See VERIFICATION.md WP14 section. `<framework-repo>/scripts/verify/verify-daemon.sh`
covers:

- **V14.1** — `systemctl --user is-active llm-memory-qmd-daemon.timer` exits 0.
- **V14.2** — the three `.sh` daemon scripts are executable; `ingest-agent.sh --dry-run`
  exits 0 and writes nothing to `.manifest.json`/`agent-queue.json`/`log.md`/`hot.md`.
- **V14.3** — `config.yaml` valid: all required keys present; `backend.provider` ∈
  {anthropic, openai, ollama, openrouter}; budget fields non-negative.
- **V14.4** — manifest scan detects a changed source: seed `_raw/test-wp14.md`, run
  `ingest-agent.sh --dry-run`, confirm it appears in the detected-changes output;
  clean up.
- **V14.5** — queue + exponential backoff: point `backend.base_url` at a dead port, run
  the agent on a seeded file, confirm `agent-queue.json` shows `attempt: 1` with a
  future `next_retry_at`; run again → `attempt: 2`, ≥ 2× delay; after `retry_limit`
  runs the item is dropped and a `hot.md` warning appears. Clean up.
- **V14.6** — budget enforcement: seed `budget.json` at the daily-token limit, run the
  agent, confirm no LLM call (manifest unchanged) and a budget-exceeded line in
  `agent-errors.log`. Restore.
- **V14.7** — no SessionStart race: seed `.qmd-last-refresh` 5 min ago → hook
  refresh-check returns `skip`; 12 min ago → returns `refresh`. Restore.

## Complexity Delta

- **Added**: three daemon scripts, `budget.cjs`, the `config.yaml` schema, 8 systemd
  unit files, `agent-queue.json` retry state, `budget.json` accumulator, the AF-5
  coordination contract.
- **Removed**: nothing (new capability).
- **Justification**: the three-layer daemon is what makes the memory vault self-maintaining
  instead of relying on the user to remember maintenance commands. Each layer is
  independently testable with a well-defined failure mode (skip + log + `hot.md`
  notice — never corrupt state, never overwrite human edits). The only non-trivial
  coupling — the queue + backoff — is fully encapsulated in `ingest-agent.sh` and
  exercised by V14.5.
