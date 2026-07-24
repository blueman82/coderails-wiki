---
title: "PR #298 — .env secret-file access denied in destructive_bash_gate"
type: source
created: 2026-07-24
last_updated: 2026-07-24
sources:
  - hooks/scripts/destructive_bash_gate.sh
  - hooks/scripts/tests/destructive_bash_gate.test.sh
  - AGENTS.md
  - README.md
  - docs/REFERENCE.md
tags: [source, hook, destructive-bash, secrets, dotenv, case-insensitivity, mutation-testing, over-blocking]
---

# PR #298 — .env secret-file access denied in destructive_bash_gate

<!-- Ingested by /wiki-ingest after merge. This is an immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR number | #298 |
| Branch | `feat/dotenv-bash-gate` |
| Merged | 2026-07-24 (`2026-07-24T09:51:52Z`) |
| Merge commit | `8fba16ded07e2303c935ab15aef0193f838af544` |
| Head commit | `9c53b2d58ed4c54cc8b08584725c2aeabe97e155` |
| JIRA ticket | — (RCA item 12) |

## Summary

Adds a `.env` deny branch to [[destructive_bash_gate]], covering **read and
write in one rule**. `.env` was genuinely unguarded: the gate had **zero**
`.env` matches before this PR (verified — `git grep -c dotenv_hit` on the
pre-merge tree returns nothing), and a previously-removed hook that *claimed*
to block `.env` access never actually worked. A real gap, not a duplicate of
existing protection.

Folded into `destructive_bash_gate.sh` rather than shipped as a new hook,
because that file already has the correct JSON-decision plumbing: the deny
routes through the existing `deny()` function and carries
`permissionDecisionReason` + `patternId` (`dotenv-access`) at exit 0, identical
to every other pattern. **No new JSON shape was invented** — the same
discipline [[hook-exit-codes]] records.

## The design call: match the path token, not a verb list

The detector is deliberately **command-agnostic**. It matches the `.env` path
token anywhere on the line rather than enumerating reader/writer verbs.

The argument, which generalises well beyond this hook: **a verb enumeration is
unbounded and every omission is a silent bypass.** `cat`, `bat`, `less`, `more`,
`view`, `vim`, `nano`, `awk`, `sed`, `xxd`, `python -c open(...)` — the list has
no closing condition, and a gate built on it fails silently at exactly the
moment someone uses the verb nobody listed. Matching the *object* rather than
the *action* has one enumeration point (what a `.env` path looks like) instead
of an open-ended one.

The cost is paid in **boundary precision**, which is where all the difficulty
went — this hook gates every Bash call, so a false positive breaks the session,
which is worse than the gap it closes.

## Boundaries

- **Left**: start-of-line, whitespace, quote, `/`, a redirect char, `=`, or `#`
  — so `./.env`, `../.env`, `/abs/path/.env`, `>.env`, `VAR=.env` all match. A
  preceding *word* character is deliberately **not** a boundary, so `myapp.env`
  and `config.env` don't match: the spec is dotfile-shaped, and treating any
  `*.env` as secret would deny `cat myapp.env.example`.
- **Right**: end-of-line, whitespace, quote, shell separator (`;` `|` `&`),
  `)`, a redirect char, or an editor-backup marker (`~` vim, `#` emacs
  autosave). **Not a word char** — that single exclusion is what keeps
  `.envrc` (direnv, an entirely different file) allowed while `.env` denies.

`.env~` and `#.env#` hold a byte-identical copy of the secret and are created
by the *editor*, not by a deliberate act, so excluding them would have left the
copies reachable while the original was denied.

## Case-insensitivity — four verified bypasses caught in review

The first draft was case-sensitive and `cat .ENV`, `cat .Env`, `.ENV.LOCAL`
were all verified bypasses. **macOS (APFS) and Windows are case-insensitive by
default, so `.ENV` opens the same inode as `.env`** — a case-sensitive matcher
is defeated by pressing shift.

