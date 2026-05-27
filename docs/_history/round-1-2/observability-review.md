---
title: "engram — Observability & Traceability Audit"
project: engram
spec_version: DRAFT 2026-05-22
review_date: 2026-05-22
reviewer: devops-engineer (Claude)
status: DRAFT — gaps identified, recommendations pending resolution
tags: [observability, traceability, devops, audit]
---

# engram — Observability & Traceability Audit

This document audits the SPEC (`docs/spec/SPEC.md`) for observability and
traceability gaps, and proposes the concrete "Observability & Traceability"
section the spec is missing. It is a design-level audit — no code exists.

---

## 1. Traceability of Memory Lifecycle

### 1.1 What the spec says

§6.3 states: *"Traceability ('who wrote what part of a memory; which agent') is
satisfied by `author` + the app-log's per-field attribution + git blame."*

The `author` frontmatter field records the creating agent. The `version` token
(OCC) records how many writes occurred. Git records file-level diffs with the
agent identity as committer. The app-log records per-field changes.

### 1.2 The gap

The stated three-layer answer (`author` + app-log + git blame) is directionally
correct but incomplete for lifecycle traceability. Specifically:

**What it cannot answer today:**

1. **Ingest chain.** A memory with `origin: ingested` and `sources: ["raw/X.pdf"]`
   has no record of *which ingest run* produced it, *which LLM call* distilled
   it, *what prompt was used*, or *what the LLM decided to omit*. If distillation
   produced a wrong or incomplete memory, there is no audit trail back to the
   LLM invocation.

2. **Dreaming provenance on fields.** When dreaming re-weights `importance` from
   0.4 → 0.72 and advances `lifecycle` from `active` → `dormant`, the app-log
   records "what" but not "why": which scoring calculation drove the decision,
   which dreaming-memory object ran, what the score breakdown was. A human
   reviewing the timeline cannot verify whether the transition was correct.

3. **Staging → memory traceability.** When dreaming distills a staging
   observation into a curated `episodic` memory, there is no field linking the
   resulting memory back to the source staging file(s). The `sources:` field is
   defined for raw ingest, not for capture staging. If a staging observation is
   discarded as noise, there is no record it ever existed.

4. **OCC conflict trace.** When an OCC write conflict is rejected and retried, the
   conflict event is not recorded anywhere. If an agent repeatedly loses races
   against dreaming, this pathology is invisible.

5. **`author` is a single field.** After an agent writes a memory and dreaming
   subsequently modifies the body, `author` still reflects the original creator.
   The field conflates *created by* with *last modified by*, and cannot represent
   a memory that has been edited by three different agents over its lifetime.

6. **Session linkage.** `session` is set for session-scoped memory, but there is
   no reverse index: given a session ID, you cannot efficiently enumerate all
   memory written or modified during that session without scanning all files.

### 1.3 Recommendation: Complete provenance model

A complete audit trail for a single memory must capture:

```
BIRTH
  created_by:    agent identity
  created_at:    timestamp
  origin:        ingested | agent-session | self-authored | human
  ingest_run:    ingest_run_id (if origin=ingested) → links to ingest log
  session:       session ID (if origin=agent-session)
  staged_from:   [staging_file_id, ...] (if distilled from staging)

MUTATIONS (one record per field-level change)
  event_id:      ULID
  memory_id:     mem_ULID
  field:         "importance" | "lifecycle" | "body" | "tags" | ...
  old_value:     previous value
  new_value:     new value
  changed_by:    agent | dream:<dreaming-memory-name>:<run-id> | human
  changed_at:    timestamp
  reason:        free text / score breakdown / LLM rationale
  version_from:  OCC version before change
  version_to:    OCC version after change
  dream_run_id:  dream run ID if changed_by is dreaming

LIFECYCLE TRANSITIONS (specialized mutation record)
  transition:    active→dormant | dormant→active | dormant→archived
  score_at_time: {importance, recency_factor, access_count}
  threshold:     the configured threshold that triggered the transition
  decision_by:   dream:<name>:<run-id> | manual

ACCESS EVENTS
  accessed_by:   agent identity
  accessed_at:   timestamp
  via_query:     query string that recalled this memory (if via recall)
  score_at_time: {importance, relevance, recency_factor}

GOVERNANCE ACTIONS
  action:        governance_delete | visibility_change | scope_change
  performed_by:  agent | human
  performed_at:  timestamp
  justification: required text
```

