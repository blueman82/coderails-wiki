---
title: "Hook: comment_citation_gate.sh"
type: hook
created: 2026-07-08
last_updated: 2026-07-08
sources:
  - sources/pr_50_2026-07-07_comment-citation-gate.md
tags: [hook, pretooluse, comment-hygiene, citation, code-quality]
---

# comment_citation_gate.sh

`PreToolUse` hook (`Write`/`Edit`/`MultiEdit`) that blocks NEW code comments citing
a session-artifact label — a review finding ID, a plan step, a task/work-unit
tag, or similar shorthand that only makes sense inside the conversation that
produced it. A comment should state the constraint the code enforces, never
the conversation, PR review, or session artifact that produced it — those
labels rot the moment the session ends, because nobody with just the repo can
resolve "F4 fix" or "reviewer finding FH" back to anything.

Source: `coderails/hooks/scripts/comment_citation_gate.sh`
Registered in `coderails/hooks/hooks.json` under `PreToolUse` (Write/Edit/MultiEdit),
alongside [[no_edit_on_main]] — same event, same tool matcher.

## Event and mode

| Field | Value |
|---|---|
| Event | `PreToolUse` (`Write`\|`Edit`\|`MultiEdit`) |
| Mode | **block** — `permissionDecision: deny` on stdout, then `exit 0` (the `no_edit_on_main.sh` idiom, not `exit 2`) |
| Timeout | 5s (hooks.json) |

## Scope

Only comment-bearing **content fields** — `Edit`'s `new_string`, `Write`'s
`content`, and each `MultiEdit` edit's `new_string`. `.md` files are skipped
entirely by a suffix check before any content is inspected — markdown is
explicitly out of scope for this hook (prose citations in docs/plans are not
the "rots in code forever" problem this hook targets).

## Block condition

A denylist regex matched case-insensitively against the gathered content,
anchored on label **shape** (a following digit for `CHANGE`/`Task`, a word
boundary elsewhere) so schema/noise values like `P0`, `WU3=`, a bare `"WU3"`
JSON key, or the English phrase "CHANGE the default timeout" don't collide on
a bare substring match:

| Family | Pattern | Example blocked |
|---|---|---|
| `E#:` | `\bE[0-9]+:` | `// E3: handle the retry case` |
| `F#` fix/`:` /design | `\bF[0-9]+ (fix\|:\|design)` | `// F4 fix for the race` |
| `CHANGE B#`/`C#` | `CHANGE [BC][0-9]` | `// CHANGE B2 per review` |
| `Task A#` | `\bTask A[0-9]+\b` | `// implements Task A3` |
| `TA-I#` | `TA-I[0-9]+` | `// TA-I1 follow-up` |
| reviewer finding | `reviewer finding` | `// fixes reviewer finding FH` |
| `eval E#` | `eval E[0-9]+` | `// covers eval E2` |
| `WU#:` (label, not assignment) | `\bWU[0-9]+:` | `// WU3: the staging fix` |
| `C2` | `\bC2\b` | `// per C2's anti-stall design` |
| indirect artifact phrasing | `per the (plan\|design\|session)` | `// per the plan's step 2` |
| `per F#` | `per F[0-9]+` | `// per F4's finding` |

**`PR #NN` is a documented, deliberate survivor** — it is excluded from the
denylist because it resolves to a durable, checkable GitHub artifact (the PR
itself), unlike an internal review-round or task label that has no existence
outside the session transcript that generated it.

## Deny message

`permissionDecisionReason` echoes the exact matched substring back to the
model: *"Blocked: comment cites a session-artifact label (\<match\>). State the
constraint the code enforces, not the conversation/PR that produced it."* —
same "show the model exactly what tripped it" pattern as
[[destructive_bash_gate]]'s deny reasons.

## Relationship to other Write/Edit/MultiEdit gates

Registered alongside [[no_edit_on_main]] in the same `hooks.json` array entry
(`PreToolUse` matcher for `Write|Edit|MultiEdit`). The two gates check
unrelated things — `no_edit_on_main` is a branch/file-path allowlist,
this hook is a content-regex denylist — and both can independently deny the
same tool call.

## Known limitations

- **Denylist, not an exhaustive citation detector.** Only the enumerated label
  shapes above are caught; a novel or differently-formatted session-artifact
  citation (a Jira ticket ID, a Slack thread link, an ad-hoc "per today's
  discussion") is not recognised and will pass through uncaught. Best-effort,
  same honesty posture as [[destructive_bash_gate]]'s blocklists.
- **Only gates NEW content.** A comment already citing a session artifact
  before this hook existed is not retroactively flagged — this hook only
  fires on `Write`/`Edit`/`MultiEdit` tool calls going forward.
- **`.md` is a blanket carve-out**, not a narrower "only source comments"
  distinction — a markdown file embedding a fenced code block with a citation
  inside it is not scanned, since the suffix check short-circuits before
  content inspection.

## Wiki coverage gap (2026-07-08)

This hook was added 2026-07-07 (commit `125aec7`, PR #50 — see the
PR-number-collision note below) but had **zero** wiki coverage until this
page: absent from `index.md`'s hooks table, unmentioned anywhere else in the
vault. Surfaced by a full-vault wiki-lint health pass the same day this page
was authored; the root cause was that the PR which added the gate was never
wiki-ingested at the time. Closed by this page + the `index.md` hooks-table
row it accompanies. `AGENTS.md`'s own Part-1 hook event map also omits this
hook — a source-side gap, flagged here but not fixed (editing `AGENTS.md`'s
conventions is the maintainer's call, not a wiki-ingest action).

## ⚠️ PR-number collision

The PR that added this hook is **PR #50, merged 2026-07-07** (post the
2026-07-05 repo recreation described in [[repo-hosting]]) — a different PR
from the older, unrelated [[pr_50_planning-sequence-gate|PR #50]] (merged
2026-06-26, pre-recreation, planning-sequence gate in the writing-plans flow).
This page's source record uses the date-qualified filename
`sources/pr_50_2026-07-07_comment-citation-gate.md` per the convention
[[repo-hosting]] documents for exactly this collision class.

## See also

- [[no_edit_on_main]] — sibling gate on the same `PreToolUse (Write|Edit|MultiEdit)` event
- [[destructive_bash_gate]] — same denylist/best-effort posture, same "echo the match back" deny-message pattern, different tool (`Bash`, not `Write`/`Edit`/`MultiEdit`)
- [[enforcement-model]] — hooks vs. commands; this hook's honest limits (denylist not exhaustive)
- [[discipline-loop]] — full hook composition
- [[repo-hosting]] — PR-number collision convention this page follows
- [[pr_50_2026-07-07_comment-citation-gate]] — source record for this hook
