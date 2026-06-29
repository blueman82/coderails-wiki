---
title: "PR #79 — docs: fix sync-docs drift (REFERENCE.md skills, README truncate, inject_context, test_gate)"
type: source
created: 2026-06-29
last_updated: 2026-06-29
sources: []
tags: [docs, drift, sync-docs, reference, inject_context, test_gate, engineering-principles]
---

# PR #79 — docs: fix sync-docs drift

<!-- Ingested by /wiki-ingest after merge. This is an immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR number | #79 |
| Branch | `docs/fix-sync-docs-drift` (inferred) |
| Merged | 2026-06-29 |
| Merge SHA | `0bcb44e` |
| JIRA ticket | — |

## Summary

Fixed 4 pre-existing doc-drift findings surfaced by `/sync-docs` (run as the agentic-loop Phase 9 step added in PR #77). These are stale docs — not from a recent code change. The source of truth (scripts, SKILL.md files) was already correct; the docs had not kept up. (verified: PR #79 body)

## Finding F1 (High) — REFERENCE.md missing engineering-principles skills

`docs/REFERENCE.md` was missing all 4 engineering-principles skills (`engineering-principles`, `engineering-principles-go`, `engineering-principles-python`, `engineering-principles-ts`) and falsely claimed to be a "Complete catalogue". Root cause: the skills were added in PR #54 but REFERENCE.md was not updated at that time.

Fix:
- Added an "Engineering principles skills" section to REFERENCE.md with accurate descriptions drawn from each SKILL.md frontmatter
- Reconciled skill count from 23 → 27
- Added the group to the REFERENCE.md TOC
- Updated the opening claim to "Catalogue of every coderails component (27 skills, plus hooks, commands, scripts)" (verified: PR #79 body)

## Finding F2 (Medium) — README listed bare `truncate` in destructive_bash_gate blocklist

`README.md` listed bare `truncate` in the `destructive_bash_gate` blocklist. The actual gate only blocks the `-s`/`--size` form — `AGENTS.md` and `REFERENCE.md` were already correct; only `README.md` had the bare form. (verified: PR #79 body)

Fix: changed `README.md` to match the other two docs and the actual gate logic.

## Finding F3 (Low) — inject_context.sh first-prompt discipline reminder omitted

`README.md` and `AGENTS.md` described `inject_context.sh` as only prepending `[ctx]` (cwd, branch, date), omitting that the script also appends the **discipline reminder** on the **first prompt of a session**. `REFERENCE.md` already had this correct. (verified: PR #79 body, cross-referenced hooks/scripts/inject_context.sh lines 12-16)

Fix: extended the description of `inject_context.sh` in both `README.md` and `AGENTS.md` to note the first-prompt discipline reminder.

## Finding F4 (Low) — test_gate.sh missing from "five core gate scripts" inline-if list

`AGENTS.md` "Hook script conventions" said "the other **four** scripts use inline if-blocks" — but `test_gate.sh` also uses inline if-blocks and was not named. The gate-pattern convention applies to five core gate scripts, not four. (verified: PR #79 body, cross-referenced hooks/scripts/test_gate.sh)

Fix: updated `AGENTS.md` to "**five** core gate scripts" (`check_verify_loop.sh`, `check_confidence_labels.sh`, `no_edit_on_main.sh`, `destructive_bash_gate.sh`, `test_gate.sh`), adding a clarifying note that the support/context scripts (`inject_context.sh`, `inject_bootstrap.sh`, `discipline_catchup.sh`) also use inline blocks but are not part of the gate-pattern convention.

## Files changed

| File | Change |
|---|---|
| `docs/REFERENCE.md` | Added engineering-principles skills section (4 skills), fixed count 23→27, fixed TOC, fixed "Complete catalogue" claim |
| `README.md` | Fixed bare `truncate` → `truncate -s/--size`; added first-prompt discipline reminder to inject_context.sh description |
| `AGENTS.md` | Added first-prompt discipline reminder to inject_context.sh description; fixed "four" → "five" core gate scripts with test_gate.sh added |

## Why this matters

These were stale docs from previous ingests. The `/sync-docs` step (added to agentic-loop Phase 9 in PR #77) surfaced all four. They had no runtime impact (the gate logic, scripts, and skill definitions were all correct) but created a misleading picture of the tool count (23 vs 27) and the behaviour of `inject_context.sh` and the gate-script convention.

## Context

- F1 traces to PR #54 (engineering-principles vendoring, 2026-06-26) — REFERENCE.md was not updated at that time; see [[pr_54_engineering-principles-vendoring]]
- F2 traces to PR #59 (destructive_bash_gate extension to `truncate -s/--size`, part of [[pr_57-62_subagent-enforcement-gate-hardening]]) — README was not updated
- F3/F4 are pre-existing gaps with no single originating PR

## Wiki pages updated

- [[inject_context]] — F3: clarified that script also injects discipline reminder on first prompt
- [[test_gate]] — F4: added note that test_gate.sh uses inline if-blocks (part of five core gate scripts)
- [[discipline-loop]] — F4: updated "gate-pattern convention" count reference
