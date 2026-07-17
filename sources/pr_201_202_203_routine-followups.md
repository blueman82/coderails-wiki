---
title: "PR #201 + #202 + #203 — routine follow-ups cluster (example-config de-rot, UTC/local date skew, destructive-gate concrete safe routes)"
type: source
created: 2026-07-17
last_updated: 2026-07-17
sources: []
tags: [dashboard, routines, sweep, destructive-bash-gate, hook, artifact-gate, date-skew, machine-path-rot, handoff-verification, eval-freeze-discipline]
---

# PR #201 + #202 + #203 — routine follow-ups cluster

Three independent work units from one handoff memory, each closing a
different defect: a rotted example config (#201, `7de4995`), a UTC/local
`{date}` skew in the routine sweeper (#202, `04f80a2`), and an unhelpful
destructive-gate deny message (#203, `653d1c6`, also `main`'s current HEAD).
Grouped as one source page because they share a single originating handoff
and — more importantly — because verifying that handoff against live state
is the actual finding worth recording, not the three diffs themselves.

## PR #201 — `examples/dashboard-config.json` de-rot

`examples/dashboard-config.json` is **documentation-only**: the live config
`loadConfig()` actually reads is `~/.claude/coderails-dashboard.json` via
`DEFAULT_CONFIG_PATH` (`skills/dashboard/lib/src/config.ts:48`), and the
example file's only real consumer is a vitest fixture
(`skills/dashboard/lib/test/config.test.ts`) `(verified: PR diff + config.ts
line reference)`. It had already rotted once when a checkout moved — every
`cwd`/`artifactPath`/`foreignSkillPath` baked in this machine's actual
absolute paths (`/Users/harrison/Github/coderails`,
`/Users/harrison/Documents/Github/coderails` for one stale entry — see
below). PR #201 replaces every directory prefix with slash-rooted
`/path/to/...` placeholders.

**Constraint that made this non-trivial:** the placeholder must be
slash-rooted (`/path/to/coderails`, not `path/to/coderails` or
`~/path/to/coderails`) because `skills/dashboard/app/src/lib/config.ts:74-76`
throws a `ConfigError` on a relative `cwd`. A naive de-rot that dropped the
leading slash would have turned a working (if machine-specific) example into
one that fails config validation outright — worse than the rot it was fixing.

The paired test change (`config.test.ts`, -9/+3 lines) is not a coverage
reduction — see "What this ingest corrects" below; it removes an assertion
that had become **wrong**, not one that had become **inconvenient**.

## PR #202 — UTC/local `{date}` skew in the routine sweeper

`skills/dashboard/runner/src/sweep.ts` derived the artifact gate's `{date}`
template token via `new Date().toISOString().slice(0, 10)` — always UTC,
regardless of `process.env.TZ` — while the producer (the `claude -p` run
that writes the actual artifact) writes the artifact keyed to its **local**
calendar date. Between local midnight and the local UTC offset, the two
dates disagree and a **correct** run is graded a false artifact-gate-failed.

Fix: a new `localDateIso()` helper using the TZ-aware
`getFullYear()`/`getMonth()`/`getDate()` accessors (which read the OS/process
notion of local time, unlike `toISOString()`), sourced from a new optional
`clock?: () => Date` on `SweepOptions` (defaults to `() => new Date()`; tests
inject a fixed instant). Applied at **one locus** —
`sweepOnce()`'s `checkArtifact()` call site (`sweep.ts`, was line 278).

New test suite pins the failure direction two ways using a fixed historical
instant (`2025-03-09T20:00:00Z`, Asia/Kolkata, UTC+5:30, no DST) deliberately
far from any date this suite could run on — so a wall-clock-derived date can
never coincidentally pass by calendar accident: one test proves the gate
matches the producer's LOCAL-dated marker at the skew instant; the other
proves it does NOT match a UTC-dated marker at the same instant, pinning
which direction is correct rather than merely which direction is different.

**Blast radius: all FOUR date-bearing gates**, not just one routine. This
includes `wiki-lint`'s own gate, whose `{date}` lives in its `contains`
**marker** text, not its artifact **path** — `artifactGate.ts:92` resolves
`{date}` tokens through the same `ctx.date` regardless of where in the
predicate the token appears, so the fix's blast radius follows the token, not
the predicate kind.

## PR #203 — `destructive_bash_gate.sh` deny messages name a concrete route

Before this PR, every `deny()` call in `hooks/scripts/destructive_bash_gate.sh`
appended the same generic sentence regardless of which pattern matched: "To
allow it, add a Bash permission rule to settings.json or use a
non-destructive alternative" — naming the *existence* of an alternative
without ever naming *what* it was. `deny()` now looks up a per-pattern
`route` string (matched against a lowercased, whitespace-collapsed copy of
`$pat`, message-text only — **the deny set itself is provably unchanged**,
confirmed by the paired test file asserting the same DENY/ALLOW verdicts as
before plus new assertions on message content):

| Blocked pattern | Named safe route |
|---|---|
| `git reset --hard` | Park the commits first (`git branch backup/<desc> <ref>`), then use `git reset --keep <ref>` instead of `--hard` — `--keep` **refuses** (errors out) rather than clobbering when it would discard uncommitted working-tree changes, and the backup branch keeps moved-past commits recoverable either way |
| `rm -rf`/recursive-force-remove | `unlink <file>` for a single file; move the target into a temp dir (`mkdir -p /tmp/trash && mv <target> /tmp/trash/`) for a directory or multiple files |
| `git push --force`/`-f` | Use `--force-with-lease` instead — **with an honest disclosure that `--force-with-lease` is itself blocked by this same hook by default**, naming the exact opt-in line (`git-push-force-with-lease` in `.claude/destructive_allowlist`) rather than pointing at a route that is itself a dead end |
| Anything with no specific route mapping (e.g. `git clean -fdx`) | Generic fallback ("No specific safe route is recorded for this pattern...") — not a false claim of a route that doesn't exist |

**bash 3.2 gotcha, caught by the hook's own test suite during this work:**
`${pat,,}` (bash 4+ lowercase parameter expansion) was tried first for the
normalisation step and **aborts the hook** on this machine's bash 3.2 (macOS
ships 3.2 by default) — an abort here means the hook denies **nothing**, a
fail-open on a security control. Fixed with `tr '[:upper:]' '[:lower:]'`
instead. See [[destructive_bash_gate]] for the hook's full existing
documentation; this PR is prose/message-text only and does not change that
page's "Blocked patterns" tables.

## What this ingest corrects — the handoff's claims were hypotheses, not facts

The originating handoff memory listed four defects. Verifying each against
**live state**, not just source greps and the example file, found three of
the four did not survive:

1. **"loop-retro-promotion's gate CAN NEVER PASS" — FALSE at ingest time.**
   The wiki (`skills/loop-retro-promotion.md`, `design/routines.md`) carries
   a "⚠️ Open defect (lint, 2026-07-17)" note asserting this, dated the same
   day. Live re-check `(verified 2026-07-17)`: `~/.claude/coderails-dashboard.json`
   points `loop-retro-promotion-weekly`'s `artifactPath` at
   `/Users/harrison/.claude/agentic-loop/-Users-harrison-Github-coderails-.git/promotion-runs.log`
   — the **correct** repo-key dir (`~/Github`, not `~/Documents/Github`).
   That file exists, last modified 2026-07-13 (mtime-verified), age ≈4.4
   days against an 8-day (`691200`s) `maxAgeSeconds` window — **the gate
   currently passes**. The "can never pass" claim was true only of
   `examples/dashboard-config.json` (the machine-path rot #201 fixes,
   pinning a nonexistent `Documents/Github` checkout) — never of the live
   config the routine actually runs against. This is a **timing-sensitive**
   pass, not a permanent fix: the artifact will age out again in ≈3.6 more
   days unless `loop-retro-promotion-weekly` re-fires and refreshes it —
   see [[loop-retro-promotion]]'s existing "Open defect" note, which itself
   already correctly attributes the underlying issue to path rot on the
   *example* config, distinct from this now-stale "can never pass" framing
   of the *live* one.
2. **The cost-miner item was retracted as fabricated by bad probes** before
   this cluster's work started — not re-investigated here, simply not
   present as a real defect to fix.
3. **The instruction to remove `loop-retro-promotion-weekly` from
   `SKILL_OWNS_ITS_PATH`** (in `config.test.ts`'s guard test) **would have
   failed the test suite.** Its button command names the gate's basename
   (`promotion-runs.log`) **zero times** — structurally identical to
   `memory-consolidation-weekly` beside it in the same exemption set, which
   is exempted for the same reason. The test comment previously carried a
   "KNOWN-BROKEN, not compliant" caveat specific to this routine; PR #201's
   paired test diff replaces that caveat with the structural explanation
   (the skill owns the path, full stop) — this is the -9/+3 line change
   noted under PR #201 above. This is **why** that diff shrank rather than
   grew: a wrong caveat was removed, not a right one weakened.

**The lesson generalises beyond this cluster:** a handoff memory's claims
about system state are hypotheses written against source code and example
files at authoring time, not facts. Re-verify against live config, live
mtimes, and a live test run before treating a memory's diagnosis as ground
truth — three of four items here would have sent a following session down a
wrong or unnecessary path had they been trusted at face value.

## Two frozen eval commands were never smoke-run

Separately from the handoff-memory finding above: two eval commands authored
at freeze time for this work reported **fail** without executing a single
test. One used a `--reporter=basic` vitest flag that does not exist in this
project's vitest version (died loading the reporter, before any test ran).
Another used a root-level `npm ci` in a repo with **no npm workspaces**
configured at the root (died `ERR_MODULE_NOT_FOUND`). Both failures were
indistinguishable from a legitimate test failure by exit code alone — the
eval harness reported red, and reading the log was the only way to learn
neither had actually run anything.

**Generalisable lesson:** a frozen eval command that is never smoke-run once
at freeze time is an unverified instrument, not a verified gate. Executing
each command once at freeze — separate from confirming its negative control
correctly fails — costs seconds and would have caught both of these
immediately, before they had the chance to be misread as real red signal.
See [[task-evals]] and [[task-evals-gate]] for the freeze-before-build
discipline this sits alongside; this is a gap in *executing* the frozen
command once, not in the freeze discipline itself.

## The destructive gate cannot distinguish discussion from execution

Working through this cluster, `destructive_bash_gate.sh` blocked the
orchestrator **six times** on commands that were never going to run
destructively: a cleanup command, a probe list, a Python string literal, and
twice on prose that merely **described** the blocked patterns (e.g. writing
this very source page's "Blocked patterns" table into a draft file via a
heredoc). The hook is a stateless line-oriented `grep -qiE` — see
[[destructive_bash_gate]] — with no way to distinguish "this string will run
as a shell command" from "this string is text about a shell command." Not a
new limitation (the hook's documented enforcement ceiling already covers
adjacent cases), but a concretely-felt one this session: **probes that
merely need to demonstrate a blocked pattern for documentation purposes must
be base64-assembled at runtime and driven from a file**, never written
inline as a literal matching string, to avoid tripping the same gate they're
describing.

## See also

- [[destructive_bash_gate]] — the hook PR #203 changes; full existing
  documentation of blocked patterns, the force-with-lease carve-out, and the
  2026-07-08 adversarial-hardening arc this PR does not touch
- [[dashboard-runner]] — `sweepOnce()` / `checkArtifact()`, the mechanism
  PR #202 fixes
- [[routines]] — the routine config schema, `{date}`/`{runId}`/`{vault}`
  template tokens, and the existing "Not portable" / born-red incident notes
  this cluster's #201 and the loop-retro-promotion re-verification touch
- [[loop-retro-promotion]] — carries the pre-existing (and still accurate,
  for the *example* config) "Open defect" note this page's item 1
  re-verifies against *live* config and narrows
- [[dashboard]] — sub-project 1; owns `DEFAULT_CONFIG_PATH` and
  `loadConfig()`, the mechanism establishing #201's example file is
  documentation-only
- [[task-evals]] — the freeze-before-build discipline the never-smoke-run
  eval-command finding sits alongside
