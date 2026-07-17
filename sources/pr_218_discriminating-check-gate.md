---
title: "PR 218 — freeze-time discriminating-check gate"
type: source
created: 2026-07-17
last_updated: 2026-07-17
sources:
  - commands/post-evals.md
  - scripts/post_evals.sh
  - hooks/scripts/tests/discriminate.test.sh
  - skills/task-evals/SKILL.md
tags: [source, task-evals, post-evals, discriminating-check, fixtures, freeze-time, negative-control]
---

# PR 218 — freeze-time discriminating-check gate

<!-- Ingested by /wiki-ingest after merge. This is an immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR number | #218 |
| Branch | `worktree-discriminating-check-gate` |
| Merged | 2026-07-17 |
| Merge SHA | `81c8857` |
| Follow-up fix | `335a382` — same-day review-findings fix, folded into this page rather than a separate source (see "Follow-up fix" below) |

## Summary

Adds `post_evals::validate_discriminating` + `post_evals::_run_formula` to `scripts/post_evals.sh`: a freeze-time gate that pipes an eval's `fixtures.good` and `fixtures.bad` into the check's formula and rejects any check where the two inputs don't produce opposite exit codes (good exits 0, bad exits non-zero). Catches the "frozen check is itself broken" class — a scripted check incapable of ever passing (false alarm) or ever failing (vacuous) — which the pre-existing `negative_control` structural check does not catch, because that check only proves a control command's *text* differs from `cmd`, not that the check's formula actually discriminates between real pass and real fail input.

Adds an optional `fixtures` field to the scripted-eval schema: `{ good, bad, formula? }`. `formula` defaults to the substring of `cmd` after the last top-level pipe when omitted. Wired into `commands/post-evals.md` as Step 3b, run after Step 3's `validate-structure` and before Step 4's result computation — abort-on-nonzero, same posture as the existing structural gate.

Grandfathering is explicit and load-bearing: an eval with no `fixtures` field is validated exactly as it was before this gate existed — zero behaviour change. `fixtures` is opt-in per eval, never retroactive; freezing this gate does not retroactively validate any pre-existing `evals.json`.

## Motivating failure

Loop 8b69e779's real, previously-shipped bug: an awk formula piped `39/39 suites passed` output through `-F'[ /]'` field-splitting. That split pattern put `$(NF-2)` on the literal word `"suites"`, not the numerator, so the formula exited 1 unconditionally — on a genuine pass (39/39) and a genuine fail (18/40) alike. Both fixtures produced the same exit code; the check was structurally present (had a `cmd`, had a `negative_control`) but could never discriminate a real pass from a real fail. This gate's case 1 test (`hooks/scripts/tests/discriminate.test.sh`) is a direct regression lock against that exact broken formula.

## Design fork: formula-against-fixtures vs. real-surface execution

Two designs were on the table. Real-surface execution — actually running the check against the real target — was rejected in favour of formula-against-fixtures, because at freeze time a not-yet-built surface and a genuinely broken check both exit non-zero **indistinguishably**: a gate that ran the real surface at freeze could not tell "this check is broken" apart from "this feature legitimately doesn't exist yet." Piping synthetic `fixtures.good`/`fixtures.bad` text into the formula sidesteps that ambiguity entirely — the formula is exercised against known inputs with known expected outcomes, independent of whether the real implementation exists yet.

## Design history: a wrongly-concluded hard-stop, overturned

An earlier point in the building loop concluded the feature was "unbuildable" on the premise that ~91% of `negative_control` values in the existing corpus looked like prose rather than machine-checkable commands — if most evals aren't real scripted checks, a fixtures-based gate would have near-zero surface to validate. That 91% figure came from a broken classifier; the real figure, re-measured, is closer to 46%. The stop was overturned by a `/coderails:disconfirm` pass plus an adversarial fable-5 red-team review, both of which challenged the classifier itself rather than accepting its output. Worth recording as a caution: a hard-stop premised on a measured corpus statistic should get the same "is this measurement broken" scrutiny as any other claim before it's allowed to kill a feature.

## Mechanism (`scripts/post_evals.sh`)

- **`post_evals::_run_formula <formula> <input>`** — pipes `<input>` into `bash -c <formula>`, wrapped in `perl -e 'alarm shift; exec ...' 10` for a 10-second timeout (chosen over hand-rolled bash job-control timing to avoid a race). Echoes the raw exit code. 142 (128+SIGALRM) signals timeout; 127 signals command-not-found — both are environmental outcomes the caller must not read as a discrimination verdict.
- **`post_evals::validate_discriminating <evals_json_path>`** — for every `mode == "scripted"` eval carrying a `fixtures` object: validates `fixtures` is an object (not a string/number, which would otherwise silently degrade every field extraction to `""` and misreport as "non-discriminating"); requires **both** `good` and `bad` non-empty (an author supplying only `good`+`formula` gets `bad=""` by default, and proving discrimination against an empty string nobody wrote is the unsafe accept direction — rejected explicitly rather than silently proving the wrong thing); resolves the formula (`fixtures.formula` if present, else the text after the last top-level pipe in `cmd`, fail-closed with an actionable message if neither is available — the split is text-position based, not shell-aware, so a quoted pipe inside e.g. an `awk` pattern forces an explicit `fixtures.formula`); runs the formula against both fixtures via `_run_formula`; and requires `good_rc == 0 && bad_rc != 0`.

