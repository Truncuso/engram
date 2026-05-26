---
name: wp06-open-questions
title: Open Questions — Capture + CaptureIntake + staging
type: wp-open-questions
wp: wp06-capture-captureintake-staging
updated: 2026-05-26
---
<!-- Template: WP-folder OPEN_QUESTIONS v2 (frontmatter-first) -->

# Open Questions — WP06: Capture + CaptureIntake + staging

> A WP cannot reach `stage: hardened` while any question has `status: open`.

> Canonical register is the plan-level `OPEN_QUESTIONS.md`; rows here mirror the
> ones that block this WP. Resolve there; this is the WP-local view.

## Active Questions

| ID | Question | Context | Blocks | Status |
|----|----------|---------|--------|--------|
| OQ-01 | Privacy-filter **match**: strip the secret and pass the rest to staging (A), or drop the whole observation (B)? Only (A) satisfies SC-4. | §6.1 "fail-closed ⇒ drop" likely covers filter *errors*, not *matches* | WP06 phase-1 | open |
| OQ-05 | Entropy-layer threshold (40 chars): false-positive rate on normal payloads (base64, stack traces, SQL, minified JS)? | affects default config + SC-4 test | WP06 phase-1 | open |

## Resolution Log

| ID | Question | Resolution | Date |
|----|----------|------------|------|
|  |
