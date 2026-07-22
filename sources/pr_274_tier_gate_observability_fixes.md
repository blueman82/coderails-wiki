---
title: "PR #274 — Tier-gate daemon: six observability fixes, and a MERGED ≠ DEPLOYED install-time repeat"
type: source
created: 2026-07-22
last_updated: 2026-07-22
sources: [scripts/tier-gate/tier-gate-runner.sh, scripts/tier-gate/install.sh, scripts/tier-gate/com.coderails.tier-gate.plist.template, scripts/tier-gate/coderails-tier-gate.conf, hooks/scripts/tests/tier_gate_runner.test.sh]
tags: [tier-review, observability, heartbeat, merged-not-deployed, launchd, log-rotation, silent-failure]
---

# PR #274 — Tier-gate daemon: six observability fixes, and a MERGED ≠ DEPLOYED install-time repeat

<!-- Ingested by /wiki-ingest after merge. This is an immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR number | #274 |
| Merged | 2026-07-22 (22:25:17Z) |
| Merge SHA | `e5c26fa` |
| JIRA ticket | — |

## Summary

[[pr_232_tier-review-gate|PR #232]]'s root-owned tier-gate daemon
(`scripts/tier-gate/tier-gate-runner.sh`) could fail silently in six
distinct ways — a rejected GitHub status POST logged as success, a failed
GitHub read indistinguishable from an empty result, an `rc` swallowed by a
pipe, and no per-tick heartbeat, so a frozen daemon looked identical to an
idle-but-healthy one. This PR closes all six, test-first where the change
was testable (design reviewed via Fable first). (verified —
`gh pr view 274 --json files,body`, `git show e5c26fa` on every file cited
below)

## The six fixes

1. **`tg_post_status` never checked its own HTTP response code.** `curl`
   exits 0 on a 401/403/422 — a rejected status POST returned success to the
   caller, so `tg_gate_pr` logged a false `state=success` while GitHub held
   no status at all. Worse: if the *terminal* post silently failed, the
   preceding `pending` post was left stale, and `tg_should_gate` would skip
   re-gating that PR until the 720s TTL expired — a false-success write and
   a 12-minute gate hole from the same defect. Fixed by capturing
   `-w '\n%{http_code}'`, checking for 2xx, and logging a named
   `status_post_failed` line instead of a false terminal state on rejection.
   Applied to the `legitimate` and `illegitimate|insufficient` verdict
   branches. (verified — `tier-gate-runner.sh` diff, `tg_post_status`)
2. **`tg_gh_get` failed silently** on a transport error or non-2xx — empty
   output was indistinguishable from "zero PRs, nothing to do." Both paths
   now write one `tg_log` line each: rc/http-code, the URL, and (for HTTP
   failures) the first ~200 bytes of the response body flattened to one
   line, which self-identifies rate-limiting. Never logs the token — no
   return path's rc or stdout changed, so the fail-closed contract is
   unaffected. (verified — `tg_gh_get`, lines 189/200 in the merged file)
3. **A pipe swallowed `tg_gh_get`'s rc** in `tg_open_prs` and
   `tg_pr_head_sha` (`tg_gh_get ... | jq` binds the pipeline's exit status to
   `jq`, not the fetch). Both restructured capture-first — the same idiom
   the file already documented as GATE-CRITICAL in `tg_commit_statuses` —
   so a failed fetch is now distinguishable from a genuinely empty result.
4. **Per-tick heartbeat + log rotation.** New `tg_log()` helper prefixes
   every line with a UTC timestamp. `tg_poll_once` now writes exactly one
   line per cycle — `tick: prs=N` on a successful fetch (N may be 0; that's
   not a failure) or `tick: pr_fetch=FAILED` (only expressible given fix 3)
   — and routes every `gated:`/`skip:` summary through the same helper, so
   the whole log carries timestamps. A frozen log is now itself an alarm: a
   healthy idle daemon writes a `tick:` line every `StartInterval` seconds,
   so silence for longer than that is diagnostic on its own. New
   `scripts/tier-gate/coderails-tier-gate.conf`, installed to
   `/etc/newsyslog.d/`, rotates the log at 1MB keeping 5 gzip generations.
5. **plist template.** Added `LimitLoadToSessionType=System` — without it,
   a legacy unprivileged `launchctl load` of the same plist registers a
   second, permanently-`EX_CONFIG` ghost job in the `gui/<uid>` domain that
   `launchctl list` still shows, alongside the real `system/`-domain job
   ([[enforcement-model|see the ghost incident below]]). `StartInterval`
   corrected 300→60 — the template had drifted from the value actually
   running live, with a comment recorded noting 60 is intentional pending
   rate-limit evidence, not an accidental tightening.
6. **`install.sh` main-guard.** Before bootstrap: resolves the console uid
   (`stat -f%u /dev/console`) and boots out any `gui/$uid`/`user/$uid` ghost
   registration, with a named warning. After bootstrap: asserts the
   `system/...` job registered and no per-user ghost survived, failing
   loudly rather than printing success regardless. Installs the newsyslog
   conf and prints the canonical health-check command in the completion
   message.

## Tests

Test-first for the shell-logic changes (fix 1's 2xx check +
`status_post_failed` path; fix 3's capture-first rc distinction). Fixes 5/6
verify by inspection (plist XML, a bootstrap-time shell guard).

| suite | pass |
|---|---|
| tier_gate_runner | 135 (124 baseline + 11 new) |
| tier_gate_prefilter | 20 |
| tier_gate_judge_auth | 10 |
| tier_gate_install | 66 |
| merge_tier_review_gate | 26 |
| **total** | **257 / 0 fail** |

## Scope note the PR called out for its own reviewer

Fix 1's rc check covers only the two *terminal* verdict branches
(`legitimate`, `illegitimate|insufficient`). The `pending` post and the
error/failure posts elsewhere in `tg_gate_pr` still ignore the post rc, so
the "log says posted, GitHub has nothing" class survives on those paths —
a deliberate partial fix per the brief's scope, called out in the PR body
rather than silently left. Extending it to `illegitimate|insufficient`
(beyond the brief's `success`-only scope) was flagged the same way.

## Needed a human force-merge — the self-edit denylist fired as designed

This PR touches `scripts/tier-gate/`, which is on the daemon's own
[[tier-gate-path-denylist-dashboard_2026-07-21|self-edit denylist]]. The
daemon posted `verdict=self_edit` and refused to judge it — correct
behaviour, not a defect: a change to the judge's own source must not be
self-approvable by the mechanism it's changing. Merged via human
force-merge, the same disposition [[pr_256_runner-transcript-persistence|PR
#256]] documents for the same denylist entry.

## MERGED ≠ DEPLOYED, fired twice on this PR itself

The tier-gate daemon runs from an **installed root copy** at
`/etc/coderails-tier-gate/tier-gate-runner.sh`, promoted there by
`install.sh` — never from `origin/main` directly. Merging this PR did not,
by itself, change what the live daemon executes; only re-running the
installer does. This is the same drift class
[[enforcement-model]]'s "A source-tree property is not a running-process
property" section already documents for [[routines|the routine-sweeper's
57-commit-stale checkout]] (PRs #260/#263) — a third instance, this time
against a root-owned install path rather than a working-checkout `launchd`
target.

It fired concretely during this PR's own rollout: the **first** `install.sh`
run promoted **stale** code — the checkout it ran from was one commit
behind `origin/main`, missing #274 itself — so it installed a runner with
an updated comment header but **none** of the six functional fixes, while
still printing "INSTALL COMPLETE." This was caught only by checking the
live log's actual line format (still untimestamped, pre-fix-4 shape)
against what a genuine deploy should look like — not by trusting the
installer's own success message. Re-running after pulling the checkout
current fixed it. *(inferred from the orchestrating session's own account,
not independently re-derived here — the specific before/after log lines and
byte-diff confirmation were reported by a sibling agent in this session
with tool access to the running daemon, not captured as a durable artifact
this page can re-verify from source alone.)*

A second, unrelated confound compounded the confusion during debugging: a
**ghost launchd registration** in the `gui/501` domain (left over from a
prior debugging session's unprivileged `launchctl load`) reported exit 78
(`EX_CONFIG`) in a plain `launchctl list`, and had misled two separate
earlier debugging sessions into believing the daemon itself was broken when
the real `system/`-domain daemon was fine the whole time. Removed via
`launchctl bootout gui/501/...`; fix 5's `LimitLoadToSessionType=System`
prevents a new one registering the same way.

## Security review

A security review (opus-level, this session) traced all seven new log
lines this PR adds and confirmed none can emit the root-held
`CLAUDE_CODE_OAUTH_TOKEN` or the GitHub credential — token-leak surface
clean, with a `PS1`-style empirical test cited as the proof method. One Low
finding: the ~200-byte response-body snippet (fix 2) lands in a `0644` log
file, readable by any local user; no change required, since the daemon's
own [[pr_232_tier-review-gate|credentials file]] is separately `0600`-root.
*(inferred from the orchestrating session's report; not re-derived
independently in this ingest.)*

## Wiki pages updated

- [[enforcement-model]] — new instance in the "A source-tree property is not
  a running-process property" section (third recorded case, alongside
  [[routines]] and [[pr_260_263_dashboard-security-review]])
- [[routines]] — cross-reference to this page as a sibling stale-deploy case
  against a root install path rather than a working checkout

## Caveats / gotchas

- The daemon's live-deployed state is a **point-in-time fact**, not a
  regression lock — the next `install.sh` run against a stale local checkout
  can reintroduce the exact gap this PR's own rollout just demonstrated.
  Fix 6's post-bootstrap assertions catch the ghost-registration half of
  that risk mechanically; they do **not** catch "the checkout `install.sh`
  ran from was behind `origin/main`" — that half was caught by a human/agent
  manually diffing log output, not by any assertion in the installer
  itself.
- The Fix 1 partial-scope gap (pending/error posts still ignore rc) is a
  documented, in-PR-acknowledged residual, not a miss this ingest is
  flagging fresh.
