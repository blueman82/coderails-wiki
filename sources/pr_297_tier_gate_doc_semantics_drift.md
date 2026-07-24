---
title: "PR #297 — docs/TIER-GATE.md had the tier scale inverted"
type: source
created: 2026-07-24
last_updated: 2026-07-24
sources:
  - docs/TIER-GATE.md
  - scripts/tier-gate/judge-prompt.md
  - scripts/tier-gate/tier-gate-runner.sh
tags: [source, tier-gate, docs-drift, task-evals, ssot, verdict-taxonomy]
---

# PR #297 — docs/TIER-GATE.md had the tier scale inverted

<!-- Ingested by /wiki-ingest after merge. This is an immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR number | #297 |
| Branch | `docs/tier-gate-tier-semantics-drift` |
| Merged | 2026-07-24 (`2026-07-24T08:34:50Z`) |
| Merge commit | `0b7262d176259f8ce520ee54dc5c3abb1095b045` |
| Head commit | `4522768a66111ef4fa55edfc07285bd34db8db91` |
| JIRA ticket | — |

## Summary

Single-file docs fix. `docs/TIER-GATE.md` described the tier scale **backwards**
relative to the predicates the daemon actually judges against — tier 0 as
"user-facing changes, architecture, API changes, security, breaking changes"
and tier 2 as "risk-free changes (comments, formatting, trivial config)". The
authoritative `scripts/tier-gate/judge-prompt.md` says the opposite: tier 0 is
the **exempt** tier and tier 2 is the **full suite**.

The drift was found and recorded by this vault's own 2026-07-22 docs-drift pass
(see `log.md`, "Tier definitions are inverted") before any PR existed. #297 is
the repo doc catching up.

## Why this mattered more than a typo

The consequence stated in the PR is concrete and asymmetric: **a contributor
following `TIER-GATE.md` would claim exactly the wrong tier and be rejected by
the daemon.** Worse, the doc's *security* framing was inverted with it — it
described the attack as forging tier 2 onto a tier-0 change, when the real
attack runs **downward**: claiming tier 0 buys an exemption from the eval suite
that the change has not earned. A doc that inverts the direction of the attack
teaches the wrong threat model, not just the wrong label.

## The corrected semantics

Unchanged from what [[task-evals]] and [[task-evals-gate]] already carried
correctly — this PR did not redefine anything, it only stopped one file
contradicting the rest:

- **Tier 0 (exempt, justified)** — all three hold: single work-unit, no
  outward/irreversible surface, and an existing test or verify-criterion
  already covers the goal state. A user-facing change is never tier 0, however
  trivial.
- **Tier 1 (standard)** — anything above tier 0 not meeting the tier-2
  predicate. A user-facing surface on its own lands here.
- **Tier 2 (full suite)** — three or more independent work-units, **or** an
  irreversible/external surface (publish, deploy, migration, data deletion,
  external send).

## The two genuinely new facts

Everything above was already in the wiki. Two things in #297 were not:

1. **`scripts/tier-gate/judge-prompt.md` is named in the doc as the
   authoritative statement of the predicates** — "read it there when the two
   disagree". The tier predicates now have an explicitly designated SSOT, and
   it is the artifact the daemon actually judges against, not the prose doc a
   human reads. This is the same **executable-over-descriptive** precedence the
   vault applies elsewhere; #297 makes it explicit for tiers.

2. **The `tier-review` status posts on every outcome, and the verdict
   vocabulary is six values, not two.** The doc previously said the daemon
   "only posts when the LLM agrees the tier is legitimate", which was wrong in
   a way that makes a blocked PR look like a broken daemon:

| Verdict | State | Meaning |
|---|---|---|
| `legitimate` | `success` | the only verdict that satisfies the merge gate (`:1188`) |
| `illegitimate` | `failure` | rejected by the judge — also a **tier-0** claim over the byte cap (`:1119`, `:1152`) |
| `insufficient` | `failure` | judge can't decide either way on blind inputs — also a **tier-1/2** claim over the byte cap (`:1156`) |
| `self_edit` | `failure` | diff touches the tier-gate's own files; posted *before* judging, so it never reaches the judge (`:1056`) |
| `pending` | `pending` | posted unconditionally before **every** judge call, as a lease (`:1074`) |
| `error` | `error` | operational failure: files-fetch failed, empty file list, missing embedded `evals.json`, diff-fetch failed, empty diff, nonzero judge rc (`:1031`, `:1039`, `:1084`, `:1140`, `:1145`, `:1173`) |

   **Verified against `origin/main:scripts/tier-gate/tier-gate-runner.sh` at
   ingest, not taken from the PR's own doc** — deliberately, because a
   freshly-corrected doc is the least battle-tested text in the repo and this
   PR exists precisely because that file was wrong for weeks. The runner
   confirms it, and adds a distinction the doc doesn't draw: **only the first
   three are verdicts the judge can return** (`TIER_GATE_JUDGE_SCHEMA` at
   `:738` constrains its output to `enum: [legitimate, illegitimate,
   insufficient]`, re-validated at `:804`); `self_edit`, `pending` and `error`
   are **daemon-side dispositions the judge never emits**.

   **Every non-`legitimate` status blocks.** The size-cap asymmetry is the
   non-obvious part, and the runner states it in its own words (`:975-980`):
   over the cap a tier-0 claim is `illegitimate` because **"size IS the tier-0
   discriminator"**, while tier-1/2 gets `insufficient` because size is *not*
   the tier-2 discriminator — a file/line cap is a worse proxy for the judge's
   "how many work-units" question, so **tier 1/2 has no file/line cap at all**;
   only the byte cap, bounding the judge's input size, applies. `tg_prefilter`'s
   file/line caps run **only** when `tier == 0`, and the daemon never
   truncates-and-judges at any tier.

   The vault had observed `self_edit` empirically three times
   ([[pr_256_runner-transcript-persistence|#256]],
   [[pr_274_tier_gate_observability_fixes|#274]],
   [[pr_289_tier_gate_install_shared_root_warning|#289]]) but had never recorded
   the taxonomy it belongs to. #256's ingest noted `self_edit` posts as a
   `FAILURE` with an empty description, "indistinguishable from a broken build"
   — the taxonomy above is what that legibility complaint was missing.

## Wiki pages updated

- [[task-evals-gate]] — verdict taxonomy + judge-prompt-as-SSOT recorded
- [[task-evals]] — judge-prompt named as the predicate SSOT

## Caveats / gotchas

- **No wiki page had to be corrected.** The vault's tier semantics were already
  right (`skills/task-evals.md`, `design/task-evals-gate.md`), which is why this
  ingest adds knowledge rather than fixing errors. The only inverted phrasing
  anywhere in the vault is inside `log.md`'s 2026-07-22 drift entry, which is
  quoting the defect — correct in context, deliberately left.
- Documentation-only. No behaviour changed; the daemon judged the same way
  before and after.
- **The missing `design/tier-gate.md` evergreen page is still missing** — a
  standing lint finding from the 2026-07-24 full-vault pass, not addressed here.

## See also

- [[task-evals-gate]] — the tier predicates and the gates that read them
- [[task-evals]] — the writer-side tier rules
- [[pr_232_tier-review-gate]] — the daemon that posts `tier-review`
- [[tier-gate-path-denylist-dashboard_2026-07-21]] — the `self_edit` denylist