The `author` frontmatter field should be **split into `created_by`** (immutable,
set on birth) **and `last_modified_by`** (updated on every mutation). The full
history lives in the app-log; frontmatter holds only the current-state summary.

---

## 2. The App-Log — Schema Definition

### 2.1 What the spec says

§6.3: *"an append-only journal in `.engram/` records per-memory, per-field
provenance: who changed `importance`, when, why, old→new. This is finer than
git (field-level) and is what the dashboard renders as a per-memory timeline."*

This is the entire spec for the app-log. No format, no schema, no query model,
no retention, no overlap analysis with git.

### 2.2 The gap

The app-log is described at the concept level but is fully unspecified as an
engineering artifact. The dashboard/CLI's `memory.history` verb depends on it,
but there is no defined:

- File format (JSONL? SQLite? flat file per memory?)
- Schema (what fields does each record carry?)
- Indexing strategy (how does `memory.history <id>` run fast at scale?)
- Retention policy (does the app-log grow forever? per-store, global?)
- Relationship to git (are these two redundant stores of the same fact? what
  does each own?)
- Write path (who writes to the app-log? the kernel write path? dreaming?)
- Query surface (does the CLI/dashboard query it via SQL? by scanning JSONL?)

### 2.3 Recommended app-log schema

**Format:** SQLite database at `.engram/app-log.db`. Rationale: the job queue
is already planned as SQLite (OQ-I); co-locating the app-log in SQLite gives
indexed queries at no additional infrastructure cost. JSONL is simpler to write
but requires full scan for `memory.history <id>` at scale. SQLite FTS5 can
index the `reason` field.

Alternatively, a JSONL file at `.engram/app-log.jsonl` is acceptable for v1
if the expected record count is low (< 100k). The CLI can use `grep`/`jq` for
queries. The dashboard port must migrate to SQLite before v2.

**Primary schema (JSONL line / SQLite row):**

```jsonc
// Every app-log record
{
  "event_id":     "evt_01HX...",        // ULID, globally unique
  "event_type":   "write|lifecycle|access|governance|dream_run|ingest_run|capture|occ_conflict",
  "memory_id":    "mem_01HX...",        // FK to memory; null for store-level events
  "store":        "global|<repo>",      // which store
  "ts":           "2026-05-22T08:30:00.000Z",

  // --- write events ---
  "field":        "importance",         // null for full-body writes
  "old_value":    "0.40",
  "new_value":    "0.72",
  "changed_by":   "agent:claude-code",  // agent:<id> | dream:<name>:<run_id> | human | system
  "reason":       "score recalculation: centrality 0.8, connection count 12",
  "version_from": 6,
  "version_to":   7,

  // --- dream context (when changed_by = dream:*) ---
  "dream_run_id": "drun_01HX...",
  "dream_name":   "claude-code-work",

  // --- ingest context (when event_type = ingest_run) ---
  "ingest_run_id": "irun_01HX...",
  "source_path":   "raw/paper.pdf",
  "llm_model":     "claude-opus-4",
  "memories_created": ["mem_01HX...", "mem_01HY..."],

  // --- access events ---
  "query":        "how does JWT auth work",
  "score_snapshot": {"importance": 0.72, "recency_factor": 0.91, "relevance": 0.88},

  // --- governance ---
  "justification": "GDPR erasure request #42",

  // --- OCC conflict ---
  "conflict_version_seen": 5,
  "conflict_version_current": 6,
  "retried": true
}
```

**Indexing (SQLite):** `(memory_id, ts)` for per-memory timeline; `(event_type, ts)` for operational queries; `(dream_run_id)` for dream audit; `(changed_by, ts)` for per-agent audit.

**Relationship to git:**

| Concern | Owner | Rationale |
|---------|-------|-----------|
| File body diffs | git | Line-level diff, history browsing, rollback |
| Per-field old→new with reason | app-log | git cannot represent sub-file field granularity |
| Dreaming branch lifecycle | git | branch create/merge/discard is git-native |
| Access events | app-log only | git has no concept of reads |
| Score snapshots at time of change | app-log only | git cannot diff computed values |
| Agent identity on write | both | git commit author + app-log `changed_by` — intentional redundancy; app-log is the queryable surface, git is the forensic backup |
| Store-wide rollback | git only | `git revert` or branch operations |

