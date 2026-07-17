---
title: "PR #228 + #229 + #230 — token-burn reduction for agentic loops, AGENTS.md split, honest headless-child cost accounting"
type: source
created: 2026-07-17
last_updated: 2026-07-17
sources:
  - skills/agentic-loop/SKILL.md
  - AGENTS.md
  - AGENTS-wiki-schema.md
  - hooks/scripts/lib/loop_cost.sh
  - hooks/scripts/tests/loop_cost.test.sh
  - skills/wiki-ingest/SKILL.md
  - skills/wiki-init/SKILL.md
  - skills/wiki-lint/SKILL.md
  - skills/wiki-query/SKILL.md
tags: [skill, agentic-loop, token-burn, compaction, model-pinning, orchestrator-discipline, tool-output-diet, agents-md, wiki-schema, loop-cost, headless-children, cost-tracking, fail-open, zsh-compat]
---

# PR #228 + #229 + #230 — token-burn reduction for agentic loops

Three PRs merged same day (2026-07-17) as one thematic cluster: cutting how
much context the `agentic-loop` orchestrator burns across a long loop. #228
and #229 are genuinely token-burn work; #230 is adjacent but distinct — it
makes the loop-cost **measurement** more honest (surfaces a previously-silent
undercount), it does not itself reduce burn. Clustered together because they
merged in the same session and #230's honesty fix is what makes any future
token-burn optimisation *measurable* — cost numbers that hide a category of
excluded spend would mask whether #228/#229's rules are actually working.

## PR #228 (`c29f147a84d1c0f23309558591159ba13836d69a`) — 4 token-burn rules in `skills/agentic-loop/SKILL.md`

Merged 2026-07-17T21:22:28Z (verified: `gh pr view 228 --json mergedAt,mergeCommit`
and `git diff c29f147..origin/main -- skills/agentic-loop/SKILL.md` returns
empty — the merge commit's content is byte-identical to what's on
`origin/main` today). 14 insertions, 1 net line removed, all inside
`skills/agentic-loop/SKILL.md`.

Four rules added, each explicitly numbered "row N of 4" in the skill text so
the four are traceable as one set even though they land in different phases:

| Row | Phase | Rule |
|---|---|---|
| 1 of 4 | **New Phase 4b sub-step** + `## Context-window persistence` | **Mandatory phase-boundary compaction.** Immediately after a work-unit's PR merges and its worktree is torn down: (a) write the phase summary into `progress.json`, (b) run `/compact`, (c) re-read `progress.json` + `plan.md` to re-orient. Called "the single largest saving" in the skill text — not optional, not skippable because the loop "still has headroom": orchestrator context grows linearly across work-units and is re-read in full on every turn at orchestrator rates, so the PR-merge boundary (the moment `progress.json` is already guaranteed current) is the cheapest place to cut it back. Explicitly distinguished from the pre-existing "don't stop early because context is filling" guidance in the same section — that rule is about not halting the *loop*; this rule is about not letting context grow *unbounded between merges* in the first place. |
| 2 of 4 | **New Phase 0.4** (between Phase 0 and Phase 0.5) | **Orchestrator model pinning at loop launch.** Pin the orchestrator's own model explicitly via `/model` (`opus` or `sonnet`) at loop launch, alongside Phase 0's envelope read — never leave it on an unpinned default, which "can silently resolve to a costlier frontier tier (e.g. 2x the cache-read rate of a pinned mid-tier model)" and compounds because the orchestrator re-reads its whole growing context every turn for the loop's life. Explicitly distinct from Phase 2.8's worker model routing — 2.8 assigns roles to *spawned workers*; 0.4 is the orchestrator's own pin, decided once at launch. |
| 3 of 4 | **Phase 4** (probe discipline) | **Batch the probe battery.** The Phase 4 idle-worker verification battery (git status, `gh pr view`, prod log, artifact check) — and any similar battery (Phase 12 artifact checks, gate-state reads, `gh pr view` sequences) — goes in ONE compound Bash call (`&&`/`;`-joined or piped), not one call per check. Rationale given: 4 probes as 4 separate turns costs "roughly 4x the cache-read volume" of the same 4 chained and read once, because each orchestrator turn re-reads the full accumulated context. Compound the *reads*, not the *decisions* — still reason once over the combined output. |
| 4 of 4 | **Phase 4** (tool-output diet, same paragraph as row 3) + **Phase 3** | **Cap probe output; orchestrator never authors deliverable files inline.** (a) Pipe probe output through `jq -c`, `head`, or an equivalent limiter before it enters context, so a large `git diff --stat` or `gh pr view` payload doesn't sit in the transcript re-inflating every later turn's re-read. (b) New Phase 3 rule: the orchestrator's own `Write`/`Edit` calls are for **loop-state only** (`progress.json`, `spec.md`, `plan.md`, `retro.json`). Every deliverable artifact (code, docs, config — anything that ends up in a PR) is authored by a spawned worker, never typed inline in main context. Workers report back structured, confidence-labelled verdicts — a short claim plus the verifying command — "not long narrative prose; a verbose report re-inflates the same context this rule is trying to keep small." |

