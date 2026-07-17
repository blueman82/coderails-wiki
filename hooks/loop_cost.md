---
title: "Lib: loop_cost.sh (dc_mine_token_usage)"
type: hook
created: 2026-07-17
last_updated: 2026-07-17
sources:
  - hooks/scripts/lib/loop_cost.sh
  - sources/pr_184_185_186_loop-cost-tracking.md
  - sources/pr_204_cost-reporter.md
  - sources/pr_216_217_safe-routes-and-cost-miner-diagnostics.md
tags: [lib, agentic-loop, cost-tracking, fail-open, diagnostics, loop-stall-guard]
---

# Lib: loop_cost.sh (`dc_mine_token_usage`)

A **sourced library**, not a hook script wired in `hooks/hooks.json` — it has
no direct lifecycle-event entry of its own. It's called from
`loop_stall_guard.sh`'s `complete`-branch teardown sequence (see
[[loop_stall_guard]]), which is why it lives alongside the other hook-family
pages rather than under a `design/` or `skills/` heading. First shipped in
PR #186 ([[pr_184_185_186_loop-cost-tracking]]); its fail-open bails gained
distinct diagnostic messages in PR #217
([[pr_216_217_safe-routes-and-cost-miner-diagnostics]]).

Source: `hooks/scripts/lib/loop_cost.sh`

## What it does

`dc_mine_token_usage <session_id>` mines per-model token usage and estimated
USD cost for one agentic-loop orchestrator session **plus every worker
(subagent) transcript it recursively spawned**, deduplicated by
`message.id`. Output is a single JSON object on stdout, priced via a dated
`hooks/scripts/lib/model_prices.json`. The miner's output becomes `retro.cost`
at teardown (Phase 13), consumed by [[pr_204_cost-reporter|the cost
reporter]] on every `LOOP-STOP: complete`.

## The fail-open contract

`dc_mine_token_usage` **fail-opens to `printf '{}'; return 0` on every error
path** — missing `jq`, empty session id, missing prices file, no transcript
found, a mining or pricing `jq` pipeline producing empty or invalid output.
This is deliberate and documented: the miner **must never block loop
teardown**. `docs/REFERENCE.md`'s hook matrix states the reason directly — a
blocking cost reporter would deadlock an already-finished loop.
[[pr_204_cost-reporter]]'s own design note is explicit that this fail-open
posture should **not** be "fixed" into the repo's house fail-closed style;
doing so would reintroduce the exact deadlock the design avoids. Mirrors the
same fail-open idiom `dc_mine_hook_blocks` in `discipline_common.sh` uses.

## The problem this created, and PR #217's fix

An identical `{}` collapses every distinct cause into one indistinguishable
value. That silence let **four consecutive loops ship four different, all
wrong, root causes** for the same `{}` symptom: multi-session blind spot →
unpriced models → cwd sensitivity → "empty `BASH_SOURCE` in both shells." All
four were falsified before the real cause was found — a zsh-only
`${BASH_SOURCE[0]}` bug (empty inside a function under zsh, so `dirname ''`
resolves to `.`, so a relative `model_prices.json` path fails its `[ -f ]`
check), fixed separately in PR #205.

PR #217 ([[pr_216_217_safe-routes-and-cost-miner-diagnostics]]) closed the
silence itself, not just that one bug: **all 7 `printf '{}'` fail-open sites**
now emit a distinct, cause-naming `echo "loop_cost: ..." >&2` line
immediately before the bail:

| Line | Bail condition | stderr message |
|---|---|---|
| 57 | `jq` not on `PATH` | `loop_cost: jq not found on PATH` |
| 58 | empty session id argument | `loop_cost: empty session id` |
| 59 | prices file missing (PR #205's original fix) | `loop_cost: prices file not found at $prices_file` |
| 100 | no transcript resolves for the session | `loop_cost: no transcript found for session $session under $projects_dir` |
| 163 | mining `jq` pipeline produced no output | `loop_cost: mining produced no output` |
| 164 | mining `jq` output isn't valid JSON | `loop_cost: mining produced invalid JSON` |
| 217 | pricing `jq` pipeline produced no output | `loop_cost: pricing produced no output` |

The `{}`/exit-0 contract is unchanged by this fix — every path still returns
`{}` on stdout, exit 0. The fix is **diagnostic-only**: it names the stage
that failed without changing what the caller receives.

**Deliberately not built:** capturing the suppressed `jq` stderr on lines
163/164/217 (those invocations run with `2>/dev/null`, discarding the
underlying jq error text, not just leaving it unlogged). Both workers who
built PR #217 independently agreed this is out of scope — the stage-naming
message already disambiguates which of the three mining/pricing stages
failed, and restructuring the capture to tee stderr without leaking it into
stdout is real contract risk (a `2>&1` merge or `tee` could corrupt the `{}`
JSON on stdout) for marginal additional diagnostic value. Recorded as a
possible future enhancement.

## Verification

- 71 ok / 0 FAIL on `hooks/scripts/tests/loop_cost.test.sh` (re-verified live
  during the PR #217 ingest).
- Live-verified under zsh on merged `origin/main`: a bogus session id
  produces `stdout=[{}]`, `exit=0`, `stderr=[loop_cost: no transcript found
  for session bogus-xyz under /Users/harrison/.claude/projects]`.
- A **pairwise-distinctness test** asserts all 7 stderr messages are unique —
  no two bails share wording — closing the same "shared needle"
  discriminating-check gap [[destructive_bash_gate]]'s route assertions hit
  in the same cluster (see
  [[pr_216_217_safe-routes-and-cost-miner-diagnostics]] for the parallel).

## Related

- [[pr_184_185_186_loop-cost-tracking]] — original build, `retro.json`
  `schema_version` 2, dashboard cost tiles
- [[pr_204_cost-reporter]] — the reporter that reads `retro.cost` and prints
  it to the human via `systemMessage`; explicitly declines to fail-close this
  miner
- [[pr_216_217_safe-routes-and-cost-miner-diagnostics]] — the diagnostic-line
  fix this page mostly documents
- [[loop_stall_guard]] — hosts the `complete`-branch call site and the wider
  teardown gate sequence
