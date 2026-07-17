---
title: "PR #232 — Tier-review gate: root daemon that judges self-classified tier-0 honesty"
type: source
created: 2026-07-17
last_updated: 2026-07-17
sources: []
tags: [tier-review, trust-domain, root-daemon, subscription-auth, verdict-laundering, prefilter, defence-in-depth, honest-boundary]
---

# PR #232 — Tier-review gate: root daemon that judges self-classified tier-0 honesty

<!-- Ingested by /wiki-ingest after merge. This is an immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR number | #232 |
| Branch | `feat/tier-review` |
| Merged | 2026-07-17 (22:29:57Z) |
| Merge SHA | `88915bbb08c682eac404dfd45435582703bf027b` |
| JIRA ticket | — |

## Summary

[[task-evals-gate|Task-evals]]' tier-0 exemption (self-classified "no eval
needed") was, until this PR, entirely self-graded — nothing reviewed whether
an agent's own `tier=0` claim on a PR was honest. This PR builds
`scripts/tier-gate/`, a **root-owned macOS LaunchDaemon** that polls open
PRs, finds each PR's SHA-bound eval-artifact marker, and for any `tier=0`
claim, judges the claim against the PR's real diff using a
subscription-authenticated Claude call running **outside the agent's own
trust domain** — then posts a `tier-review` GitHub commit status the merge
gates ([[merge]], [[enforce_pr_workflow]]) can check. Nine fixes shipped
across the PR's own commit history, plus four blockers a security review
caught and closed before merge. (verified — `gh pr view 232 --json commits`,
`git show 88915bb` on every file cited below)

**Honest boundary, stated prominently because it governs how this whole
system should be read:** the gate is **available, not active**. It is dormant
until the repo owner completes four separate activation steps outside this
PR's scope: a root-owned `claude` install (this step is done — the daemon
execs `/var/root/.local/bin/claude`), a minted `CLAUDE_CODE_OAUTH_TOKEN` in
the root credentials file, a GitHub branch-protection ruleset requiring the
`tier-review` status check, and a live-fire E1–E9 validation pass. Until all
four land, `coderails/`'s own repo runs with no tier-review daemon polling it
— this PR ships the mechanism, not its activation. (verified — PR body /
commit history; no evidence in the diff of a ruleset or a populated
credentials file)

## Files changed

- `scripts/tier-gate/tier-gate-runner.sh` (947 lines, new) — the daemon's poll/judge/post lifecycle
- `scripts/tier-gate/install.sh` (221 lines, new) — preflight + root install (testable without root; side effects behind a main-guard)
- `scripts/tier-gate/judge-prompt.md` (66 lines, new) — the static, unsubstituted judge prompt
- `scripts/tier-gate/com.coderails.tier-gate.plist.template` (27 lines, new) — LaunchDaemon plist template
- `scripts/merge.sh` (+69/-0) — redundant local tier-review status check before merge
- `hooks/scripts/enforce_pr_workflow.sh` (+87/-0) — same check, mirrored into the `gh pr merge` hook path
- `skills/task-evals/SKILL.md` (+2) — one paragraph pointing tier-0's self-exemption note at this daemon
- `examples/workflow.config.yaml` (+10) — documents the opt-in `tier_review.machine_user` key
- Six new test files (`tier_gate_install.test.sh`, `tier_gate_judge_auth.test.sh`, `tier_gate_prefilter.test.sh`, `tier_gate_runner.test.sh` [991 lines], `merge_tier_review_gate.test.sh`, plus additions to `enforce_pr_workflow.test.sh`/`post_evals.test.sh`/`exec_bit_invariant.test.sh`)

## How the judge is kept blind and injection-immune