The overlap on "agent identity on write" is deliberate: git blame is available
even if the app-log is corrupt or missing. The app-log is the primary operational
query surface; git is the durable forensic layer.

**Retention:** The app-log is append-only and never pruned automatically. For
stores with high capture volume, `access` events may be sampled (e.g. record
every 10th access per memory per day) to bound growth. A `engram log prune
--before <date> --keep-mutations` command can archive old access-event rows
while preserving all mutation and governance records.

---

## 3. Daemon Observability (engramd)

### 3.1 What the spec says

§2.2 defines engramd as the always-up daemon process. §1.3 lists it as a v1
requirement. §5.5 mentions process isolation for dreaming. §7 mentions the
daemon serves the dashboard. That is the entirety of operational detail for
a long-running background process.

### 3.2 The gap

There is no specification for:

- Daemon logs (location, format, rotation)
- Health/readiness endpoint
- Operational metrics surface
- How the user learns the daemon has crashed or is unhealthy
- How the user debugs a stalled recall or a staging backlog
- Startup/shutdown sequences and error behavior

### 3.3 Recommendations

**Log output**

The daemon emits structured JSON logs to `~/.engram/logs/engramd.log` (global)
or `<store>/.engram/logs/engramd.log` (project). Log format:

```jsonc
{"ts":"2026-05-22T08:30:00.000Z","level":"info","component":"mcp-server","msg":"client connected","agent":"claude-code","client_id":"cli_01HX..."}
{"ts":"2026-05-22T08:30:01.123Z","level":"warn","component":"retrieval","msg":"QMD index stale","store":"global","age_seconds":3612}
{"ts":"2026-05-22T08:30:05.000Z","level":"error","component":"dreaming","msg":"dream worker exited with code 1","dream_run_id":"drun_01HX...","exit_code":1}
```

Rotation: daily rotation, 14-day retention, configurable. The daemon should
not be responsible for its own log rotation; the recommended approach is to
delegate to logrotate (Linux) or newsyslog (macOS) via a config file installed
alongside the systemd/launchd unit. If the daemon manages it internally, use a
rolling file appender (e.g. `winston-daily-rotate-file`).

**Health endpoint**

The daemon exposes a local HTTP health endpoint (not the MCP server — a
separate, unauthenticated port, default `127.0.0.1:7474`):

```
GET /health
→ 200 {"status":"ok","uptime_s":3612,"version":"0.1.0"}

GET /health/ready
→ 200 {"status":"ready","stores":["global","project:engram"],"qmd_index_age_s":120}
→ 503 {"status":"degraded","reason":"QMD index rebuild in progress","eta_s":45}

GET /metrics   (plain text, Prometheus exposition format)
→ engram_memories_total{store="global",type="semantic"} 142
  engram_memories_total{store="global",type="episodic"} 891
  engram_staging_backlog{store="global"} 23
  engram_recall_latency_p50_ms{store="global"} 38
  engram_recall_latency_p99_ms{store="global"} 210
  engram_qmd_index_age_seconds{store="global"} 120
  engram_dream_runs_total{status="completed"} 14
  engram_dream_runs_total{status="failed"} 1
  engram_mcp_calls_total{verb="memory.recall",status="ok"} 3204
  engram_mcp_calls_total{verb="memory.remember",status="ok"} 87
  engram_mcp_calls_total{verb="memory.recall",status="denied"} 2
  engram_occ_conflicts_total{store="global"} 5
  engram_uptime_seconds 3612
```

This surface is the primary answer to "is the daemon healthy?" for both the
user and the v2 dashboard. It requires no external monitoring stack — the CLI
can poll it (`engram status`).

**`engram status` CLI command**

```
$ engram status
engramd         running   uptime 1h 02m
  global store  ok        142 semantic, 891 episodic, 23 staging
  QMD index     ok        last built 2m ago
  dreaming      idle      last run 45m ago (completed)
  MCP server    ok        3204 calls today

$ engram status --full
... + per-verb call counts, p99 latency, OCC conflict count, recent errors
```

**Startup/shutdown contract**

- On startup: validate store layout, check git status, rebuild QMD index if
  stale (async, non-blocking to MCP availability), write PID file to
  `.engram/engramd.pid`.
- On graceful shutdown (`SIGTERM`): drain in-flight MCP requests (timeout 30s),
  wait for no active dream workers (up to configurable max), flush app-log
  write buffer, remove PID file.
