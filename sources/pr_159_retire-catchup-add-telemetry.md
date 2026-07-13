---
title: "PR 159 — Retire discipline_catchup; add event= telemetry; version lockstep repair"
type: source
created: 2026-07-13
last_updated: 2026-07-13
sources: []
tags: [hook, discipline_catchup, retirement, telemetry, versioning, hooks-json]
---

# PR 159 — Retire discipline_catchup; add event= telemetry; version lockstep repair

## PR metadata

| Field | Value |
|---|---|
| PR number | #159 |
| Branch | `hooks/retire-catchup-telemetry` |
| Merged | 2026-07-13 |
| Merge SHA | `1ac216658782f8a5d322875b19ca9147073d9cf4` |
| JIRA ticket | — |

## Summary

Three independent changes landed in one PR:

1. **`discipline_catchup.sh` retired clean-break.** Both the script
   (`hooks/scripts/discipline_catchup.sh`) and its test
   (`hooks/scripts/tests/discipline_catchup.test.sh`) were deleted; `hooks/hooks.json`'s
   `UserPromptSubmit` array is now a single entry (`inject_context.sh`) instead of two.
   (verified: `hooks/hooks.json` current state — `discipline_catchup.sh` absent from
   both the file listing and the hooks array; `git log --diff-filter=D` confirms the
   deletion commit)

   Justification (from the merge commit and prior session record): a flat
   23–26% first-attempt miss rate on the discipline rules had held steady since
   block-mode was introduced 2026-05-05 — the warn-mode `UserPromptSubmit`
   catchup nudge measurably added nothing on top of the two block-mode Stop
   hooks (`check_confidence_labels.sh`, `check_verify_loop.sh`). This is a
   retirement based on a null result over ~2 months of live data, not a
   design reversal.

2. **`event=<hook_event_name>` telemetry added to every discipline-hook log
   line.** `check_confidence_labels.sh` and `check_verify_loop.sh` both now
   log which `hook_event_name` (`Stop` vs `SubagentStop`) produced each
   `discipline.log` line. Before this PR, a log line gave no way to
   distinguish main-agent Stop events from subagent worker events after the
   fact — segmentation was structurally impossible without re-deriving it from
   session context. (verified: check_verify_loop.sh:69, 89 read
   `hook_event_name` into `hook_event`, threaded into every `log_line` call;
   same pattern in check_confidence_labels.sh)

3. **Enriched `[discipline-block]` message + version lockstep repair.**
   `check_confidence_labels.sh`'s block-message text was rewritten to match
   `check_verify_loop.sh`'s more actionable style (names the rule, gives a
   concrete example, tells the model what to do next) rather than a bare
   "labels missing" statement. Separately, PR #155 (concurrent session,
   referenced not duplicated) had bumped only `plugin.json`'s version,
   breaking the `plugin.json`/`marketplace.json` lockstep the repo's own test
   suite depends on. This PR repairs it — both files now read `1.1.5`.
   (verified: `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json`
   both show `"version": "1.1.5"`)

## Follow-up hardening (same-day, commit 109987b, not a separate PR)

Three review findings closed against this PR's own test additions, on `main`
directly:

- `hooks_json_timeout_floor.test.sh`'s `UserPromptSubmit` guard previously only
  checked `[0].hooks` — a hook re-added as a second matcher object (`[1]`) would
  have passed unnoticed. Now also asserts the matcher array itself has length 1
  (closing the exact shape this PR produced: one matcher, one hook).
- The version-lockstep check in the same test passed vacuously when both `jq`
  reads degraded identically (both files missing, or both missing the `version`
  key, yield equal empty-string/`"null"` pairs and compare equal) — now asserts
  each side is non-empty and not `"null"` before comparing.
- `check_confidence_labels.test.sh`'s blocked-path case previously asserted only
  `exit 2`, discarding stderr entirely (zero coverage of the actual message
  content). Added a `run_stderr` helper and a case asserting the message
  contains both `"[discipline-block]"` and `"Rule (CLAUDE.md)"`.

## Files changed

- `hooks/scripts/discipline_catchup.sh` — deleted
- `hooks/scripts/tests/discipline_catchup.test.sh` — deleted
- `hooks/hooks.json` — `UserPromptSubmit` down to `inject_context.sh` only
- `hooks/scripts/check_confidence_labels.sh` — `event=` telemetry, rewritten block message
- `hooks/scripts/check_verify_loop.sh` — `event=` telemetry
- `hooks/scripts/lib/discipline_common.sh` — minor support changes for telemetry
- `.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json` — both `1.1.5`
- `hooks/scripts/tests/check_confidence_labels.test.sh`,
  `hooks/scripts/tests/check_verify_loop.test.sh`,
  `hooks/scripts/tests/hooks_json_timeout_floor.test.sh` — new/hardened tests

## Wiki pages updated

- [[discipline_catchup]] — marked retired/orphan (hook no longer exists)
- [[discipline-loop]] — Stop/UserPromptSubmit composition, retirement note, jq-slurp closure
- [[check_verify_loop]] — `event=` telemetry field added to logging section
- [[check_confidence_labels]] — `event=` telemetry, block-message rewrite
- [[install-and-cache-trap]] — version claim corrected (was: pinned at 1.0.0; actual: semver, 1.1.5 live)

## Caveats / gotchas

- This is a **clean-break retirement**, not a deprecation window — there is no
  compatibility shim and no grace period. Any external reference (docs, memory,
  wiki pages) describing `discipline_catchup.sh` as live is now describing a
  deleted file.
- The null-result justification (23–26% flat miss rate) is itself only as good
  as the sample the miss-rate figure was drawn from — the PR/commit record does
  not cite the measurement methodology in detail; treat the number as the stated
  reason, not independently re-derived here.
- The `event=` field is additive to the existing log line format — greppable,
  does not change field order for existing consumers that parse by field name
  (`key=value`), but any consumer doing positional parsing would need to adapt.
- Global `~/.claude/settings.json` separately had 3 duplicate discipline hooks
  (`discipline_catchup`, `check_confidence_labels_v2`, `check_did_not_verify_v2`)
  deregistered from `UserPromptSubmit` — that was **not part of this PR** (global
  config isn't repo-tracked) but was done the same session as a related cleanup;
  see [[discipline-loop]] for the note. Root cause there was independent: regex
  drift between the global copies (required closing paren) and the plugin's more
  lenient canonical regex, not the same null-result reasoning that retired the
  plugin-tracked `discipline_catchup.sh`.
