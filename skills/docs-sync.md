---
title: "Skill: docs-sync"
type: skill
created: 2026-07-17
last_updated: 2026-07-22
sources:
  - sources/pr_207_209_docs-sync-nightly-and-drift-fix.md
  - sources/pr_262_runner-stdin-stall.md
tags: [skill, docs-sync, sync-docs, routines, machine-run, self-merge, security, agentic-os]
---

# Skill: docs-sync

A **machine-run** pipeline that audits this repo's own git-tracked
documentation for drift against the actual codebase every night and, only
if drift is found, edits, pushes, reviews, and self-merges a fix with **no
human in the loop**. Ships in-repo at `skills/docs-sync/SKILL.md`. Runs as
the `sync-docs-nightly` routine (button `sync-docs`) — see [[routines]].

**Not the same skill as [[sync-docs]].** `docs-sync` *invokes*
`sync-docs`'s audit (its Phases 1-3: discover project structure, traditional
audit, generate drift report) as step 1 of a larger pipeline; `sync-docs`
itself has no self-merge behaviour and is invoked interactively or by
[[agentic-loop]] Phase 9. This page and [[sync-docs]] document two distinct
skills that happen to sound similar and depend on each other.

## Why it exists — replacing a routine that was broken for 9 days

`docs-sync` replaces the former `sync-docs-weekly` routine, which was
**read-only** (it only ever produced a drift report, never fixed anything)
and had a `foreignSkillPath` pointing at
`/Users/harrison/.claude/skills/sync-docs/SKILL.md` — a path that never
existed; the real skill had already moved in-repo. That routine ran green
2026-07-08, escalated a correctly-detected `skill-missing` red run on
2026-07-15, and then never ran again — everything merged after 2026-07-15
drifted unaudited. `docs-sync` needs no `foreignSkillPath` at all: its
skill ships in-repo, the same pattern [[loop-retro-promotion]] uses. Full
root-cause account, including a retracted-and-corrected claim about how the
failure was found, is in
[[pr_207_209_docs-sync-nightly-and-drift-fix]].

## Pipeline

1. **Audit** — invoke `/coderails:sync-docs`'s traditional audit (Phases
   1-3) to detect drift. Scope: git-tracked `.md` files only — `README.md`,
   `AGENTS.md`, `CLAUDE.md`, tracked `docs/*.md` — confirmed via `git
   ls-files --error-unmatch` before treating anything as in-scope. Never
   "fixes" a doc to match code it has not read; reports uncertainty rather
   than guessing.