- On unclean shutdown: next startup detects missing/stale PID, runs integrity
  check, reports last known state.

---

## 4. Dreaming Observability

### 4.1 What the spec says

§5 is the most detailed section of the spec. It defines what a dreaming run
*does* (distill, connect, re-weight, verify/learn, commit), the merge policy,
and the decoupling guarantees. §7 mentions a "dreaming view" in the dashboard.

What is missing: what the dreaming worker *emits* as an observable artifact.
"Watch dreaming, see what it changed" is stated as a user need but not
specified.

### 4.2 The gap

1. The dreaming run produces a git branch — but a git branch diff is a raw
   file-level diff, not a human-readable summary of dream decisions.
2. There is no defined format for "the audit record of an autonomous run."
3. The review queue (for gated operations) is mentioned but not specified —
   what is in a review queue item? What does the user see?
4. Token cost and model usage during a dream are not tracked anywhere.
5. When dreaming *discards* a staging observation as noise, there is no record
   of the discard decision.
6. The rationale for emergent-entity proposals is not captured.

### 4.3 Dream run audit record

Every completed (or failed) dream run produces a `dream-run-<run_id>.json`
file in `.dreaming/runs/` (this directory is git-tracked, so dream audit
records are versioned). Format:

```jsonc
{
  "run_id":        "drun_01HX...",
  "dream_name":    "claude-code-work",
  "dream_config":  { /* snapshot of .dreaming/claude-code-work.md at run time */ },
  "triggered_by":  "session-end|cron|mcp:dream.trigger|threshold",
  "trigger_detail": "session sess_01HX... ended",
  "started_at":    "2026-05-22T03:00:00.000Z",
  "completed_at":  "2026-05-22T03:04:22.000Z",
  "status":        "completed|failed|timeout|budget_exceeded",
  "exit_code":     0,

  // --- cost tracking ---
  "llm_model":     "claude-opus-4",
  "tokens_in":     18240,
  "tokens_out":    4120,
  "estimated_cost_usd": 0.42,
  "budget_remaining_daily_tokens": 181760,

  // --- staging processed ---
  "staging_files_considered": 47,
  "staging_files_distilled": 23,
  "staging_files_discarded": 24,
  "staging_discards": [
    {
      "file": "staging/obs_01HX....json",
      "reason": "duplicate of mem_01HY...",
      "duplicate_of": "mem_01HY..."
    },
    {
      "file": "staging/obs_01HZ....json",
      "reason": "below noise threshold",
      "score": 0.12
    }
  ],

  // --- memories touched ---
  "memories_created": [
    {"id": "mem_01HA...", "type": "episodic", "title": "OCC conflict in project X"},
    {"id": "mem_01HB...", "type": "procedural", "title": "Pattern: retry on OCC conflict"}
  ],
  "memories_modified": [
    {"id": "mem_01HC...", "fields": ["importance","lifecycle"], "old":{"importance":0.4,"lifecycle":"active"}, "new":{"importance":0.22,"lifecycle":"dormant"}, "reason":"score below 0.25 threshold for 3 consecutive runs"}
  ],
  "links_added": [
    {"from": "mem_01HA...", "to": "mem_01HB...", "kind": "derived_from", "reason": "episodic instance of this procedural pattern"}
  ],
  "links_removed": [],

  // --- emergent entities ---
  "emergent_entity_proposals": [
    {
      "term": "OCC conflict",
      "appearances": 7,
      "in_memories": ["mem_01HA...", "mem_01HC...", "mem_01HD..."],
      "proposed_memory_title": "Concept: OCC conflict in engram",
      "decision": "queued_for_review"
    }
  ],

  // --- review queue items produced ---
  "review_queue_items": [
    {
      "item_id": "rqi_01HX...",
      "kind": "archive_proposal",
      "memory_id": "mem_01HC...",
      "reason": "lifecycle advanced to archived; merge_policy=auto-safe requires human confirmation",
      "auto_approve_if_not_reviewed_by": null
    }
  ],

  // --- git ---
  "branch":        "dream/claude-code-work/2026-05-22T030000Z",
  "commit":        "abc1234",
  "merged_at":     "2026-05-22T03:04:25.000Z",
  "merge_strategy": "auto-safe",
  "gated_items":   1,

  // --- errors ---
  "errors": []
}
```

**CLI access:**

