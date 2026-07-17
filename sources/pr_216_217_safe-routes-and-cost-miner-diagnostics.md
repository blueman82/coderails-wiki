---
title: "PR #216 + #217 — safe routes for every blockable pattern, and cause-naming stderr on every loop_cost fail-open bail"
type: source
created: 2026-07-17
last_updated: 2026-07-17
sources:
  - hooks/scripts/destructive_bash_gate.sh
  - hooks/scripts/tests/destructive_bash_gate.test.sh
  - hooks/scripts/lib/loop_cost.sh
  - hooks/scripts/tests/loop_cost.test.sh
tags: [hook, destructive-bash-gate, loop-cost, fail-open, diagnostics, discriminating-check, safe-route, cost-miner]
---

# PR #216 + #217 — a blocked or failed thing must tell you what to do next

Two independently merged PRs, same day, one theme: **a blocked or failed thing
must tell you what to do next instead of leaving you to guess.** PR #216 gives
every blockable [[destructive_bash_gate]] pattern a named safe route. PR #217
gives every fail-open bail in the loop-cost miner (`loop_cost.sh`,
`dc_mine_token_usage`) a distinct, cause-naming stderr line. Both are
message-only changes — zero changes to blocking/matching logic in either file.

## PR #216 — safe route for every blockable pattern

Merged `be84433362cdb6c9aaa3f07b1118dc6842a5787de`, 2026-07-17T12:46:07Z
(verified: `gh pr view 216 --json mergedAt,mergeCommit`, matches `git show
origin/main:hooks/scripts/destructive_bash_gate.sh` live).

Before this PR, [[destructive_bash_gate]]'s `deny()` mapped a matched pattern
to a `route` string, but only 3 of ~13 blockable patterns had one (`git reset
--hard`, `rm -rf`, `git push --force`, added PR #203 — see
[[pr_201_202_203_routine-followups]]). Every other pattern fell through to a
bare generic fallback: "No specific safe route is recorded... add a
settings.json permission rule." Route arms went 4 → 14 (13 real patterns + the
generic fallback kept for anything genuinely unmapped). Route assertions in
the test suite went 0 → 13.

### No route is fabricated — the honest-disclosure principle

Six patterns genuinely have **no safe alternative**: `find -delete`,
`truncate -s`/`--size`, `shred`, `DROP TABLE`/`DATABASE`/`SCHEMA`, `TRUNCATE
TABLE`, `dd if=`, `mkfs.`. Each of these routes says so explicitly — "there is
no safe equivalent" — and names the `settings.json` opt-in permission-rule
path as the only way forward, rather than inventing a plausible-sounding
alternative that doesn't exist. The `dd` route, for instance, deliberately
refuses to teach `dd` itself more safely; it only tells the caller how to
double-check the target and how to opt in if the command really is intended.
**A fabricated safe route was judged worse than the generic fallback** — the
honest disclosure is a real answer, an invented one sends someone down a false
path. See the "Lessons" section below.

Two routes name a genuine alternative that the gate itself already permits:

| Pattern | Route |
|---|---|
| `git clean -f`/`--force`/`-fd`/etc. | `git clean -n` (dry-run/preview) or `git clean -i` (interactive) — both already ALLOWed by this same gate; only the force forms are blocked |
| `chmod -R 777` | `chmod -R u+rwX,go+rX <path>` — owner rw(+x on dirs/already-executable files), group/other read, without world-writable/world-executable |

The full 13-pattern route table now lives on [[destructive_bash_gate]]'s
"Deny messages name a concrete safe route" section (updated by this PR — see
below).

### Sync enforcement: a TEST, not a scheduled routine — the resolved design fork

The build had to decide how to keep the route table from silently rotting as
new patterns are added later. Two candidate mechanisms were considered:

- **A scheduled routine** (like [[docs-sync]] or [[loop-retro-promotion]])
  that periodically diffs the gate's pattern set against a recorded baseline
  and reports drift.
- **A test in the existing suite**, run on every PR touching the file.

**Resolved: a test, not a routine.** Rationale: a test fails at the instant
someone adds pattern #14 without a route arm — CI-red on the PR that
introduces the gap. A scheduled routine reports the drift days later, after a
routeless block has already shipped and been hit by a real user or agent. The
routine model is right for *slow* drift (doc staleness); this is *instant*
drift (a code change), so the enforcement needs to be as fast as the change
that causes it.