**Also in the diff, not a token-burn rule:** the Phase -2 stub's example JSON
comment shows `"schema_version": 2` unchanged (the diff visible on the
feature branch tip momentarily read `1`, an artefact of the feature branch
being cut before PR #198's proof-gate schema bump landed on `main` — the
3-way merge at `c29f147` correctly reconciled this; **verified: the merged
content on `origin/main` matches pre-#228 `main` exactly on `schema_version`
and every `proof_disposition` reference — no regression shipped**). Not a
finding, recorded here only because a raw two-point branch diff looked like
one at first read.

Stage-map table also gained the new phase: `Setup` row now reads `-2, -1, 0,
0.4, 0.5` (was `-2, -1, 0, 0.5`).

## PR #229 (`47f28d616791dc03482f050bc678bef52270868f`) — split `AGENTS.md` into slim core + `AGENTS-wiki-schema.md`

Merged 2026-07-17T21:25:46Z. `AGENTS.md` went from 439 → 328 lines (net,
across the two files: +146/-130 across 6 files). New file
`AGENTS-wiki-schema.md` (127 lines) now holds everything wiki-specific: vault
location + `git`/`wiki` config block, vault directory structure, the "Three
layers" model (raw sources / the wiki / AGENTS.md), the page-type table, page
frontmatter format, the enforcement-model wiki-lens pointer, and the
ingest/query/lint workflow descriptions.

`AGENTS.md` itself keeps: what the plugin is, how the pieces wire together,
the two-enforcement-mechanisms distinction (hooks vs commands) and its
enforcement-ceiling caveats, the sandboxed-workers narrowing note, the
skills↔hooks seam convention, the full hook event map table, the workflow
command architecture (`/workflow` chain, config resolution), project-specific
assumptions, and "working in this repo" notes. At the bottom, a short pointer
section — "The coderails wiki schema — see AGENTS-wiki-schema.md" — replaces
the inline content, directing to the new file for anything wiki-specific.