```
engram dream log                        # list recent runs
engram dream log <run_id>               # show full run record
engram dream log --since 7d             # last week
engram dream log --status failed        # failed runs only
engram dream review                     # show pending review queue items
engram dream review approve <item_id>   # approve a gated item
engram dream review reject <item_id>    # reject; reason required
```

**What the v2 dashboard renders from this:**

- "Dreaming view": timeline of runs, token cost chart, memories touched/created
  per run, staging discard rate, pending review queue with inline diff.
- "Dreaming visualization": per-run link graph delta (nodes added, edges added)
  overlaid on the memory graph.

---

## 5. MCP Call Observability

### 5.1 What the spec says

§4.4 defines the MCP verb set and §6.1 defines capability-gated access control.
There is no specification for whether MCP calls are logged or audited.

### 5.2 The gap

The MCP server is the single API surface for all agent interactions. Without an
access log:

- There is no record of which agent recalled which memory at what time.
- Denied calls (access control violations) are silent — no audit, no alerting.
- There is no way to detect a misbehaving agent (e.g. polling `memory.recall`
  at high frequency, or repeatedly attempting to access another agent's private
  memory).
- `memory.governance_delete` is described as "audited" but the audit mechanism
  is not defined.

### 5.3 Recommendation

**MCP access log** (append to app-log, `event_type: "mcp_call"`):

```jsonc
{
  "event_id":   "evt_01HX...",
  "event_type": "mcp_call",
  "ts":         "2026-05-22T08:30:00.000Z",
  "verb":       "memory.recall",
  "agent":      "agent:claude-code",
  "session":    "sess_01HX...",
  "store":      "global",
  "status":     "ok|denied|error",
  "denied_reason": null,
  "duration_ms": 38,
  "result_count": 5,           // for recall: how many results returned
  "memory_ids":  ["mem_01HX...", "..."],  // for get/remember/forget
  "query":      "JWT auth"     // for recall: the query string
}
```

**Access control audit:** denied calls MUST be logged with `status: "denied"`
and `denied_reason` (e.g. `"scope:private:agent:other-agent"`). This is the
audit trail for the "Agent B cannot recall Agent A's private memory" success
criterion.

**`memory.governance_delete` audit:** requires a mandatory `justification`
field; produces both an app-log record (`event_type: "governance"`) and a
git commit with the deletion in the commit message. The git record is the
durable proof of deletion; the app-log record is the queryable audit entry.

**Rate limiting / anomaly surface:** the daemon's metrics endpoint exposes
per-verb, per-agent call counts. The daemon itself enforces configurable rate
limits per agent (e.g. `max_recall_per_minute: 60`). Violations are logged at
`warn` level and counted in metrics.

---

## 6. Capture Observability

### 6.1 What the spec says

§4.1 defines capture hooks (`PostToolUse`, `PostToolUseFailure`, `Stop`,
`SessionEnd`, `UserPromptSubmit`) that record raw observations into `staging/`.
A privacy filter strips secrets/API keys before writing. Staging is distilled
by dreaming; noise is discarded.

What is missing: the user cannot see what was captured, what was filtered by the
privacy filter, or what was dropped and why.

### 6.2 The gap

1. **Capture is a black box.** The user's Claude Code session runs, hooks fire
   silently, and staging accumulates. There is no capture receipt.
2. **Privacy filter decisions are invisible.** If a staging observation was
   partially redacted (an API key stripped), or entirely suppressed (the whole
   observation matched a suppression rule), the user has no visibility.
3. **Staging backlog is unmonitored.** If dreaming has not run in 3 days and
   staging has 800 files, the user cannot know this without listing the directory.
4. **Capture hook failures are silent.** If a hook fails to write to staging
   (permissions, disk full, daemon unreachable), there is no user-visible error
   at session time.

### 6.3 Recommendations

**Capture receipts in app-log** (`event_type: "capture"`):

```jsonc
{
  "event_id":      "evt_01HX...",
  "event_type":    "capture",
  "ts":            "2026-05-22T08:30:00.000Z",
  "session":       "sess_01HX...",
  "hook":          "PostToolUse",
  "staging_file":  "staging/obs_01HX....json",
  "status":        "captured|filtered|suppressed|failed",
  "filter_actions": [
    {"field": "tool_output", "action": "redacted", "pattern": "API_KEY"}
  ],
  "suppression_reason": null,
  "size_bytes":    1240
}
```