2. **No-drift short-circuit** — runs BEFORE any git state is touched. If
   the audit finds nothing, append a `no-drift` line to the run log (this
   IS the routine's `expectedArtifact`) and exit 0. No branch, no PR. This
   specifically prevents nightly PR spam on the (expected-common) nights
   nothing needs fixing.
3. **Delivery**, only if drift was found — the full gate chain, in order:
   fetch + branch off fresh `origin/main` → freeze `/coderails:task-evals`
   (pr scope) BEFORE editing → make the doc edits → **manifest assertion**
   (see Security below) → `/coderails:push` → `/pr-review-toolkit:review-pr`
   → `/coderails:post-review` → `/coderails:post-evals` → `/coderails:merge`.
   Any step can REFUSE; a refusal is treated exactly like a manifest abort
   (close PR if opened, delete branch locally+remotely, log
   `refused=<gate>`) — never retried in the same run, never relaxed.
4. **Run log** — one append-only log at `expectedArtifact.artifactPath`
   (`run-{date}.log`), one line per stage, mirroring
   [[loop-retro-promotion]]'s `promotion-runs.log` convention. Both the
   no-drift short-circuit and every delivery stage write here.

## Security: the manifest is a self-governance deny-list, not just an extension check

The core safety property is "this routine only ever touches `.md` files,"
but a bare extension check is insufficient — the routine's own governing
files (`skills/docs-sync/SKILL.md` itself, `AGENTS.md`, `CLAUDE.md`,
`docs/routines.md`) are ALL `.md`, so they'd pass a naive check. The
manifest step asserts `git diff origin/main...HEAD --name-status`
(**three-dot**, never two-dot — a sibling PR merging mid-run must not move
the comparison base and indict a clean branch; **`--name-status`, never
`--name-only`** — `--name-only` prints a `git mv`'d file as its bare
destination alone, letting a renamed shell script slip through as `.md`,
and prints a deletion identically to an edit) against four conditions:
every path is a tracked `.md` file; no path is on the self-governance
deny-list even though it's `.md`; no `R`/`C` line whose source wasn't
already in-scope; no `D` line for an in-scope doc. Any violation is an
ABORT WITH CLEANUP, never warn-and-continue. See
[[pr_207_209_docs-sync-nightly-and-drift-fix]] for the two exploits proven
empirically (`git mv` smuggling a script in as `.md`, `git rm` masquerading
as an edit) and why `--name-status` is what makes both detectable.

**Honest limit:** this whole manifest+deny-list is enforced in the skill's
own prompt inside `claude -p`, where `PreToolUse` hooks do not fire (see
[[routines]]'s security warning). It reduces the risk of self-governance
drift; it does not eliminate it the way a hook-level or server-side check
would. This repo carries no branch protection by design, so the manifest
assertion plus `scripts/merge.sh`'s own script-internal gates plus
`/pr-review-toolkit:review-pr` are the entire merge rail — same honest
ceiling as [[loop-retro-promotion]].

## Contract tests are regression-locked, not just present

`hooks/scripts/tests/docs_sync_routine.test.sh` doesn't just grep for
keywords — a naive keyword grep (`grep -qi 'no-drift'`) passes even on
prose that states the *opposite* of the intended behaviour, and an earlier
pushed revision of the SKILL.md had already silently dropped its entire
failure-visibility section while all 14 original checks stayed green. The
shipped fix anchors each check on the exact normative sentence and adds
`neg_check` negative controls that strip that sentence from a scratch copy
and assert the check goes RED without it.

## Not yet exercised end-to-end

Live-fired 2026-07-17: seeder queued it, sweeper claimed and ran it
successfully, frozen loop evals E6/E7 passed against the live config. But
no drift existed that run, so only the no-drift short-circuit path was
exercised — steps 3-9 of Delivery (the actual self-merge chain) remain
unexercised in production. The pattern it copies,
[[loop-retro-promotion]]'s routine, has *also* never fired its full
self-merge chain (`promotion-runs.log` shows one `predicate=unmet` line) —
so `docs-sync` is this repo's first real production test of the complete
headless task-evals→push→review→merge chain.

**Update (2026-07-22): the routine HAS since fired with a real failure mode.** An
investigation behind [[pr_262_runner-stdin-stall|PR #262]] observed `docs-sync` running
**6 red / 3 green**, the dominant cause being routines exiting 0 without writing their
run-log artifact `(inferred — the run ledger was not read for this ingest)`. So the "not
exercised" framing above is stale for the *run-scheduling* path: the routine fires and fails;
what remains unexercised is the successful full self-merge chain. The exit-0-without-artifact
failure is one of five [[dashboard-runner#Known open runner defects|open runner defects]]
surfaced by that same investigation — its root cause was out of scope for PR #262 and is
unfixed.

## Scope narrower than the literal ask

The original ask was "no human involved." The shipped routine instead
refuses to auto-fix drift in its own governing docs (the deny-list above)
and escalates those findings to a human instead — a deliberate, disclosed
narrowing.

## Source

`coderails/skills/docs-sync/SKILL.md`

## See also

- [[sync-docs]] — the separate, older skill this one invokes for its audit step
- [[routines]] — `sync-docs-nightly`'s full routine-config treatment, the `maxAgeSeconds`/`foreignSkillPath` fixes, and the security warning this page cites
- [[loop-retro-promotion]] — the precedent pattern this skill's self-merge chain and manifest-lock idea both copy
- [[pr_207_209_docs-sync-nightly-and-drift-fix]] — source record: root cause, all findings, live-fire verification
- [[pr_262_runner-stdin-stall]] — the investigation that observed this routine's real 6-red/3-green failure mode and catalogued five open runner defects
