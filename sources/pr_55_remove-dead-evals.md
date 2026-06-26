---
title: "PR 55 — remove dead planning-sequence evals.json"
type: source
created: 2026-06-26
last_updated: 2026-06-26
sources: []
tags: [source, skill-testing, cleanup, planning-sequence]
---

# PR 55 — remove dead planning-sequence evals.json

<!-- Ingested by /wiki-ingest after merge. Immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR number | #55 |
| Branch | `remove-dead-planning-sequence-evals` |
| Merged | 2026-06-26 |
| Merge SHA | `ce499d7` (change commit `7528454`) |
| JIRA ticket | — |

## Summary

Deleted `skills/planning-sequence/evals/evals.json` (23 lines). The file was an **orphan**: nothing in coderails referenced it, and coderails never invokes the tooling that could read it. It was the only `evals.json` in the repo, so its presence falsely implied coderails had a skill-testing harness — a "decoy" that cost a whole session of confusion before being traced. Deleting it removes the false signal. The eval scenarios it held are preserved in git history and in the maintainer's project memory.

## Files changed

- `skills/planning-sequence/evals/evals.json` — **deleted** (dir becomes empty → untracked by git).

## Wiki pages updated

- [[skill-testing-state_2026-06-26]] — the investigation that motivated this deletion
- [[writing-skills]] — gained a skill-testing note pointing at that investigation

## Caveats / gotchas

- The file's schema (`{skill_name, evals:[{id, prompt, expected_output, files}]}`) is skill-creator's **quality-eval** format (defined in skill-creator's `references/schemas.md`) — NOT its **trigger-eval** format (`{query, should_trigger}`, the one `run_eval.py` consumes). It was likely produced during a past skill-creator session on planning-sequence, then left behind. `(verified)`
- It was also incomplete: skill-creator's grader scores the `expectations[]` array, which this file lacked — it had only `expected_output` (a human-readable description). So even fed into skill-creator it had nothing to grade. `(verified)`
- Do NOT recreate this as a "head start" without also wiring a runner and adding `expectations[]`. A dormant eval file is worse than none — it implies infra that isn't there. See [[skill-testing-state_2026-06-26]].