**Privacy filter audit:** when a suppression rule matches, write a
`status: "suppressed"` record with `suppression_reason: "matched rule: secrets"`.
No content is written (that is the point), but the *fact* that an observation
was suppressed is recorded. This prevents silent data loss and lets the user
verify that filtering is working.

**Staging dashboard (CLI):**

```
$ engram capture status
Session sess_01HX...  (active, 2h 14m)
  Captured:    47 observations
  Filtered:    3 (partial redaction: API keys)
  Suppressed:  1 (matched secrets rule)
  Failed:      0
  Staging backlog: 47 files, 12 KB

$ engram capture list --session <id>   # list staging files for a session
$ engram capture show <staging_file>   # inspect a staging observation
$ engram capture suppress-rules        # show active suppression rules
```

**Capture hook failure handling:** if the capture hook cannot reach the daemon
(daemon down, timeout), it writes a minimal fallback record to a local emergency
buffer at `~/.engram/capture-fallback/`. The next daemon startup drains this
buffer into staging. The user is warned via daemon log and `engram status`.

---

## 7. Operational Concerns

### 7.1 What the spec says

§3.3 defines store layout. §8.1 confirms TypeScript/Node. §2.2 defines
`engramd` as always-up. §5.5 discusses dreaming process isolation. No
installation, upgrade, backup, log rotation, or systemd/launchd specification
exists.

### 7.2 State inventory

State is spread across locations that must all be documented:

| Location | Content | Backup-required |
|----------|---------|----------------|
| `~/.engram/` | Global store: `memories/`, `staging/`, `.dreaming/`, `.engram/app-log.db`, `.engram/engramd.pid`, `.git/` | yes |
| `<repo>/.engram/` | Project store (same layout) | yes (with repo) |
| `~/.engram/logs/` | Daemon logs | no (rotated) |
| `.dreaming/runs/` | Dream run audit records | yes (git-tracked) |
| `.engram/qmd-index/` | QMD index (derived, rebuildable) | no |
| `~/.engram/capture-fallback/` | Emergency capture buffer | yes |

### 7.3 Install/upgrade

The spec declares `github.com/Truncuso/engram` but does not specify how
the user installs it. Minimum required:

```
npm install -g engram      # or: npx engram@latest
engram install             # writes systemd/launchd unit, sets up log rotation
engram install --upgrade   # upgrade daemon binary, restart service
```

The `engram install` command must:
- Detect OS (Linux/macOS) and write the appropriate service unit.
- Install logrotate config (Linux) or newsyslog config (macOS).
- Validate Node version requirement.
- Confirm capture plugin hooks are installed in the correct harness config.
- Emit a post-install summary: what was installed, where state lives, how to
  check status (`engram status`).

**Upgrade:** the daemon must support graceful restart (`SIGUSR2` or
`engramd --reload`) so an upgrade does not drop in-flight MCP requests.
Dream workers in flight at upgrade time must complete on the old binary
(they are separate processes — this is naturally handled by process isolation).

### 7.4 systemd unit (Linux)

```ini
# /etc/systemd/user/engramd.service  (user-level — no root required)
[Unit]
Description=engram agentic memory daemon
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/node /usr/lib/engram/dist/engramd.js
Restart=on-failure
RestartSec=5s
StandardOutput=journal
StandardError=journal
SyslogIdentifier=engramd
Environment=NODE_ENV=production
# Resource limits — dream workers are separate processes; limit the daemon only
MemoryMax=512M
CPUQuota=50%

[Install]
WantedBy=default.target
```

**Dreaming cron (alternative to daemon-scheduled):** if the user prefers not
to rely on the daemon's internal scheduler, a systemd timer:

```ini
# /etc/systemd/user/engram-dream.timer
[Timer]
OnCalendar=*-*-* 03:00:00
RandomizedDelaySec=600

# /etc/systemd/user/engram-dream.service
[Service]
Type=oneshot
ExecStart=/usr/bin/node /usr/lib/engram/dist/dream-cli.js --trigger cron
```

The daemon's internal scheduler and the systemd timer must not both fire
simultaneously — the daemon detects an already-running dream job and skips.

### 7.5 launchd plist (macOS)

