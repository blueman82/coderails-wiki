---
title: "PR #207 + #209 — README/REFERENCE drift fix, and the nightly self-merging docs-sync routine that replaces the broken weekly one"
type: source
created: 2026-07-17
last_updated: 2026-07-17
sources: []
tags: [docs-sync, sync-docs, routines, self-merge, security, self-governance, dashboard-runner, escalation, wrong-surface-trap, agentic-os]
---

# PR #207 + #209 — docs-sync drift fix + nightly self-merging routine

Two PRs treated as one cluster: #207 fixed the documentation drift that had
accumulated (skill count, missing rows, sandbox workers), and #209 replaced
the routine that was supposed to catch that drift automatically but had been
dead for 9 days. The second PR only makes sense in light of the first —
#209's `docs-sync-nightly` routine exists specifically because the mechanism
#207 patched by hand had failed.

**Precision on "dead", because the imprecise version was the loop's own worst
error:** the routine did **not** fail silently. It went red on 2026-07-15 with
`failureClass: skill-missing` and escalated correctly, to a vault-note and a
macOS notification. Nobody was watching those channels. The distinction is the
whole lesson — see [[the wrong-surface trap]] below.

## PR metadata

| Field | Value |
|---|---|
| PR #207 | "docs/sync drift 2026 07 17", merged `70d0fe0` (dc4f3bb tree, 2026-07-17T10:10:51Z) |
| PR #209 | "feat/docs sync nightly routine", merged `dc4f3bb` (2026-07-17T10:38:19Z) |
| Files (#207) | `README.md`, `docs/REFERENCE.md`, `instructions/self-checking-discipline.md` |
| Files (#209) | `skills/docs-sync/SKILL.md`, `docs/routines.md`, `examples/dashboard-config.json`, `hooks/scripts/tests/docs_sync_routine.test.sh` |

## Summary

**PR #207** `(verified, gh pr diff 207)`: README skill count 34→35; five
missing skill rows added (`dashboard`, `fable-mode`, `sync-docs`,
`verify-merged-pr`, `workflow-audit`); the `offload_push_guard.sh` hook row
added; a new "Sandboxed workers" README section documenting
`scripts/sandbox/*` (from PR #199 — sandbox worker containment, not part of
this cluster, no wiki page yet); removed the legacy "started as two separate
plugins (`workflow-tools` and `claude-guardrails`)" origin story; removed a
stale "(claude-guardrails)" title parenthetical.

**PR #209** `(verified, gh pr diff 209, skills/docs-sync/SKILL.md read in full)`:
ships a **new, separate** skill `docs-sync` (`skills/docs-sync/SKILL.md`) —
distinct from the pre-existing `sync-docs` skill it invokes for its audit
step — wired as a `bypass`-profile, nightly routine named
`sync-docs-nightly` (button `sync-docs`, renamed from `docs-sync` in
`f3ef802` mid-PR). The routine audits git-tracked docs every night and, only
if drift is found, edits/pushes/reviews/self-merges a fix with **no human in
the loop**. `docs/routines.md` gets a full new section; the shipped test
file `docs_sync_routine.test.sh` regression-locks the SKILL.md prose against
silent rot.

**Naming — three distinct strings, easy to conflate:**
- `sync-docs` — the pre-existing, older skill (imported in-repo 2026-07-10,
  `f488ca5`) that does the actual drift *audit*. `[[sync-docs]]`.
- `docs-sync` — the new skill this PR ships (`skills/docs-sync/SKILL.md`)
  that *invokes* `sync-docs`'s audit as one step of a larger self-merge
  pipeline. `[[docs-sync]]`.
- `sync-docs-nightly` — the routine/config entry name; `buttonRef: sync-docs`
  (the button, renamed from `docs-sync` to `sync-docs` mid-PR, `f3ef802`).

## Finding 1 — root cause of the drift, WITH a correction on how it was found

The `sync-docs-weekly` routine ran green on 2026-07-08, went red on
2026-07-15 with `failureClass: skill-missing`, and never ran again
`(verified, coderails-wiki/dashboard-runs/sync-docs-weekly.md, read in full)`:

```
## [2026-07-08] run 0c718d27bccb7b0b — green
succeeded

## [2026-07-15] run 2328c240b5f9f2ff — red
Failure class: skill-missing
Reason: Referenced skill not found at /Users/harrison/.claude/skills/sync-docs/SKILL.md
```

Its `foreignSkillPath` pointed at a path that never existed. The real skill
had already been imported in-repo (`skills/sync-docs/SKILL.md`, commit
`f488ca5`, 2026-07-10) — a personal-plugin path referenced a location the
skill had since moved away from. Everything merged after 2026-07-15 drifted
unaudited, including PR #199's sandbox workers (which #207 had to
hand-document because the automated check never ran).

**The escalation actually worked — an earlier claim to the contrary was
false and is retracted here on purpose, because the retraction is the more
useful lesson than either version alone.** During this cluster's own
investigation, a first pass concluded the routine "failed silently and
never escalated," based on grepping
`~/.claude/coderails-dashboard/routines/sweeper.log` (launchd stdout/stderr)
and finding no obvious trace. **That claim was wrong and the wrong surface
was the reason**: `sweeper.log` is only "did the job even run" —
`escalate.ts:55-68` (`writeRunNote`) writes structured run-notes to
`config.wikiPaths[0]/dashboard-runs/<routine>.md` and fires a synchronous
macOS notification via `osascript` on every escalation `(verified,
skills/dashboard/runner/src/escalate.ts read directly)`. The 2026-07-15 red
run-note above IS that escalation, sitting exactly where the design says it
should. Checking only the launchd log and concluding "silent failure" is
the trap: it's the wrong surface for this question, and it produces a
confident, false negative.

**The real gap, stated precisely: escalation fired correctly, to a channel
nobody was watching.** A vault-note file and a transient `osascript` popup
are not equivalent to something landing in front of the routine's owner day
to day. For an unattended routine, "escalated to an unread channel" and
"failed silently" are operationally indistinguishable outcomes even though
mechanically they are not the same failure. Both the wrong claim and the
correction are recorded here because the wrong-surface trap (grep the
convenient log, not the log the design actually specifies) is the reusable
lesson, not just the eventual right answer.

## Finding 2 — config validation checks shape, not existence

`skills/dashboard/lib/src/config.ts:102-111` (verified — read directly):

```ts
if (routine.foreignSkillPath !== undefined) {
  if (typeof routine.foreignSkillPath !== "string" || routine.foreignSkillPath.length === 0) {
    throw new ConfigError(`Routine "${routine.name}" foreignSkillPath must be a non-empty string`);
  }
  if (!isAbsolute(routine.foreignSkillPath)) {
    throw new ConfigError(`Routine "${routine.name}" has relative foreignSkillPath (must be absolute): ${routine.foreignSkillPath}`);
  }
}
```

Non-empty and absolute are checked; existence on disk is not. A dead path
loads clean at config-load time and only fails later, per-run, at
`runner/src/sweep.ts:243` (`checkForeignSkillExists`) — verified directly,
same shape as the escalation site above. The structural fix this PR takes:
an in-repo skill needs **no** `foreignSkillPath` at all (same pattern as
`[[loop-retro-promotion]]`), so there is no path left to rot. As of this PR,
zero of the five shipped example routines use `foreignSkillPath`.

## Finding 3 — the `maxAgeSeconds` trap

`maxAgeSeconds` is a manual field on `expectedArtifact`, not derived from
`cadence` `(verified, config.ts:76-79 defines the field with no
cadence-linkage; examples/dashboard-config.json's shipped values confirm no
derivation exists)`. Simply flipping `cadence: "weekly"` → `"nightly"`
while leaving the old weekly 691200s (8-day) freshness bar would let a
routine dead for a full week still read as "fresh" — silently
reintroducing the exact bug this PR fixes. The shipped `sync-docs-nightly`
block in `examples/dashboard-config.json` (verified, read directly):

```json
{
  "name": "sync-docs-nightly",
  "label": "Docs Sync (nightly)",
  "buttonRef": "sync-docs",
  "cadence": "nightly",
  "expectedArtifact": {
    "artifactPath": "/path/to/.claude/coderails-dashboard/routines/docs-sync/run-{date}.log",
    "maxAgeSeconds": 129600,
    "predicate": { "kind": "exists" }
  },
  "escalation": ["notification", "vault-note"]
}
```

`129600` = 36h, correctly narrowed for a 20h-due nightly cadence (same value
`wiki-lint`'s own nightly routine uses).

## Finding 4 — SECURITY: an extension check is not a path allow-list (the most important finding)

The routine's manifest asserts the diff contains ONLY `.md` files before
pushing. But its OWN governing files — `skills/docs-sync/SKILL.md`,
`AGENTS.md`, `CLAUDE.md`, `docs/routines.md` — are ALL `.md` themselves, so
they would pass a bare extension check. Prohibitions in the SKILL.md prose
state the routine never edits them; nothing mechanical enforced that until
this PR added a deny-list. Without it, `/coderails:docs-sync`'s nightly
audit has zero self-referential exclusion and could flag-and-rewrite its
own safety contract through completely normal operation, each night
starting from whatever contract the previous night's run left behind.

Two further holes, both proven empirically (not merely asserted in the PR's
own prose — re-derived live in this ingest session, commands and output
below):

```
$ git init; echo "#!/bin/bash" > scripts/gate.sh; git add -A; git commit -m base
$ git checkout -b test-branch
$ git mv scripts/gate.sh evil.md
$ git diff --name-only HEAD
evil.md
$ git diff --name-status HEAD
R100    scripts/gate.sh    evil.md
```

`--name-only` prints the bare destination `evil.md` — a `.md` path on no
deny-list, passing an extension-only check while smuggling in a renamed
shell script. `--name-status` exposes the rename and its source.

```
$ git rm README.md; git commit -m "remove readme"
$ git diff --name-only HEAD~1
README.md
evil.md
$ git diff --name-status HEAD~1
D    README.md
R100    scripts/gate.sh    evil.md
```

Under `--name-only`, a deletion (`README.md`) is textually indistinguishable
from an ordinary edit to the same path — both just print the filename.
`--name-status` prints `D README.md`, the only way to tell "removed" from
"changed."

**Fix shipped:** `--name-status` (never `--name-only`) + an explicit
self-governance deny-list (`skills/**/SKILL.md` including its own file,
`AGENTS.md`, `CLAUDE.md`, `docs/routines.md`, anything under `.claude/`,
`examples/dashboard-config.json`) + reject any `R`/`C` line whose *source*
path wasn't already in-scope + reject any `D` line for an in-scope doc. Any
violation is an ABORT WITH CLEANUP (close PR if opened, delete branch
locally and remotely, log `abort=<reason>`) — never warn-and-continue.

## Finding 5 — tautological contract tests, and a proven regression

The shipped `hooks/scripts/tests/docs_sync_routine.test.sh` checks are bare
keyword greps on the SKILL.md prose. The test file itself documents why a
naive version of this is worthless (verified, read directly, lines
~181-185):

```
echo "no-drift handling was removed; the routine now always opens a PR" | grep -qi 'no-drift'
```

This line **passes** — it matches on prose meaning the exact opposite of
what the check is supposed to guarantee. This was not hypothetical: a
reviewer found an earlier pushed revision of the SKILL.md had **already**
silently dropped the entire failure-visibility section while all 14
original checks stayed green, because none of them anchored on the specific
sentence that section existed to make true.

**Fix:** each check anchors on the exact normative sentence (e.g. `grep -qi
'do \*\*not\*\* create a branch'`, not just `no-drift`), and the test file
adds `neg_check` negative controls — strip the normative sentence from a
scratch copy of the file and assert the check goes RED without it — proving
each check actually discriminates rather than just pattern-matching
adjacent vocabulary.

## Finding 6 — honest limits (stated plainly, not softened)

- Every manifest condition and the self-governance deny-list are
  **prompt-enforced, not hook-enforced.** `PreToolUse` hooks do not fire
  under `claude -p` `(verified in-repo — see [[routines]]'s security
  warning section, empirically confirmed 2026-07-07)`, and this repo
  deliberately carries no GitHub branch protection (owner decision,
  2026-07-15). The routine also generates its own review and eval
  artifacts and merges on them — the same headless run authors the docs
  AND attests they are good. Blast radius is capped at `.md` by the
  allow-list (the actual mitigation); the self-attestation is real and
  unresolved by this PR.
- **Scope narrowed vs. the literal ask.** The owner's original ask was "no
  human involved." The shipped routine instead *refuses* to auto-fix drift
  in its own governing docs and escalates those findings to a human
  instead — a deliberate, disclosed narrowing, not a silent shortfall.
- The precedent this pattern copies, `loop-retro-promotion-weekly`, has
  never fired its full self-merge chain end-to-end —
  `promotion-runs.log` holds exactly one line, `predicate=unmet`. So
  `sync-docs-nightly` is this repo's **first actual production exercise**
  of the full headless chain (task-evals → push → review-pr → post-review
  → post-evals → merge in one `claude -p` run), not a repeat of a
  battle-tested path.

## Finding 7 — proven live, 2026-07-17

The real seeder queued the routine (`Seed complete: 1 seeded, 4 not due`)
and the live watch-triggered sweeper claimed and ran it (`Sweep complete: 1
claimed, 1 succeeded, 0 failed`). Frozen loop evals E6/E7 passed against the
live config. **Not exercised:** the self-merge chain itself — no drift
existed on this run, so the no-drift short-circuit (step 2 of the SKILL.md)
was the correct path taken, and steps 3-9 (the actual edit → push → review
→ merge chain) remain unexercised in production.

## Wiki pages updated

- [[routines]] — primary aggregation target; swapped `sync-docs-weekly` for
  `sync-docs-nightly` in the shipped-routines list, folded in findings 2-4/6
- [[docs-sync]] — new skill page for `skills/docs-sync/SKILL.md`
- [[sync-docs]] — corrected a stale claim that it is "user-level only, not
  part of coderails"; it was imported in-repo 2026-07-10 (`f488ca5`),
  predating this cluster
- `index.md` — added `docs-sync` skill row, bumped `last_updated`

## Caveats / gotchas

- Do not confuse `sync-docs` (the older audit skill) with `docs-sync` (the
  new nightly pipeline that invokes it) or with `sync-docs-nightly` (the
  routine/config name) or the `sync-docs` button (renamed from `docs-sync`
  mid-PR, commit `f3ef802`).
- No wiki page exists yet for PR #199's sandbox workers; PR #207 documents
  them in README only. Out of scope for this ingest — flagged, not built.
- `dashboard-runs/sync-docs-weekly.md` is the OLD routine's run-note file;
  the new routine's note lives at `dashboard-runs/sync-docs.md` (button
  name), a separate file. Both are non-wiki operational output per
  `AGENTS.md`'s page-type table — never linked via `[[wiki-links]]`.
