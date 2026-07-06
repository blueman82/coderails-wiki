---
title: "Trust floor for artifact gates: repo permission, not OWNER-badge"
type: design
created: 2026-07-06
last_updated: 2026-07-06
sources:
  - sources/pr_7-10_task-evals-followups.md
  - sources/pr_11-14_gate-hardening-followups.md
  - sources/pr_21-22_loop2-suggestion-tier-followups.md
tags: [design, trust-floor, viewerPermission, comment-spoofing, artifact-gates, tempfile]
---

# Trust floor for artifact gates: repo permission, not OWNER-badge

This page is a map, not a second spec. The mechanism itself — the exact API
call, the permission-level list, the anti-spoof property — is documented
once, in [[merge]]'s "Trust floor for artifact gates: repo permission, not
OWNER-badge (PR #14)" section, and stays there. This page exists because the
concept is cited across 5+ pages with no single concept-level home before
now.

## Context

Both artifact gates (review and eval) read PR comments through a shared
trust filter before matching either gate's marker. That trust filter's rule
changed twice: [[pr_7-10_task-evals-followups|PR #8]] first closed
comment-spoofing with an `author_association == "OWNER"` check, which then
turned out to fail closed on org-owned repos (the same authenticated
identity's own comments there carry `MEMBER`/`COLLABORATOR`, never `OWNER`).
[[pr_11-14_gate-hardening-followups|PR #14]] (`7c1dd19`, 2026-07-06, owner
directive) replaced the OWNER-badge check with a repo-permission check,
closing that gap. This history is flagged in the wiki's own 2026-07-06
full-vault-resweep entry (`log.md`) as a concept cited across
[[review-artifact-seam]], [[task-evals-gate]], [[merge]], and the two
PR-follow-up source pages, with no dedicated concept page — hence this page.

## The rule

Comment-author trust for both artifact gates is a repo-permission check
(`gh repo view --json viewerPermission`, requiring `ADMIN`/`MAINTAIN`/`WRITE`)
plus a login-match anti-spoof check, not an `author_association == "OWNER"`
badge check (which failed closed on org-owned repos). Introduced by
[[pr_11-14_gate-hardening-followups|PR #14]] (`7c1dd19`, 2026-07-06). See
[[merge]]'s "Trust floor for artifact gates" section for the full mechanism
— the exact call shape, the permission-level list, and the anti-spoof
property all live there as the single source of truth.

## Where it is enforced

Shared by both gate consumers through `scripts/lib/git-common.sh`'s trust
filter (see [[merge]] for the specific function names). The two consumers:

- [[review-artifact-seam]] — the review-artifact gate
- [[task-evals-gate]] — the eval-artifact gate

Neither page restates the mechanism; both point back to [[merge]].

## Known caveats / edge cases

A resolvable-but-insufficient permission (e.g. `READ`) is treated as "no
trusted comment found," not a fetch failure; an actual permission-lookup
failure fails closed, same posture as an identity-fetch failure. This
distinction lives in [[merge]] and is not repeated here.

## Failure-reason vocabulary (extended PR #21)

`PR_TRUST_FETCH_FAIL_REASON`, the variable `merge.sh`'s error-message split
switches on, carries four values: `identity` (PR #14), `permission` (PR #14),
`comments` (generic gh-fetch failure), and `tempfile`
([[pr_21-22_loop2-suggestion-tier-followups|PR #21]], merged 2026-07-06,
`d0e4c5a`). `tempfile` fires when `pr::_trusted_comment_bodies_or_fail`'s own
`mktemp` call fails — this precedes any `gh` API call, so `merge.sh` gives it
a dedicated case arm on both gates rather than folding it into the generic
"GitHub fetch failed" fallback, which would misname the cause. The variable
was already being set for this case before PR #21; only `merge.sh`'s
consumer-side `case` was missing the arm. Full detail in [[merge]]'s
"Error-message split for fetch failures" section.

## See also

- [[merge]] — **SSOT** for the mechanism itself (the `viewerPermission` call,
  the permission-level list, the anti-spoof property, the error-message
  split for fetch failures)
- [[review-artifact-seam]] — one of the two gates this trust filter guards
- [[task-evals-gate]] — the other gate this trust filter guards
- [[pr_11-14_gate-hardening-followups]] — PR #14: widens the trust floor to
  a repo-permission check, closing the org-repo limitation; PR #11 (same
  cluster) fixes the loop-scope tier-0 NO-GO precedence
- [[pr_7-10_task-evals-followups]] — PR #8: the original comment-spoofing
  closure (`author_association == "OWNER"`) that PR #14 later widened