```xml
<!-- ~/Library/LaunchAgents/io.engram.daemon.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key><string>io.engram.daemon</string>
  <key>ProgramArguments</key>
  <array>
    <string>/usr/local/bin/node</string>
    <string>/usr/local/lib/engram/dist/engramd.js</string>
  </array>
  <key>RunAtLoad</key><true/>
  <key>KeepAlive</key><true/>
  <key>StandardOutPath</key><string>/Users/USER/.engram/logs/engramd.log</string>
  <key>StandardErrorPath</key><string>/Users/USER/.engram/logs/engramd-error.log</string>
  <key>EnvironmentVariables</key>
  <dict><key>NODE_ENV</key><string>production</string></dict>
</dict>
</plist>
```

### 7.6 Backup and restore

```
engram backup --output engram-backup-$(date +%Y%m%d).tar.gz
  # tars: memories/, .dreaming/, app-log.db, dreaming run records
  # excludes: staging/ (transient), qmd-index/ (derived), logs/ (transient)

engram restore engram-backup-20260522.tar.gz
  # validates schema version, rebuilds QMD index, confirms git state
```

The git history of the store IS the primary backup for the `memories/` content.
The `engram backup` command is a convenience wrapper for state that lives
outside git (app-log.db if stored outside the git-tracked tree) and for users
who run git-off stores.

**Note:** if `app-log.db` is placed *inside* the git-tracked store root (i.e.
`.engram/app-log.db` is git-tracked), the git history is sufficient backup.
This is the recommended default. If the app-log is excluded from git for size
reasons, it must be backed up separately.

### 7.7 Log rotation

logrotate config (`/etc/logrotate.d/engramd` or equivalent):

```
~/.engram/logs/engramd.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    postrotate
        systemctl --user reload engramd 2>/dev/null || true
    endscript
}
```

### 7.8 Schema migrations

The app-log and store layout will evolve across versions. The spec must
establish a schema version mechanism from day one:

- `.engram/schema-version` file containing a semver string.
- `engramd` refuses to start against a store with a newer schema version than
  it knows.
- `engram migrate` applies forward migrations; never automatic on daemon start.
- Migrations are logged to the daemon log and to a `migration` event in the
  app-log.

---

## 8. Proposed Spec Section: Observability & Traceability

The following is the complete text of a new `§ OT` section the spec should
incorporate. It is intentionally written in the spec's voice.

---

### § OT. Observability & Traceability

#### OT.1 Design principles

1. **No silent mutations.** Every change to a memory — by any agent, dreaming
   run, or governance action — produces an app-log record. There is no write
   path that bypasses the log.
2. **No silent capture.** Every capture hook invocation, including suppressed
   and filtered ones, produces an app-log record. The user can always account
   for what was captured, filtered, and dropped.
3. **Dream runs are self-documenting.** Every completed or failed dream run
   produces a structured audit record in `.dreaming/runs/`. The record is
   git-tracked alongside the dream branch it produced.
4. **Metrics over documentation.** The daemon exposes a Prometheus metrics
   endpoint. "Is the system healthy?" is answered by `engram status`, not by
   log-grepping.
5. **The app-log is the queryable history; git is the forensic backup.**
   These two layers are complementary and intentionally overlapping on agent
   identity. Neither is authoritative for what the other owns.

#### OT.2 App-log

**Location:** `.engram/app-log.db` (SQLite) within the store root. Git-tracked
by default. If the store is git-disabled, the app-log is backed up by
`engram backup`.

**Format:** one row per event. Event types:

| event_type | When written |
|-----------|-------------|
| `write` | any memory field mutation |
| `lifecycle` | active↔dormant↔archived transition |
| `access` | memory retrieved via `memory.recall` or `memory.get` |
| `governance` | `governance_delete`, visibility/scope change |
| `mcp_call` | every MCP verb invocation (ok, denied, error) |
| `capture` | every capture hook invocation |
| `dream_run` | dream run start, completion, failure |
| `ingest_run` | ingest pipeline start/completion |
| `occ_conflict` | OCC write rejection |
| `migration` | schema migration applied |

Schema: see §OT.2 specification (to be filled during plan — see `docs/adr/`).

**Retention:** append-only. `access` events may be sampled for high-volume
stores. Mutation, lifecycle, governance, and dream_run records are never pruned.

**CLI query surface:**

```
engram history <memory_id>         # per-memory timeline
engram history --agent <id>        # all changes by a specific agent
engram history --dream <run_id>    # all changes from a dream run
engram log --type governance       # governance actions only
engram log --denied                # denied MCP calls
engram capture status              # capture summary for active session
```

