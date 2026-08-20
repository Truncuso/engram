---
name: meta-harness-memory-ownership
status: idea
created: 2026-08-20
references:
  - type: file
    locator: "/home/cunger/10_Projects/Agentic AI/Workflows/plans/meta-harness/wayfind/ticket-44-agent-memory-ownership.md"
    note: the owning research ticket
  - type: file
    locator: "/home/cunger/10_Projects/Agentic AI/Workflows/plans/meta-harness/wayfind/findings/vision-crosscheck-2026-08-20.md"
    note: Q5 — user's memory vision
  - type: file
    locator: "~/.claude/plans/agentic-memory/overview.md"
    note: the other store candidate
---

# Relation — engram in the meta-harness memory question

Parked 2026-08-20 from the meta-harness vision cross-check lap.

The meta-harness effort (Workflows repo, `plans/meta-harness/wayfind/`) is
researching shared agent-memory ownership (**ticket-44**): engram vs the
agentic-memory plan as the STORE, and meta-harness ticket-21 as the
cross-harness TRANSPORT (how CC/dsh/codex agents mount memory — symlink, MCP,
context injection). Engram's MCP surface makes it a natural swappable plugin
behind a meta-harness transport seam; whether engram supersedes the
agentic-memory plan is part of the same research.

Before starting engram's bottom-up build, read ticket-44 findings — the
ownership outcome may shape engram's capture seam (meta-harness egress
proxy/event log could become a capture source) and its plugin posture.

## Outcome (2026-08-20, ticket-44 resolved, user-confirmed)

Engram = **design corpus**, not a build target: agentic-memory superseded it
(June 2026); its ideas (SPEC §3.4 scope/visibility, §6.3 MCP verbs, §15.5
multi-agent install) get MINED into the standalone memory repo that the
agentic-memory plan extracts to (base-map t06 own-repo model), under a new
name — "engram" rejected as publish name for collision. Repo stays as-is,
not deleted.