### Env-guard: distinguishing environmental crashes from discrimination results

Exit 127 (command not found) and exit 142 (the timeout sentinel) get their own distinct rejection messages, checked before the discrimination logic. The follow-up fix (`335a382`, same day) broadened this to exit 126 (permission denied) and any exit ≥128 (signal deaths — 137=SIGKILL, 139=SIGSEGV, etc.), ordered **after** the 142 check so the timeout message stays distinct even though 142 is itself ≥128 — the source comments this ordering as load-bearing. Rationale stated inline: a formula that *crashes* on the bad-fixture leg is environmental-suspect, not a valid content fail, and without this check it would fall through into the accept path (`good_rc=0 && bad_rc!=0`) and read as a legitimate "bad correctly fails" result when it actually just crashed.

## New `fixtures` schema field

```json
"fixtures": { "good": "<sample stdin that SHOULD pass>", "bad": "<sample stdin that SHOULD fail>", "formula": "<optional: defaults to the segment after the last top-level pipe in cmd>" }
```

Optional, scripted-mode only. Documented in `skills/task-evals/SKILL.md`'s schema block alongside the pre-existing eval fields.

## Follow-up fix (`335a382`, same day, folded into this record)

Four PR-review findings fixed same-day, before this page's own ingest:
1. Wired `validate-discriminating` into `commands/post-evals.md` as Step 3b (was built but not yet called from the command chain).
2. Require `good` **and** `bad` together — reject if either is missing, closing the unsafe-accept-on-empty-default gap described above.
3. Broadened the env-guard from 127/142-only to also cover 126 and any ≥128 signal death.
4. Reject malformed `fixtures` (non-object) with a distinct message instead of letting it degrade into a false "non-discriminating" verdict.

## Test coverage (`hooks/scripts/tests/discriminate.test.sh`, 17 cases + CLI dispatch)

Notable cases beyond the straightforward accept/reject pairs:
- **Case 1** — regression lock for the exact loop-8b69e779 broken awk formula (both fixtures exit 1) → rejected.
- **Case 3** — vacuous-pass rejection: a formula that exits 0 on both fixtures (e.g. bare `cat`) is equally non-discriminating and rejected with a distinct "both exit 0" message from case 1's "both exit 1".
- **Case 4** — grandfathering: no `fixtures` field validates exactly as before this gate existed.
- **Case 5 / 17** — formula-derivation fail-closed half (no pipe, no explicit formula → reject) and happy-path half (derives from the last pipe segment, discriminates correctly → accept) are separately exercised; case 17 was added specifically because every other fixtures block in the suite supplies `fixtures.formula` explicitly, so the actual derivation-succeeds branch had no direct coverage until this case.
- **Case 8** — an *inverted* formula (good fails, bad passes — opposite exit codes but the wrong way round) gets a message distinct from "non-discriminating," since it's a different defect class (same-exit-code-both-ways vs. correctly-opposite-but-backwards).
- **Cases 11/12** — `bad` omitted / `good` omitted, each constructed so the omitted field's empty-string default would otherwise produce the exact unsafe-accept shape (`good_rc=0, bad_rc≠0`) without ever checking a real fixture the author wrote — proof against the gap the follow-up fix closed, not just a presence check.
- **Cases 13–15** — env-guard broadening: exit 126, exit 137 (SIGKILL), and a proof that the pre-existing 142-timeout message still fires distinctly after the ≥128 check was added (ordering regression guard).
- **Case 16** — malformed (non-object) `fixtures` rejected with a message containing "object", not silently degraded.

## Honest boundary (stated prominently in `skills/task-evals/SKILL.md`, do not overclaim)

This gate validates only checks that **carry** `fixtures`. It does not retroactively validate the ~46% of the existing corpus that already has scripted checks without fixtures, and it does nothing at all for the remaining prose/judgement-mode evals. Even where `fixtures` is present, a pass proves the formula **can discriminate between these two specific inputs** — it proves nothing about whether the formula tests the **right** claim, whether `cmd` and `fixtures.formula` stay in sync after later edits, or whether the two fixtures are representative of real pass/fail states. This closes the "never fails" (or "never passes") class of defect; it is not a general correctness proof of the check.

## Wiki pages updated

- [[task-evals-gate]] — added "Discriminating-check gate" section (mechanism, env-guard, honest boundary, design-fork rationale)
- [[task-evals]] — schema section now cross-references the optional `fixtures` field
- [[post-evals]] — Step 3b added to the seven-step (now eight) command sequence

## Caveats / gotchas

- The formula-derivation split (`"${cmd##*|}"`) is a **text-position** split, not shell-aware — a quoted pipe character inside an `awk`/`grep` pattern will land inside the quoted string if relied on implicitly; the author must supply `fixtures.formula` explicitly whenever `cmd` contains a quoted pipe.
- The env-guard ordering (142-timeout check before the ≥128 signal-death check) is load-bearing, not incidental — reordering would swallow the distinct timeout message since 142 also satisfies `>= 128`.
- This is purely a freeze-time author discipline gate — it runs inside `/coderails:post-evals` Step 3b, at the same point as the existing `validate-structure` check, not as a standalone CI step.