#### OT.3 Daemon observability

**Health endpoint:** `http://127.0.0.1:7474/health` — unauthenticated, local
only. Returns daemon status, store state, index freshness, active dream jobs.

**Metrics endpoint:** `http://127.0.0.1:7474/metrics` — Prometheus exposition
format. Key metrics: `engram_memories_total` (by store/type), `engram_staging_backlog`,
`engram_recall_latency_p50_ms`, `engram_recall_latency_p99_ms`,
`engram_qmd_index_age_seconds`, `engram_dream_runs_total` (by status),
`engram_mcp_calls_total` (by verb/status), `engram_occ_conflicts_total`,
`engram_uptime_seconds`.

**`engram status` command:** human-readable summary of `/health` + `/metrics`.
Exit code 0 = healthy, 1 = degraded, 2 = down.

**Log format:** structured JSON to `~/.engram/logs/engramd.log`. Fields: `ts`,
`level`, `component`, `msg`, plus structured context per component. Daily
rotation, 14-day retention.

#### OT.4 Dream run audit

Every dream run produces `.dreaming/runs/drun-<run_id>.json` (git-tracked).
Mandatory fields: `run_id`, `dream_name`, `triggered_by`, `started_at`,
`completed_at`, `status`, `llm_model`, `tokens_in`, `tokens_out`,
`estimated_cost_usd`, `staging_files_considered`, `staging_files_distilled`,
`staging_files_discarded` (with per-file reason), `memories_created`,
`memories_modified` (with old/new/reason), `links_added`, `links_removed`,
`emergent_entity_proposals`, `review_queue_items`, `branch`, `commit`,
`merge_strategy`, `errors`.

Budget tracking: each run records consumed tokens against the dreaming-memory's
`budget` config. The daemon warns (log + metrics) when a dreaming memory
has consumed >80% of its daily token budget.

#### OT.5 Memory provenance fields

The frontmatter `author` field is split:

- `created_by` — immutable, set on birth, never modified.
- `last_modified_by` — updated on every write (last writer wins, frontmatter
  summary only — full history in app-log).

For memories of `origin: ingested`, frontmatter additionally carries:
- `ingest_run_id` — links to the ingest_run app-log record.

For memories distilled from staging:
- `staged_from` — list of staging file IDs that contributed to this memory.

#### OT.6 Access control audit

All denied MCP calls are logged to the app-log (`event_type: "mcp_call"`,
`status: "denied"`) with `denied_reason`. Governance deletes require a
mandatory `justification` parameter; the justification is recorded in both
the app-log and the git commit message. Denied-call counts are surfaced in
`/metrics` and visible via `engram log --denied`.

#### OT.7 Operational

**Service management:** `engram install` writes a systemd user unit (Linux) or
launchd plist (macOS) and a log rotation config. The daemon is user-level —
no root required.

**State locations documented by `engram doctor`:** a diagnostic command that
prints all state locations, checks their integrity, and reports any missing or
corrupt components.

**Schema versioning:** `.engram/schema-version` tracks the store schema version.
Migrations are explicit (`engram migrate`), never automatic. Migration events
are logged.

**Backup:** `engram backup` / `engram restore` for stores running git-off or
for point-in-time snapshots of the app-log. Primary backup for git-on stores
is the git history of the store root.

---

## 9. Open Questions — Observability Additions

These should be added to the spec's §10 Open Questions:

| ID | Question | Leaning |
|----|----------|---------|
| OQ-K | App-log format: SQLite vs JSONL for v1? | SQLite — avoids full scan; matches job queue decision (OQ-I) |
| OQ-L | Is `app-log.db` git-tracked or excluded? | git-tracked by default; document size implications |
| OQ-M | Health port configurable? Default `7474` conflict risk? | configurable in `engram.config.json`; default `7474` |
| OQ-N | Should `access` events be sampled by default, or opt-in sampling? | sampled (1-in-10) by default; configurable |
| OQ-O | `capture-fallback/` buffer: size limit? TTL? | 10MB limit, 7-day TTL; warn user when non-empty |
| OQ-P | Dream run audit records: git-track in `.dreaming/runs/` or in app-log only? | both: app-log for queries, `.dreaming/runs/` JSON for human inspection and git history |
| OQ-Q | Does `engram doctor` auto-repair or report-only? | report-only; repairs require explicit `engram repair` |

---

*End of observability & traceability audit.*