Two independent mechanisms were built, because a behavioural test alone only
proves *today's* pattern list has routes — it says nothing about a *future*
pattern added without one:

1. **Per-pattern route assertions** via a new `assert_specific_route` test
   helper (`hooks/scripts/tests/destructive_bash_gate.test.sh:82-93`) that
   drives the REAL gate (not a mock) and fails if the deny reason contains the
   generic "no specific safe route" fallback text, AND fails unless it
   contains pattern-specific needle strings unique to that route's own
   wording. 13 such assertions now exist, one per pattern.
2. **A source-drift tripwire** (`destructive_bash_gate.test.sh:854+`,
   "Deliverable B") comparing the gate's actual blockable set — the `deny
   "..."` literal call sites plus the monolithic `pattern=` ERE line — against
   two committed `EXPECTED_*` string snapshots. Any change to either (a new
   pattern added, an existing one removed or reworded) fails the tripwire with
   a message naming exactly what to do: add a route case, then update the
   snapshot to match. This catches the case the per-pattern assertions can't:
   a *brand-new* pattern with no assertion written for it yet.

### Two discriminating-check defects found and fixed before merge

A discriminating check is one that can both PASS and FAIL depending on input —
a check that only ever passes isn't testing anything. Two defects of exactly
this class were found in the first draft of the route assertions, both via
adversarial review, both fixed pre-merge:

1. **Shared needles across routes.** The six honest ("no safe equivalent")
   routes initially all asserted against the same generic needles
   (`"no safe equivalent"`, `"settings.json"`) — common to all six. A
   copy-paste-swapped route body (e.g. `dd`'s route text pasted into `shred`'s
   case arm) would still contain both generic needles and pass every
   assertion. Proven by deliberately swapping `dd`'s route text for `shred`'s
   and confirming the swapped test still went green. **Fixed** by adding
   route-specific needles unique to each pattern's own wording (see the
   comment block above `assert_specific_route` calls in the test file —
   `"raw bytes"`/`"of= target"` for `dd`, `"reformats a filesystem"` for
   `mkfs.`, `"unrecoverable"`/`"securely wipe"` for `shred`, etc.) so a
   swapped-in wrong route fails on the pattern-specific needle even though the
   shared generic needles still match.
2. **Bare two-char substring needles.** The `git clean` route check originally
   grepped bare `-n` and `-i` as its needles for "names the dry-run/interactive
   route." These are two-character substrings that match incidentally inside
   unrelated words (a reviewer passed a fabricated route mentioning
   `"manual-n"` and `"4-i"` and the check went green — neither substring has
   anything to do with `git clean -n`/`-i`). **Fixed** by requiring the full
   literal strings `"git clean -n"` and `"git clean -i"` plus the distinctive
   prose `"dry-run"` and `"interactive prompt"` — see the comment directly
   above the `git clean` `assert_specific_route` call.

Both fixes are visible directly in the merged test file's comments, which
record the incident that motivated each fix rather than just the fixed code.

### Verification

- **Message-only change, confirmed by inspection and by test.** Zero changes
  to any matching regex, condition, or control-flow branch — verified against
  the diff and by a security review of the PR. The paired
  behavioural test suite asserts the same DENY/ALLOW verdict set as before
  PR #203 already established this invariant; PR #216 only adds to the
  *reason string*, never the *verdict*.
- **Full suite green under the machine's real bash 3.2.57**
  (`hooks/scripts/tests/destructive_bash_gate.test.sh`), including every
  historical bypass-regression case from the five-PR 2026-07-08 hardening arc
  ([[pr_69_2026-07-08_substitution-bypass-audit]] etc.) — re-run live during
  this ingest: **202 ok, 0 FAIL.**

## PR #217 — cause-naming stderr on every loop_cost fail-open bail

Merged `400178f384b17fbb108c50278dc2391efbafc0be`, 2026-07-17T12:57:29Z
(verified: `gh pr view 217 --json mergedAt,mergeCommit`, matches
`hooks/scripts/lib/loop_cost.sh` on `origin/main`).

`dc_mine_token_usage` (`hooks/scripts/lib/loop_cost.sh`) mines per-model
token/USD cost for a completed agentic loop, for `retro.json`'s `cost` field
(see [[pr_184_185_186_loop-cost-tracking]]). By documented contract, it
**fail-opens to `printf '{}'; return 0` on every error path** — it must never
block loop teardown (this is the same fail-open idiom [[pr_204_cost-reporter]]
explicitly declines to "fix" into the house fail-closed style, because a
blocking cost reporter would deadlock an already-finished loop).

### The problem this PR closes: an identical `{}` for every cause

The contract is correct, but its silence was costly. `docs/REFERENCE.md`'s
hook matrix documents the reporter as never blocking "because the cost miner
fails open to `{}` by contract" — true, but before this PR the `{}` gave no
signal about *why* it fired. **That silence let FOUR consecutive loops ship
FOUR different, all-wrong root causes for the same `{}` symptom:**
multi-session blind spot → unpriced models → cwd sensitivity → "empty
`BASH_SOURCE` in both shells." All four were falsified. The real cause was a
zsh-only `${BASH_SOURCE[0]}` bug (empty inside a function under zsh, so
`dirname ''` resolves to `.`, so a relative `./model_prices.json` path fails
the `[ -f ]` check) — fixed separately in PR #205.

### The fix: 6 new distinct stderr lines (was 1)

All 7 `printf '{}'` fail-open sites in `loop_cost.sh` now emit a distinct,
cause-naming `echo "loop_cost: ..." >&2` line immediately before the bail.
Line 59 (the prices-file message) already had one from PR #205; this PR adds
the other six:

| Line | Bail condition | stderr message |
|---|---|---|
| 57 | `jq` not on `PATH` | `loop_cost: jq not found on PATH` |
| 58 | empty session id argument | `loop_cost: empty session id` |
| 59 | *(pre-existing, PR #205)* | `loop_cost: prices file not found at $prices_file` |
| 100 | no transcript file resolves for the session | `loop_cost: no transcript found for session $session under $projects_dir` |
| 163 | the mining jq pipeline produced no output | `loop_cost: mining produced no output` |
| 164 | the mining jq pipeline's output isn't valid JSON | `loop_cost: mining produced invalid JSON` |
| 217 | the pricing jq pipeline produced no output | `loop_cost: pricing produced no output` |

The `{}`/exit-0 contract is preserved **exactly** — every bail still returns
`{}` on stdout and exit 0. The fix is diagnostic-only: it adds a stage-naming
stderr line, nothing else.

### Scope: 3 named in the prompt, 6 found and fixed

The authorising prompt named three bails to fix. Reading the source directly
found seven `printf '{}'` sites total, one (line 59) already carrying a
message from PR #205, leaving six unaddressed. Scope was expanded from 3 to
all 6 remaining sites and **flagged as an expansion, not silently widened** —
worth recording because two of the six (lines 163/164, and 217) mask real
`jq` failures behind `2>/dev/null`, which is the *exact* "identical `{}`, no
cause" symptom class this PR exists to kill; leaving them unmessaged would
have been an inconsistent, partial fix of the very problem being solved.

### Deliberate non-goal: not capturing suppressed jq stderr

Lines 163/164/217's `jq` invocations run with `2>/dev/null`, so their real
error text (a jq syntax/runtime error) is discarded, not just unlogged. Both
workers who built this independently agreed: **do not** restructure
`result=$(jq ... 2>/dev/null)` to tee the suppressed stderr into the new
diagnostic line. The stage-naming message already tells you which of the
three mining/pricing stages failed; capturing the underlying jq stderr too is
real additional plumbing risk (a `2>&1` merge or a `tee` risks leaking stderr
into the captured stdout, corrupting the `{}` contract) for marginal
diagnostic gain over what the stage name already tells you. Recorded here as a
possible future enhancement, not built.

### Verification

- **71 ok, 0 FAIL** on `hooks/scripts/tests/loop_cost.test.sh`, re-run live
  during this ingest.
- **Live-verified under zsh on merged `origin/main`** (re-run during this
  ingest, matches exactly): a bogus session id produces
  `stdout=[{}]`, `exit=0`,
  `stderr=[loop_cost: no transcript found for session bogus-xyz under
  /Users/harrison/.claude/projects]`.
- New test: **"pairwise distinctness"** — asserts exactly 7 loop_cost stderr
  bail messages exist and that all 7 are pairwise unique (no two bails share
  wording), closing off the same "shared needle" discriminating-check failure
  mode PR #216 hit above, this time applied to messages instead of route
  needles.

## Lessons (repo-agnostic, worth carrying beyond this cluster)

1. **A fabricated safe route is worse than the generic fallback.** "There is
   no safe equivalent, here is the opt-in path" is a real, honest answer. An
   invented alternative sends someone down a false path with false confidence.
   Six of PR #216's 13 routes chose honest disclosure over invention.
2. **A check that cannot both PASS and FAIL is not a check.** Two real defects
   in PR #216 were exactly this class — shared needles across six routes
   (proven by successfully swapping `dd`'s text into `shred`'s case and
   staying green), and bare `-n`/`-i` two-char substrings matching inside
   unrelated words (proven by a fabricated route containing `"manual-n"`/
   `"4-i"` passing clean). Both were fixed and negative-controlled before
   merge — see [[feedback_validate_frozen_checks]] for the general form of
   this failure mode.
3. **A fail-open collapses N causes into one value.** `{}` said nothing about
   *why*, so four loops each guessed a different wrong cause from outside the
   function, none verified inside it. Fixing the silence is as important as
   fixing whatever bug the silence was hiding — see
   [[feedback_fail_open_hides_its_cause]].
4. **The gate blocks its own patterns even inside a comment or quoted
   string — a known, NOT-fixed-here follow-up.** During this cluster's own
   build, [[destructive_bash_gate]] blocked an orchestrator's own
   `progress.json` update multiple times for merely *quoting* a pattern while
   documenting that it had been blocked — a mention-vs-invocation false
   positive. Workaround used: split the literal across concatenated fragments
   (e.g. `"mk""fs"`) assembled at runtime rather than written inline. The
   gate's own comments already record three prior narrowing attempts at this
   exact exemption, each of which admitted a real bypass — so this is
   recorded as a standing, deliberately-unresolved tension, not a defect to
   silently patch. See [[pr_201_202_203_routine-followups]] for the
   identical symptom hit by the previous cluster.

## Files changed

**PR #216:**
- `hooks/scripts/destructive_bash_gate.sh` — `deny()`'s route lookup extended
  from 4 to 14 case arms
- `hooks/scripts/tests/destructive_bash_gate.test.sh` — new
  `assert_specific_route` helper, 13 new route assertions, new source-drift
  tripwire (`EXPECTED_FIXED_LABELS`/`EXPECTED_PATTERN_LINE` snapshots)

**PR #217:**
- `hooks/scripts/lib/loop_cost.sh` — 6 new `echo ... >&2` diagnostic lines
  (lines 57, 58, 100, 163, 164, 217)
- `hooks/scripts/tests/loop_cost.test.sh` — new stderr-message assertions per
  bail site + the pairwise-distinctness assertion

## Wiki pages updated

- [[destructive_bash_gate]] — route table extended from 3 to 13 named
  patterns; new subsections on the sync-enforcement design fork and the two
  discriminating-check defects
- [[loop_cost]] — new page (did not exist before this ingest); documents
  `dc_mine_token_usage`'s fail-open contract and this PR's diagnostic layer
  in one place, rather than only as scattered cross-references from
  [[loop_stall_guard]] and [[pr_204_cost-reporter]]

## See also

- [[pr_201_202_203_routine-followups]] — PR #203, the first 3 named routes
  this PR extends to all 13; the mention-vs-invocation gate self-block was
  already hit there
- [[pr_184_185_186_loop-cost-tracking]] — `dc_mine_token_usage`'s original
  build and fail-open contract
- [[pr_204_cost-reporter]] — the reporter that reads `retro.cost`, and the
  explicit "do not fix this into fail-closed" design note this PR's fix
  respects
- [[feedback_fail_open_hides_its_cause]] — the four-wrong-causes memory this
  PR's stderr fix directly addresses
- [[feedback_validate_frozen_checks]] — the discriminating-check principle
  both PRs' review caught violations of