**Four wiki skills' SKILL.md files were updated to point at the new file**
(one-line change each, `skills/wiki-ingest/SKILL.md`, `skills/wiki-init/SKILL.md`,
`skills/wiki-lint/SKILL.md`, `skills/wiki-query/SKILL.md`) — wherever they
referenced `AGENTS.md` as the wiki-schema source, they now reference
`AGENTS-wiki-schema.md` instead (or in addition, for `wiki-init` which
presumably still needs `AGENTS.md`'s pointer). This ingest agent did not
re-read each skill's exact wording; the fact of the redirect is verified from
the PR's file list and the receiving document's own text ("Split out of
`AGENTS.md` (2026-07-17)... `AGENTS.md` is still the entry point; it links
here" — verified: `AGENTS-wiki-schema.md` current content on `origin/main`).

**No wiki page previously existed for `AGENTS.md`'s own structure** — the
vault's `index.md` names `AGENTS.md` as the schema pointer inline ("Schema
and maintenance protocols are in `/Users/harrison/Github/coderails/AGENTS.md`")
but there is no dedicated `design/agents-md.md` or similar page. This source
page is the wiki's record of the split; no new concept page was manufactured
for it, per the existing convention of not creating pages ahead of an actual
information need.

## PR #230 (`ebd21d22d8f0c1b2fc3e4c5875ad68f492f5b499`) — `loop_cost.sh` surfaces headless `claude -p` children instead of silently excluding them

Merged 2026-07-17T21:27:27Z. 3 commits on the branch, all in
`hooks/scripts/lib/loop_cost.sh` (+ its test file):

1. **`c70c2387` — feature.** Headless `claude -p` children (e.g.
   `skills/dashboard/scripts/run-builder.sh`'s bypass spawn) land as their
   OWN top-level `<proj>/<other-session>.jsonl` transcript with no
   `subagents/` linkage back to the orchestrator session — `dc_mine_token_usage`
   has no sound way to attribute their tokens, so it has always excluded them
   from `per_model`/`total_tokens` (documented in the pre-existing `notes`
   field: "headless claude -p child sessions excluded"). What was missing
   before this PR: **any signal that exclusion had happened at all** — a
   reader saw a clean total with no indication it might be an undercount.
   The fix adds `headless_children_excluded_count` to the miner's output: a
   candidate count of sibling top-level `.jsonl` files in the same `<proj>`
   dir (excluding the orchestrator's own transcript) whose mtime falls within
   a symmetric window of the orchestrator transcript's mtime — default
   `3600`s, overridable via `CLAUDE_HEADLESS_WINDOW_SECS`, chosen to cover a
   single build's wall-clock budget (`BUILDER_WALL_CLOCK_SECS` default 2700s
   in `run-builder.sh`) with margin. The `notes` field text was extended to
   point a reader at the new count. This is honest-disclosure, not
   attribution — the excluded tokens are still not folded into the total;
   the count just tells you an undercount category exists and roughly how
   large the candidate set is.
2. **`931095cc` — review-driven guard.** `CLAUDE_HEADLESS_WINDOW_SECS` was
   read directly into a `-le` numeric comparison with no validation — a
   non-numeric override would leak a raw `"[: integer expression expected"`
   to stderr. Guarded with `case "$headless_window" in ''|*[!0-9]*)
   headless_window=3600 ;; esac` — falls back to the default rather than
   mining a garbage window; still fail-open either way, consistent with the
   rest of this function's contract (see [[loop_cost]]).
3. **`ebd21d22` (the branch's actual HEAD, security-review-driven) —
   zsh stdout-corruption fix.** The new headless-counting loop originally
   declared `local f_mtime diff` *inside* the `while read` loop body. Under
   zsh 5.9 specifically, a `local` redeclared on each loop iteration echoes
   `"name=value"` to stdout on the 2nd+ pass — this corrupts the function's
   single-JSON-object-on-stdout output contract (the file has explicit
   zsh-compat support, so this is a real, not theoretical, break). Fixed by
   hoisting `f_mtime diff` out to the same `local` declaration as `orch_mtime`,
   before the loop starts.

**Net effect:** `dc_mine_token_usage`'s returned object gains one new integer
field, `headless_children_excluded_count`; the pre-existing `{}` fail-open
contract, the 7 distinct-stderr-message bail sites from PR #217, and the
dedupe-by-`message.id` mining logic are all unchanged. See [[loop_cost]] for
the full field contract and fail-open posture this PR extends rather than
replaces.

## Relationship between the three PRs

Not a dependency chain — three independent branches (`feat/tokenA-skill`,
`feat/tokenB-agents`, `feat/tokenC-loopcost`) merged sequentially into `main`
the same session, each touching disjoint files (`SKILL.md` only; `AGENTS.md`
+ `AGENTS-wiki-schema.md` + 4 skill files; `loop_cost.sh` + its test file).
Clustered here as one wiki source page per this loop's Phase 9 instruction
("cluster wiki ingest, don't fragment... one source page covers the cluster")
— not because the three PRs share code, but because they share a session and
an adjacent theme (cost/context discipline for the orchestrator).

## Wiki pages updated

- [[agentic-loop]] — the 4 token-burn rules (PR #228) added under their
  respective phases; the `## Context-window persistence` no-touch-region
  claim reconciled against the new "Compaction cadence" content PR #228 added
  inside that exact section (see the reconciliation note added to that page).
- [[loop_cost]] — `headless_children_excluded_count`, the numeric guard, and
  the zsh stdout-corruption fix (PR #230) added to its existing sections.
- No dedicated `AGENTS.md` structure page existed before this ingest and none
  was created — this source page is the wiki's only record of the PR #229
  split; `index.md`'s existing pointer to `AGENTS.md` needs no change since
  it already names the file generically as "Schema and maintenance
  protocols," which remains true (just now split across two files).

## Caveats / gotchas

- **PR #228's Phase 4b compaction rule and the `## Context-window
  persistence` section's new "Compaction cadence" paragraph are two mentions
  of the same rule, not two different rules** — Phase 4b states the
  mechanical trigger (right after per-unit worktree teardown), and
  `## Context-window persistence` restates the same cadence for readers who
  land on that section first. Do not wiki-document them as if they were
  independent decisions.
- **The `## Context-window persistence` section is one of the skill's six
  documented "no-touch regions"** (byte-stable, per the Slimming-era
  convention documented on [[agentic-loop]]) — PR #228 edited it anyway. This
  is the same kind of intentional supersession PR #86 made to the Phase 13
  KPI no-touch region: the no-touch convention protects content from
  *accidental* drift, not from a deliberate, reviewed rule addition. Treat
  this as documented precedent, not a violation.
- PR #230's `headless_children_excluded_count` is a **candidate count**, not
  a resolved attribution — it can overcount (any unrelated session active in
  the same project dir within the time window) or undercount (a headless
  child outside the window, or in a different project dir). Do not treat it
  as a precise number of excluded tokens' worth of sessions.