The judge sees exactly two inputs — `{claimed_tier, diff}` — and nothing
else. The defendant's own `evals.json` prose (the tier-0 `tier_justification`
field an agent writes about its own work) is **never read by the judge at
all**; this channel was deleted, not sanitized or hardened. (verified —
`cc5c570 Fix tier-gate judge injection: judge sees {claimed_tier, diff}
only`, and `judge-prompt.md`'s own header comment: "STATIC — it carries no
placeholders and no defendant-authored text... nothing here for the
defendant to inject into") `judge-prompt.md` is concatenated with the two
blind inputs strictly appended after the instructional text, never
substituted into it — so there is no fence, token, or delimiter inside the
prompt template for diff content to collide with or escape.

The judge's verdict predicate is a restatement of [[task-evals]]'s own
tier-0 rules (single work-unit, no outward/irreversible surface, existing
coverage) — the prompt explicitly treats `skills/task-evals/SKILL.md` as
source material, not documentation to defer to, and restates the predicate
directly rather than delegating interpretation to the model at judge time.

## Auth model: subscription, not a metered key — and why that matters

The judge authenticates via `claude -p` using `CLAUDE_CODE_OAUTH_TOKEN` —
the **owner's own Claude subscription**, root-held in a credentials file —
never a metered Anthropic API key. (verified — `51768a3 tier-gate: rewire
judge to subscription claude -p + pin cwd (fixes 5-6)`) This is a
**permanent owner veto**: the daemon cannot make judge calls the owner
hasn't already paid for and enabled by installing a live token, and revoking
that token silently disables judging with no separate kill-switch needed.

The daemon execs a **root-owned** `claude` binary
(`/var/root/.local/bin/claude`) with `HOME` and `cwd` pinned to a root-owned
directory. This closes a specific escalation: uid 501 (the interactive
agent's own account) is the same identity whose PRs get judged, and if the
judge process picked up a uid-501-writable `CLAUDE.md` or config from an
inherited `HOME`, the defendant could plant instructions the judge would
read as its own system context — a much stronger injection than anything
reachable via the diff content alone (the diff-content path is closed
separately, see above). Pinning `HOME`+`cwd` to a root-owned directory
means no uid-501-writable path is ever on the judge process's config
resolution path. (verified — `git show 88915bb:scripts/tier-gate/tier-gate-runner.sh`
header comments + the `51768a3` commit)

## All daemon GitHub I/O routes through root-owned curl, never `gh`

Every read AND the credentialed status write goes through
`TIER_GATE_CURL_BIN` (`/usr/bin/curl`, root:wheel) to the GitHub REST API
directly — the daemon deliberately never execs `gh`, because `gh` lives
under `/opt/homebrew/bin`, which is uid-501-writable. A root daemon exec'ing
a uid-501-writable binary is a privilege-escalation surface (uid 501 swaps
the binary, root runs it next poll tick) — there is no root-owned `gh` to
pin instead, so curl at a genuinely root-owned path is the fix. (verified —
`7dc8012 tier-gate: route daemon reads through root-owned curl, fail-closed`,
and the `tg_gh_get` header comment in `tier-gate-runner.sh`)

Reads are fail-closed by construction: `curl`'s own exit code reflects
transport only, not HTTP status (unlike `gh`, which exits nonzero on a 4xx/5xx).
`tg_gh_get` appends the HTTP status via `-w` and rejects any non-2xx
response **before** the body is ever used, so a 404/500 error body can never
be silently read as valid PR/diff data. An empty 2xx body is not itself
treated as failure (a legitimate empty statuses list is `[]`); call sites
that must reject an empty body enforce that themselves.

## Mechanical prefilter — blocks the obvious cases with no model call

`tg_prefilter(filelist, line_count)` is model-free by construction: it only
inspects file paths and a line count (both mechanically derived facts about
the diff's shape), never diff content, so there is nothing for diff text to
inject into at this stage. It blocks on either a path-denylist hit
(`TIER_GATE_PATH_DENYLIST`) or a size cap
(`TIER_GATE_MAX_FILES`/`TIER_GATE_MAX_LINES` exceeded) — the PR's own
example of an obviously-dishonest tier-0 claim is a 205-line, 1-file diff
that the size cap alone would reject before any judge call runs. (verified —
`tg_prefilter` function body, `tier-gate-runner.sh:739-768`)

## Verdict-laundering is closed, not just detected

Two independent closures, both from the same review-caught blocker
(`b87272c tier-gate: close verdict-laundering bypass (review blocker
C1/HIGH)`):

1. **The daemon posts nothing for a non-tier-0 claim.** A `tier=1`/`tier=2`
   artifact short-circuits to a no-op — no status of any kind is posted for
   it. This matters because a forged `tier=0` marker still faces the judge +
   prefilter (the real defence), but a forged *non*-tier-0 marker can't mint
   a reusable success status either, since none is ever posted for it to
   later be reinterpreted as a tier-0 pass.
2. **`state=success` alone is not accepted as approval — the description
   must carry `verdict=legitimate`.** Both `scripts/merge.sh` and
   `enforce_pr_workflow.sh`'s local gates check this explicitly: a
   `tier-review` status can be `state=success` for reasons that are NOT a
   genuine tier-0 approval (the code comment names this precisely as the
   "verdict-laundering path"), so a bare state check cannot distinguish
   those from a real judged pass — only the description's embedded verdict
   string can. (verified — `scripts/merge.sh` diff, the `tr_desc` check and
   its inline comment; identical logic mirrored in
   `enforce_pr_workflow.sh`'s `gate_tier_review_status`)

## Where the gate is enforced — redundant-not-primary, explicitly labelled

Both `scripts/merge.sh` (the `/coderails:merge` path) and
`hooks/scripts/enforce_pr_workflow.sh` (the raw `gh pr merge`/`scripts/merge.sh`
hook path) carry near-identical local checks, both config-keyed and inactive
by default: they only run when `tier_review.machine_user` is set in
`workflow.config.yaml` **and** the PR's own eval artifact reports `tier=0`.
When active, the check requires the newest `tier-review` commit status on
the head SHA to be `state=success`, posted by exactly the configured
machine-user login (a mismatch is treated as a misconfiguration-or-forgery
signal, never bypassed), and carrying `verdict=legitimate` in its
description.

Both call sites' own code comments state the same thing verbatim: **this
layer is redundant by design once the server-side ruleset is live** — it
exists to fail loudly on misconfiguration and to hold the line during the
pre-ruleset interim, and it is explicitly NOT the primary control. The
primary control is the GitHub branch-protection ruleset (required status
check), which no local agent-side hook can fake — the same
enforcement-ceiling distinction [[enforcement-model]] already documents for
every other hook in this repo. The comments explicitly instruct: do not
delete this as dead code once the ruleset is active, because it remains the
only *local* check that catches a machine-user misconfiguration before
GitHub itself would. (verified — identical comment block duplicated in both
`scripts/merge.sh` and `hooks/scripts/enforce_pr_workflow.sh`'s diffs)

`examples/workflow.config.yaml` documents the new key with the check
inactive by default (commented out, `machine_user: null`) — installs that
never opt in are unaffected.

## The nine in-PR fixes + four review-caught blockers (chronology)

Reconstructed from the PR's own commit history (`gh pr view 232 --json commits`),
oldest to newest:

1. Runner build: poll/judge/post lifecycle (`tg_gate_pr`, marker parsing, comment fetch)
2. `post_evals.sh`: validate that the posted body embeds the tier-0 eval artifact
3. Redundant local gate added to `merge.sh` + `enforce_pr_workflow.sh`
4. `tg_judge`: initial direct Anthropic API call implementation
5. Install script + tests, exec-bit manifest registration
6. **Injection fix** — judge narrowed to `{claimed_tier, diff}` only, `evals.json` prose channel deleted (`cc5c570`)
7. **Auth rewire** — judge moved off the direct API call to subscription `claude -p`, `HOME`/`cwd` pinned (`51768a3`, "fixes 5-6")
8. `MACHINE_USER` sourced from the credentials file rather than a phantom env var
9. Identity-bound status POST + honest `MODE` preflight line in `install.sh`

Four blockers caught by review before merge, each with its own fix commit:
- **C1/HIGH — verdict-laundering bypass** closed (`b87272c`): the two fixes described above (post-nothing for non-tier-0; `verdict=legitimate` required, not just `state=success`)
- Runner tests reconciled to the post-nothing contract (`3ba9faa`, direct follow-up to C1)
- **Root-owned curl routing** for daemon reads, fail-closed (`7dc8012`) — closes the `gh`-is-uid-501-writable privesc surface described above
- **A5 discriminating-check fix** — an `is_error` test that was a tautology (always passed regardless of input) made to actually discriminate (`d5830e8`) — the same defect class [[task-evals-gate]]'s "Discriminating-check gate" section documents generally: a frozen check that cannot fail is worse than no check
- Prefilter/gate coverage gaps closed (`8dc2aea`)

## The pitch, stated honestly

This system raises the cost of a dishonest tier-0 claim from **free**
(nothing reviewed it before this PR) to **expensive** (a diff-shape
prefilter plus a blind, injection-immune, subscription-authenticated model
judgment, both outside the defendant's trust domain). It is explicitly **not**
a claim of impossibility — an LLM judging a diff remains fallible, and this
PR doesn't change that. What it does change is that the judge is no longer
manipulable via the injection channel that existed before the `cc5c570` fix
(the defendant's own `evals.json` prose is deleted from what the judge ever
sees, not merely escaped or sanitized).

## Wiki pages updated

- [[task-evals-gate]] — new section, or cross-reference, needed for the tier-0 self-exemption caveat
- [[task-evals]] — the `SKILL.md` paragraph pointing tier-0's self-exemption note at this daemon
- [[merge]] — new tier-review status check in the merge path
- [[enforce_pr_workflow]] — new `gate_tier_review_status` function in the hook's merge-gate path
- [[trust-floor]] — this is a second, distinct trust-boundary mechanism (root daemon vs. repo-permission check) worth cross-referencing, not conflating
- [[enforcement-model]] — another concrete instance of the redundant-local-check-vs-primary-server-side-control distinction

## Caveats / gotchas

- **AVAILABLE != ACTIVE.** This is the single most important fact about this
  PR for anyone reading the wiki later: the mechanism ships fully built and
  tested, but this repo (`coderails` itself) does not yet run it — no
  ruleset, no live credentials file confirmed populated with a real
  `CLAUDE_CODE_OAUTH_TOKEN`, no E1–E9 live-fire pass recorded in this PR.
  Treat any future claim that "the tier-review gate caught X" as needing a
  fresh check that activation actually happened by then.
- The local merge-gate checks in `merge.sh`/`enforce_pr_workflow.sh` are
  **inactive by default** even after activation elsewhere — they require
  `tier_review.machine_user` to be explicitly set per-repo in
  `workflow.config.yaml`. A repo can have the daemon running and posting
  statuses while its own local merge gate stays dark if that config key is
  never set.
- The daemon deliberately does not filter PR comments by author (unlike
  `git-common.sh`'s trust filter for the review/eval artifacts) — in this
  deployment the agent self-merges, so the "trusted" comment author and the
  potential adversary are the same uid. The real defences are elsewhere (the
  judge call against the real diff; the post-nothing contract for non-tier-0)
  and don't depend on comment authorship at all.
