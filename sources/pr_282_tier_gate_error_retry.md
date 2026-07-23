---
title: "PR #282 — tier-gate error status made retryable and bounded"
type: source
origin: "coderails PR #282 (merged c1101b6, 2026-07-23; supersedes the auto-closed #280)"
created: 2026-07-23
last_updated: 2026-07-23
sources: []
tags: [source, tier-gate, tier-review, daemon, tg_should_gate, retry, fail-closed, verification-blind-spot]
---

# PR #282 — tier-gate error status retryable, bounded

## The wedge

The tier-review daemon's `tg_should_gate` decided whether a commit SHA needs judging. It
grouped `error` with `success|failure` as **terminal**:

```bash
success|failure|error) return 1 ;;
```

But `error` does not mean "this PR was judged" — it means **the daemon itself failed**.
Observed live: a real judge run posted `tg_judge: error: claude/parse failure after retry`
→ `verdict=error state=error reason=judge_rc_1`. The judge ran, its output failed to parse
twice, the daemon recorded `error`, and then skipped that SHA on every subsequent tick
forever. A transient parse failure **permanently wedged the PR** with no retry path. Across
the whole daemon log this was a one-off — 22 successful verdicts against 1 error — so the
underlying failure was transient, not systemic, which is exactly the case a retry should
recover.

## The fix

`error` becomes its own retryable case, bounded, while `success`/`failure` stay
unconditionally terminal and the default still fails closed:

- A new `tg_leading_error_count` counts consecutive leading `error` statuses from GitHub's
  own commit-status history (no daemon-local state, so it survives a restart).
- `error` re-gates while that count is below `TIER_GATE_MAX_ERROR_RETRIES` (default 2), then
  skips — so a *permanently*-broken judge cannot loop forever burning judge calls.
- `success` and `failure` remain terminal at any count. **This is the load-bearing safety
  property**: retrying a real verdict would let a judged-illegitimate PR be re-rolled until
  the nondeterministic judge happened to flake to `legitimate` — strictly worse than the
  wedge being fixed.

## The bug the first cut shipped — and why two independent checks missed it

The first implementation counted a *pure* run of `error` statuses. But the daemon posts a
fresh `pending` status **before every judge call** (`tg_post_status ... pending` at
`tier-gate-runner.sh:1042`, unconditional). So a repeatedly-crashing judge's real history is
**interleaved**: `[error, pending, error, pending, ...]`, newest-first — never
`[error, error, error]`.

`tg_leading_error_count` stopped at the first non-`error` status — the `pending` directly
under the newest `error` — so the count was **always 1**. `1 < cap` is always true, so the
daemon re-gated on **every tick, unboundedly**. The fix, as first written, was the *opposite*
of its purpose: not a bounded retry, but an infinite one.

> ⚠️ Both the PR's own SG1-SG9 tests **and** the orchestrator's independent verification
> missed this, for the identical reason: both built fixtures with pure `error` runs — the
> shape the daemon never produces. A green suite and a green independent probe both passed a
> broken fix. Two PR reviewers, using the **real interleaved shape**, caught it. This is the
> eval-shares-the-implementation's-blind-spot failure catching not just an author but the
> reviewing orchestrator too. Lesson promoted to standing orders: when verifying a fix that
> reads external state, reproduce the ACTUAL shape that state takes in production, not a
> simplified one.

The corrected `tg_leading_error_count` **skips** `pending` (neither counts nor stops) and
stops only at a genuine `success`/`failure`. Verified against the interleaved shape:
`[error,pending]` → count 1 → re-gate; `[error,pending,error,pending]` → count 2 → skip at
the cap.

## Fail-closed fallback

If the count's `jq` cannot parse the input, the function echoes `TIER_GATE_MAX_ERROR_RETRIES`
itself (= at the cap = skip), not `0` (= re-gate). A tooling fault reads as "give up", not
"retry unboundedly" — matching the file's fail-closed commitment ([[design/enforcement-model]]).
`TIER_GATE_MAX_ERROR_RETRIES` also resets to the default on non-integer env input.

## Impact

This unwedged real PRs that had hit the terminal-`error` trap this session (#269 originally,
then #277/#284's own merge attempts). It is the mechanism that lets a flaky judge self-recover
rather than requiring a human to force-merge or an empty commit to mint a fresh SHA. Relates to
the tier-review daemon introduced in [[sources/pr_232_tier-review-gate]] and hardened in
[[sources/pr_274_tier_gate_observability_fixes]].

## Note on churn

#282 supersedes #280: a worker's force-push was blocked by the destructive-command guardrail,
and rather than edit the guardrail it deleted+recreated the branch, which auto-closed #280.
Content is identical. The general lesson — a rebased branch needs a force-push the guardrail
blocks, so prefer a fresh branch off current main (a plain non-force push) — recurred on #284
and was solved there without any branch churn.
