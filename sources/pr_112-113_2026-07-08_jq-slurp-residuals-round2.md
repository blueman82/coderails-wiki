---
title: "PRs #112, #113 — jq-slurp tolerant-parse residuals, round 2"
type: source
created: 2026-07-08
last_updated: 2026-07-08
sources: []
tags: [hook, jq, malformed-transcript, discipline-common, unregistered-loop-guard]
---

# PRs #112, #113 — jq-slurp tolerant-parse residuals, round 2 (2026-07-08)

Follow-up cluster closing the two residuals [[pr_86-107_2026-07-08_loop-lib-residuals|PRs #91/#107]]
explicitly left out of scope: `discipline_common.sh`'s `dc_extract_last_text`
and `unregistered_loop_guard.sh`'s own `ulg_count_dispatch_turns`. Same bug
shape as #91 in both cases — a bare `jq -s` (slurp) call aborts its entire
parse on a single malformed JSONL line, rather than dropping just that line.

## PR metadata

| PR | Title | Merged |
|---|---|---|
| #112 | dc extract tolerant parse | 2026-07-08 19:22:33Z |
| #113 | ulg count tolerant parse | 2026-07-08 19:22:14Z |

## #112: `dc_extract_last_text` (discipline_common.sh)

`dc_extract_last_text` collapsed to empty when a single malformed line
appeared in its tail window, because `jq -s` aborts on the first parse error —
identical failure shape to the pre-#91 `als_extract_last_text`. Fix: try the
existing whole-stream `jq -s` parse first (unchanged fast path, preserves every
pre-existing fixture including pretty-printed multi-line JSON used by
[[check_confidence_labels]]/[[check_verify_loop]]/[[discipline_catchup]]
tests), and fall back to a per-line tolerant parse — `jq -R 'fromjson? //
empty'` per line, then aggregate — only when the fast path fails. Mirrors
`als_extract_last_text` in `loop_state_common.sh`.

Unlike #91's `als_count_invocations`, this fix is **unconditional and silent
by design** — no `als_log`, no stderr, no reason attribution — matching
`dc_extract_last_text`'s prior contract of never distinguishing "malformed"
from "no text yet."

**Known-but-out-of-scope sibling, called out in the PR itself:**
`dc_file_count` (same file) has the identical `jq -s` fragility but was
deliberately not touched — out of scope per the work-unit manifest (exactly
`discipline_common.sh` + its test file).

Verification: RED (new malformed-line test failed pre-fix) → GREEN (fix
applied, new + all pre-existing tests in the file pass) → full suite
`hooks/scripts/tests/run_all.sh` 38/38 suites, 1108 assertions, 0 failures.
Diff scope confirmed to match exactly the two manifest files.

## #113: `ulg_count_dispatch_turns` (unregistered_loop_guard.sh)

`ulg_count_dispatch_turns` now mirrors `als_count_invocations`'s two-stage
tolerant parse (the #91 pattern) instead of a bare `jq -s` slurp: a single
malformed transcript line no longer collapses the whole dispatch-turn count to
0. Two distinct outcomes, matching #91/#107's failure-attribution split:

- **Benign partial skip** (some lines parse, at least one malformed) — the
  valid count is recovered, `ULG_PARSE_REASON` is left empty, and the nudge
  logic proceeds normally against the recovered count.
- **Total parse loss** (every line malformed) — count reports `0` **with** a
  non-empty `ULG_PARSE_REASON`, so [[unregistered_loop_guard]]'s caller can
  distinguish "genuinely 0 dispatch turns" from "untrustworthy input, suppress
  the nudge" — the same `all_lines_malformed`-style distinction #107 introduced
  in the shared lib, now applied to this hook's own local parse.

Companion commit `919d3c9` (review follow-up, same day) fixed an `als`
comment-rot spot and added two more test cases from the PR review: a
malformed-line-**first** ordering case (proves the stage-1 parse is
position-independent — a bad line at the start doesn't behave differently from
one buried in the middle) and a blank/whitespace-lines-mixed-with-a-malformed-line
case (proves the total non-blank line count used to distinguish "some parsed"
from "all malformed" correctly excludes blank lines from that discriminator).

## Closes the round-1 residuals

This pair closes both items [[pr_86-107_2026-07-08_loop-lib-residuals|PRs
#91/#107]]'s "Known residuals, explicitly out of scope" section named by path:
`discipline_common.sh`'s `dc_extract_last_text` (closed by #112) and
`unregistered_loop_guard.sh`'s own `ulg_count_dispatch_turns` slurp (closed by
#113). `dc_file_count` remains open — flagged again, explicitly, in #112's own
description as still out of scope.

## Verification

- (verified) PR bodies, merge timestamps, and title text from
  `gh pr view 112 --json title,body,mergedAt,number` and
  `gh pr view 113 --json title,body,mergedAt,number`.
- (verified) Post-fix source for both functions read directly from
  `git show origin/main:hooks/scripts/lib/discipline_common.sh` and
  `git show origin/main:hooks/scripts/unregistered_loop_guard.sh`.
- (inferred) The companion commit `919d3c9` test-addition detail is drawn from
  its commit message (`git log`), not a separate `gh pr view` body — it landed
  as part of the same review cycle rather than its own PR.

## See also

- [[pr_86-107_2026-07-08_loop-lib-residuals]] — round 1: the shared-lib fix
  (`loop_state_common.sh`) and the residuals this round closes
- [[unregistered_loop_guard]] — hook whose own dispatch-turn counter is fixed by #113
- [[discipline_catchup]], [[check_confidence_labels]], [[check_verify_loop]] — consumers of `discipline_common.sh`'s `dc_extract_last_text`
