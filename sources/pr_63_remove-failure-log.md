---
title: "PR #63 — chore: remove orphaned failure_log.md and its plumbing"
type: source
created: 2026-06-26
last_updated: 2026-06-26
sources: []
tags: [source, failure-log, deletion, calibration-ritual, discipline]
---

# PR #63 — remove orphaned failure_log.md

<!-- Ingested by /wiki-ingest after merge. This is an immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR number | #63 |
| Branch | `chore/remove-orphaned-failure-log` |
| Merged | 2026-06-26 |
| Merge SHA | `18392fb` |
| JIRA ticket | — |

## Summary

Deleted the seeded `templates/failure_log.md` and all its live plumbing. The file was the data input for claude-guardrails' weekly **calibration ritual** — a feature the plugin merge explicitly removed ("no launchd, no calibration ritual", README). Its consumer was gone, no hook/skill/command read or wrote it, and two months of use produced zero entries. It was the orphaned input of a deleted pipeline. (verified — grep over hooks/skills/commands found no reader/writer; live `~/.claude/failure_log.md` had 3 seed rows, none added since 2026-05-01)

## Files changed (net diff: 5 files, +7 / −52)

- `templates/failure_log.md` — deleted (the only file in `templates/`; the directory goes with it).
- `install.sh` — removed the failure-log seeding section; renumbered the trailing "Claude Code steps" section 8→7. `bash -n` clean.
- `uninstall.sh` — dropped the two `failure_log` references; kept the `discipline.log` + memory preservation lines.
- `CLAUDE.md` — removed the `templates/failure_log.md` row from the architecture map and from both install/uninstall invariants.
- `INSTALLATION.md` — removed the two mentions (seeding line + preservation line).

## Durable points

**The empty log was a demand signal, not a friction signal.** (inferred) An automation plan was considered (a `/failure` capture command + `/failure-review` recurrence consumer) and **declined**. The reasoning: a disciplined, willing maintainer had a prose "add a row" instruction for two months and added zero rows. Zero entries from a willing user indicates the practice itself was not reached for — lower-friction tooling makes an unwanted act cheaper, it does not make it wanted. Per the codebase's YAGNI/reactive discipline (and Ng's "build evals when recurrence *demands* it"), an empty log is not recurrence you failed to capture. See the caught-vs-escaped distinction below.

**Caught vs. escaped failures are different classes — only one is hook-capturable.** (verified — from the file's own header vs. what `discipline.log` records) Hooks already log *caught* failures (e.g. `blocked=1` lines in `discipline.log`); these never shipped wrong. `failure_log.md`'s header said it was for *escaped* failures — "output that was wrong/misleading/incomplete" that passed every hook. An escaped failure cannot be auto-captured by a hook by construction: if a hook could detect it, it would not have escaped. So no fully-automated producer was ever viable for the class the file actually targeted.

**Kept deliberately (not dangling):** `docs/coderails-review.md` (a historical analysis record, not live docs) still references the file by name, and `README.md`'s "no calibration ritual" line stays — it is accurate history and *explains* why the file was orphaned. The maintainer's live `~/.claude/failure_log.md` is untouched (install skips-if-exists, uninstall preserves it).

## Wiki pages updated

- [[discipline-loop]] — the "Founding Thesis" section cited `templates/failure_log.md` line 26 for the empirical failure that motivated block-mode. The quote is preserved verbatim in the wiki (no knowledge lost); the dead pointer was re-attributed to note the file's removal in this PR and that the record now lives only in the maintainer's live `~/.claude/failure_log.md` (user data, not a repo artifact). The page's `sources:` list dropped `templates/failure_log.md` for this source.

## Caveats / gotchas

The founding-thesis quote that motivated the whole block-mode discipline loop originated in `templates/failure_log.md` — so deleting the file removed the *citation target*, not the *content*. The content survives in [[discipline-loop]]. Anyone tracing the discipline loop's origin should look there, not at the (now absent) template.