The fix is a single up-front lowercasing of the line into `$dotenv_cmd` via
`tr`, and **deliberately not a bare `grep -i`**. The reason is a genuine
interaction, not preference: the suffix branch does a case-**sensitive**
`${tok#.env.}` strip and compares against a lowercase allow-list, and neither
parameter expansion nor `case` honours grep's `-i`. Matching case-blind while
stripping case-sensitively would have flipped `.ENV.EXAMPLE` — the *same file*
as `.env.example` on APFS, a benign template — from allowed to **over-blocked**.
`tr`, not `${var,,}`, because bash 3.2 is this file's floor — the same
constraint that produced a fail-open in
[[pr_201_202_203_routine-followups|PR #203]].

The generalisable lesson: **a case-folding fix has to be applied at the same
layer as every consumer of the folded value**, or it fixes one branch and
breaks another.

## Why two branches instead of one regex

"Deny `.env.<suffix>` **except** example/sample/template/dist" is a negative
lookahead. POSIX ERE (`grep -E`, bash 3.2 — no PCRE available) cannot express
it. So the suffixed case extracts each token and tests its suffix in bash.

Only the **first** suffix segment is compared (`${suffix%%.*}`): `.env.example.md`
(docs *about* the template) stays allowed, while `.env.local.bak` — a backup of
a real secret file — still denies. Comparing the whole multi-part suffix would
deny the former, which is the over-block this block was most at risk of.

**The sharpest discriminator, both directions**, and the one that kills the
obvious wrong implementation:

- `cp .env.example .env` → **DENY**. One line carrying both an allow-token and
  a deny-token. Any "grep the line for a template name and exempt it wholesale"
  implementation gets this wrong and would allow *writing the real secret file*.
- `cp .env.example .env.sample` → **ALLOW**. Template to template, no real
  secret named.

## Mutation testing — 8 mutants, 8 killed, 0 survivors

Each pattern was reverted individually against a **copy** of the gate (the live
file was never modified) to confirm the corresponding test actually fails, so
kills are isolated and no pattern is covered only incidentally by another
branch. Removing the bare-`.env` branch flips 28 checks; removing the suffixed
branch flips exactly 3; widening the right boundary to word chars flips
`.envrc` to allow; removing the template carve-out flips 6; comparing the whole
suffix instead of the first segment flips exactly 1 (`.env.example.md`).

The **isolation** is the point, and it is the direct answer to the defect
[[pr_216_217_safe-routes-and-cost-miner-diagnostics|PR #216]] found in its own
first draft — checks that pass for the wrong reason because another branch
incidentally covers them.

## Ceilings — deliberate, and stated as uncovered rather than closed

Same class as this gate's other documented ceilings:

- **Shell-expansion forms are uncaught.** Matching runs before the shell
  expands anything, so `cat .env*` and `cat .en?` glob onto the real file and
  are **allowed**. **Frame this precisely: no `.env` gate existed at all
  beforehand, so these are uncovered cases, not a bypass the PR introduced.**
  The PR's own comment says so and tells the reader not to read the boundary
  paragraph as closing them.
- **A variable-held path** (`F=.env; cat "$F"`) is uncaught once the literal is
  off the line — the same ceiling the source-edit blocks already carry.
- **A template suffix off the closed list** (`.env.tpl`, `.env.defaults`)
  denies — **fails closed**, costs one `settings.json` rule, leaks nothing.
- **A real secret named to look like a template** (`.env.example` holding live
  keys) is allowed — naming convention is the only signal a line-oriented
  matcher has.
- **Admitting `#` as a boundary over-blocks** any literal `#.env` on the line,
  including a URL fragment like `http://host#.env` — fails closed, on input no
  real workflow emits.

**A follow-up should invert the boundary rule** — treat any non-filename
character as a boundary — rather than patching `*` and `?` individually. The
enumerate-the-exceptions approach is the same open-ended shape this PR rejected
for verbs.

## The safe route

`dotenv-access`'s route is unusual among this gate's 14 arms in that it names
several alternatives at different privilege levels: test whether a key is *set*
without revealing it (`set -a; . ./.env; set +a; [ -n "$MY_KEY" ]` — while
honestly noting that still needs an allow rule, so prefer asking the owner);
read the committed template to see the file's *shape* (`cat .env.example`,
which this hook already permits); edit by hand outside the session; or add a
`settings.json` Bash permission rule. It does not claim a route that doesn't
exist — the discipline [[destructive_bash_gate]] adopted in PR #216.

## Wiki pages updated

- [[destructive_bash_gate]] — new `.env` section, blocklist and route tables extended

## Caveats / gotchas

- `AGENTS.md`, `README.md` and `docs/REFERENCE.md` were updated **in the code
  repo** by this PR. Those are raw source (layer 1), not wiki content — the
  wiki records the knowledge, it does not mirror those files.
- This is a **new deny with no approval path** beyond a `settings.json`
  permission rule, on a hook that gates every Bash call. The over-block ceilings
  above are the live cost of that.

## See also

- [[destructive_bash_gate]] — the hook this extends
- [[hook-exit-codes]] — why the deny is JSON at exit 0, not exit 2
- [[pr_216_217_safe-routes-and-cost-miner-diagnostics]] — the named-safe-route discipline and the incidental-coverage defect this PR's mutation isolation answers
- [[pr_201_202_203_routine-followups]] — the bash 3.2 `${pat,,}` fail-open, the same floor that forced `tr` here
- [[pr_92_2026-07-08_reference-drift-and-lookalike-fp]] — the previous boundary-anchor false-positive on this same gate
