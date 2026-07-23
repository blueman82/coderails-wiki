---
title: "enforce_pr_workflow.sh"
type: hook
created: 2026-06-25
last_updated: 2026-07-23
sources: [sources/pr_19-30_self-containment-and-hardening.md, sources/pr_40_hook-hardening.md, sources/pr_42_skills-hooks-seam.md, sources/pr_46_gate-git-push-on-main.md, sources/pr_49_gate-function-rename.md, sources/pr_57-62_subagent-enforcement-gate-hardening.md, sources/pr_64_loop-review-via-skill.md, sources/pr_76_harden-hook-stdin-read.md, sources/pr_96-98_evals-gate-uniform-enforcement_2026-07-08.md, sources/pr_144-149_agentic-loop-hardening-from-loop-engineering.md, sources/pr_232_tier-review-gate.md, sources/pr_279_merge_time_smoke_reexecution.md]
tags: [hook, PreToolUse, enforcement, pr-workflow, workflow-chain, merge-sh, tier-review, root-daemon, smoke-verify, worktree, re-execution]
---

# enforce_pr_workflow.sh

PreToolUse(Bash) hook that mechanically guards the PR workflow chain: blocks `gh pr create` unless `/coderails:push` ran this session; blocks `gh pr merge` unless `/pr-review-toolkit:review-pr` ran this session referencing the same PR number (PR #58); (since PR #40) blocks `git merge` on `main`/`master` unless `/pr-review-toolkit:review-pr` ran since the last `git merge` (consume-on-use, PR #58); and (since PR #46) blocks `git push` that lands on `main`/`master` unless `/pr-review-toolkit:review-pr` ran this session, including bare positional `git push origin main` from any branch (PR #58). (since PR #97, 2026-07-08) also blocks `gh pr merge <N>` — after the review-pr check above already passes — unless a SHA-bound `GO` coderails eval artifact exists for the PR's current head, mirroring `scripts/merge.sh`'s own eval gate. (since PR #146, 2026-07-12) `scripts/merge.sh <N>` invocations are now recognised as the same gated `merge` subcommand as raw `gh pr merge <N>` — previously a hand-rolled `scripts/merge.sh <N>` bypassed this entire hook. (since [[pr_279_merge_time_smoke_reexecution|PR #279]], 2026-07-23) also blocks, right after the eval-artifact gate and before the tier-review gate below, unless `post_evals::smoke_verify` re-executes every scripted eval's `cmd`/`negative_control` successfully against the trusted head SHA in a detached worktree — closing the gap where the eval-artifact gate only ever parsed the marker's `result=GO` text, never re-validating the artifact's content. (since [[pr_232_tier-review-gate|PR #232]], 2026-07-17) also blocks, for a tier-0 eval artifact only and only when `tier_review.machine_user` is configured, unless a matching `tier-review` commit status of `state: success` from that exact machine user exists for the head SHA (Gate 9).

## Event and mode

| Field | Value |
|---|---|
| Event | `PreToolUse (Bash)` |
| Mode | **block** (permissionDecision: deny) |
| Timeout | (default) |

## Logic summary

Skip gates (cheap first):

1. Empty command string — pass.
2. `--help` / `--dry-run` flags — pass (word-boundary anchored since PR #58 so `--dry-run-data` doesn't accidentally bypass). `git merge --abort/--continue/--quit/--skip` (conflict-resolution ops) — pass.
3. Command does not match `gh pr create`, `gh pr merge`, `git merge`, `git push`, or `merge.sh` — pass. Gate 3 splits on shell separators (`;|& && ||`) and tests whether any segment **begins with** a gated command, avoiding false-positives on substring mentions (e.g. `printf 'gh pr create' > file` was previously blocked). (verified — hook source) The matched segment is captured (`matched_seg`, PR #146) for downstream PR-number extraction — see "The `merge.sh` matcher" below.
4. `workflow.config.yaml` absent (NO_CONFIG sentinel) — pass. The hook is opt-in via the full workflow stack. (verified — PR #30)
4b. For `git merge` and `git push` only: pass unless the operation touches `main`/`master`.
   - `git merge` integrates into the **checked-out** branch → the current branch decides. If not on `main`/`master` — pass.
   - `git push` is decided by its **destination** → gate when on `main`/`master`, OR when the command names an explicit main/master destination refspec (`HEAD:main`, `feature:master`, `:refs/heads/main`) OR a bare positional main/master target (`git push origin main`) from any branch. (added positional target PR #58; destination-refspec model PR #46)
   Feature branches are unconditionally allowed. Detached HEAD / empty branch name falls through to allow (same safe-fail default as [[no_edit_on_main]]).
5. No transcript path in hook payload — pass (can't enforce). **Subagent support (PR #58):** when `.agent_transcript_path` is present and readable, it is scanned **in addition to** `.transcript_path`. Closes the gap where a subagent ran `/push` or `/review-pr` without the parent session having a record.
6. Transcript scan for required preceding step. If evidence found — pass. If not found — deny.

Subcommand routing after Gate 3: `create` → requires `/coderails:push` evidence; `merge` (gh) → requires `/pr-review-toolkit:review-pr` referencing the same PR number (per-PR check, PR #58); `git_merge` → requires `/pr-review-toolkit:review-pr` since the last `git merge` (consume-on-use, PR #58); `git_push` → requires any `/pr-review-toolkit:review-pr` this session.

## Block condition

- `gh pr create` called without a prior `/coderails:push` in the session transcript.
- `gh pr merge <N>` called without a prior `/pr-review-toolkit:review-pr <N>` (leading-token PR number match) in the session transcript. Bare `gh pr merge` (no number) accepts any review-pr. (per-PR check added PR #58)
- `gh pr merge <N>` called (past the review-pr check above) without a SHA-bound `GO` coderails eval artifact for the PR's current head — fail-closed, including on a `gh` fetch failure. Does not apply to `git_merge`/`git_push` (no PR number to resolve a SHA-bound artifact against — documented residual). (added PR #97, 2026-07-08)
- `scripts/merge.sh <N>` (or `./scripts/merge.sh <N>`, or `bash <path>/merge.sh <N>`) called — gated identically to `gh pr merge <N>` above (same review-pr + eval-artifact checks). (added PR #146, 2026-07-12)
- `git merge` on `main`/`master` called without a prior `/pr-review-toolkit:review-pr` SINCE the last `git merge` (consume-on-use, added PR #58; original session-scope check PR #40).
- `git push` landing on `main`/`master` (current branch, explicit destination refspec, or bare positional target) called without a prior `/pr-review-toolkit:review-pr` in the session transcript. Bare positional `git push origin main` gated as of PR #58; destination-refspec model added PR #46.

All checks are NO_CONFIG-gated (Gate 4).

## Why the `git merge` gate was added (PR #40)

The `finishing-a-development-branch` skill includes a "merge locally" option that bypasses the PR path entirely — no `gh pr merge` ever runs, so the pre-existing gate never fires. This left a bypass route. The `git merge` gate closes it: even a local fast-forward merge on main now requires review evidence. (verified — PR #40)

## merge-base exclusion: the word-boundary footgun (PR #42)

PR #40's original gate regex was `\bgit +merge\b`. This also matched `git merge-base` because in POSIX ERE (used by `grep -E`), `-` is a word boundary — so the boundary fires between `merge` and `-base`, and `\bmerge\b` matches. `git merge-base` is a read-only ancestor-lookup plumbing command; blocking it was wrong.

**Before (PR #40):** `\bgit +merge\b`  
**After (PR #42):** `\bgit +merge([[:space:]]|$)`

The fix requires the token after "merge" to be whitespace or end-of-line, excluding `merge-base`, `merge-file`, and `merge-tree`. Applied at both Gate 3 (command classification) and the subcommand-detection block. New test Case 14 asserts `git merge-base HEAD main` on main → allow. (verified — hook source lines 28 and 37)

See [[skills-hooks-seam]] for the general pattern and a note on hyphenated-command regex design.

## The `git push` gate: destination-refspec model (PR #46)

PR #46 added `git_push` as a fourth gated subcommand, mirroring the `git merge` gate's review-evidence requirement. The key difference is **what decides whether to gate**:

- `git merge` integrates into the checked-out branch, so the **current branch** is sufficient.
- `git push`'s effect is decided by its **destination**, so the destination must be parsed. Gate 4b gates when the current branch is `main`/`master` **OR** the command names an explicit main/master destination refspec (`HEAD:main`, `feature:master`, `:refs/heads/main`) from any branch.

The destination anchor is `:(refs/heads/)?(main|master)([[:space:];&|)]|$)`. The trailing class accepts whitespace, EOL, **or** a shell separator (`;& |)`) so that `git push origin HEAD:main;echo` can't abut a metachar onto the ref to evade the gate. (verified — hook source lines 81–84)

**Documented limitation:** bare positional `git push origin main` from an off-main branch is **not** parsed (low risk — it pushes local `main`, which tracks `origin/main`). The colon-refspec form is the realistic direct-to-main bypass, and that is closed. Feature-branch pushes are never gated — the PR flow requires them. (verified — hook source lines 9–10, 70–73)

**Review-caught Critical:** the first implementation checked the current branch only, which let a `HEAD:main` refspec push from a feature branch through — a silent false-allow flagged Critical by the `pr-review-toolkit` multi-agent review and fixed by destination-refspec gating + the metachar anchor, each TDD'd. Enforce test suite went 14→27 cases. This does **not** reverse [[pr_44_no-edit-plugin-source]]'s "don't gate git push for [[no_edit_on_main]]" decision — that was edit-seam protection; this is review-evidence enforcement in a different hook. See [[pr_46_gate-git-push-on-main]].

## Reordered git-merge block-message hint (PR #42)

The block message for `git merge` on main now leads with the actual resolution ("Run /pr-review-toolkit:review-pr first") before listing `/coderails:merge` and the settings.json bypass. Matches the adjacent `gh pr merge` hint order.

## Named gate functions (PR #49)

PR #49 replaced positional `# Gate N` comments with named bash functions, making each gate's purpose self-documenting and greppable. Originally seven functions; now nine, in evaluation order (Gate 8 added by [[pr_279_merge_time_smoke_reexecution|PR #279]], 2026-07-23; Gate 9 by [[pr_232_tier-review-gate|PR #232]], 2026-07-17 — see the sections below the table for both):

| Function | Gate | Purpose |
|---|---|---|
| `gate_has_command` | 1 | Pass if command string is empty |
| `gate_safe_passthrough` | 2 | Pass for `--help`, `--dry-run`, conflict-resolution ops |
| `gate_in_scope` | 3 | Pass if command is not a gated subcommand; sets `$subcommand` (and `$matched_seg`, PR #146) |
| `gate_config_present` | 4 | Pass if `workflow.config.yaml` absent (NO_CONFIG opt-in) |
| `gate_targets_main` | 4b | Pass if `git merge`/`git push` does not target main/master |
| `gate_have_transcript` | 5 | Pass if no transcript path in payload |
| `enforce_required_step` | 6 | Scan transcript; pass if evidence found, deny if not |
| `gate_eval_artifact_for_merge` | 7 | For `merge` subcommand only, after Gate 6 passes: pass if a SHA-bound `GO` coderails eval artifact exists for the PR's current head, deny if not (added PR #97) |
| `gate_smoke_verify` | 8 | For `merge` subcommand only, after Gate 7 passes: pass if `post_evals::smoke_verify` re-executes every scripted eval successfully against the trusted head SHA, deny if not (added PR #279) |
| `gate_tier_review_status` | 9 | For `merge` subcommand only, after Gate 8 passes, tier-0 artifacts only: pass if a matching `tier-review` commit status exists, deny if not; inactive no-op unless `tier_review.machine_user` is configured (added PR #232) |

`gate_targets_main` is the headline rename: the former label "Gate 4b" conveyed only position; the name now states the decision the gate makes. Mirrors the `require::` / `pr::` naming idiom in `scripts/lib/git-common.sh`. (verified — PR #49)

## Evidence model and known limitation (PR #49 documented, deferred)

The transcript scan in `enforce_required_step` looks for *invocation evidence* — a Skill tool call with the required name appearing anywhere in the session transcript. It does **not** verify completion. Two weaknesses:

1. **Hollow invocation**: a Skill call that errors immediately still satisfies the gate.
2. **Substring false-positive**: assistant prose that *mentions* `gh pr create` (not a tool call) can trigger a false block.

This repo deliberately does not enable GitHub branch protection — a standing decision (2026-07-15), not a gap awaiting a deferred fix. There is no server-side "no unreviewed merge to main" guarantee here: the redirect + audit layer above is the whole enforcement story for this concern, by design. (verified — AGENTS.md / docs/routines.md, PR #182)

## Log output

Appends to `$CLAUDE_DISCIPLINE_LOG` on block. Format matches `key=value` convention.

## PR #58 hardening summary

Four changes in a single PR, each closing a specific bypass:

**Change A — subagent transcript support:** `gate_have_transcript` and `enforce_required_step` now also scan `.agent_transcript_path` when present, so subagent-run `/push` or `/review-pr` invocations are recognised as valid evidence by the parent session.

**Change B — per-PR / consume-on-use review evidence:** `gh pr merge <N>` now requires a review-pr whose `args` starts with the same PR number (leading-token match, not substring). `git merge` consumes evidence: review-pr must have run SINCE the last `git merge`, preventing a single review from satisfying multiple merges.

**Change C — positional git push origin main gated:** `gate_targets_main` detects `git push origin main` form from any branch as a destination target. Previously only colon-refspec forms were caught.

**Change D — flag-boundary tightening:** `gate_safe_passthrough` uses word-boundary anchors on `--dry-run` and `--help` so `--dry-run-data` or `--helpfulness` don't accidentally bypass the gate.

**Review-caught Critical (fixed before merge):** the initial Change-C implementation missed bare positional targets; flagged Critical by the multi-agent review and fixed before merge.

## Accepted PR-review evidence: the review-pr Skill with PR number (PR #64)

The `enforce_required_step` function (Gate 6) scans the transcript for `/pr-review-toolkit:review-pr` Skill invocations as the accepted form of review evidence. The accepted form has two requirements:

1. **Tool type is Skill** — a Skill tool call with name `pr-review-toolkit:review-pr`. A manually-spawned set of reviewer agents (via `Agent` or `Task` tool calls) does NOT satisfy this gate.
2. **PR number in args** — for `gh pr merge <N>`, the review-pr Skill invocation must carry the same PR number as a leading token in its args (per-PR evidence model, PR #58).

**Why Skill-only, not agent fanout:** The transcript scanner looks for specific Skill invocation records. An orchestrator that hand-rolls the six reviewer agents as parallel `Agent` blocks produces no `/pr-review-toolkit:review-pr` Skill record in the transcript — so the gate sees no evidence and blocks merge. This is the root cause documented in PR #64: the agentic-loop's Phase 4b was hand-rolling agents, which cleared the review conceptually but not mechanically.

**Resolution path:** Invoke `/pr-review-toolkit:review-pr <PR#>` as a Skill, passing the PR number. The Skill itself orchestrates the six reviewers and security pass internally. See [[agentic-loop]] Phase 4b section for the orchestrator-side obligation. See [[pr_64_loop-review-via-skill]] for the source record.

## Gate 7 — eval-artifact gate on gh pr merge (PR #97, 2026-07-08)

`scripts/merge.sh` already required a SHA-bound `GO` coderails eval-artifact PR comment (`pr::has_coderails_eval_for_head`) before running `gh pr merge` — but this hook only ever checked for review-pr evidence on that same command, so a raw `gh pr merge <N>` run outside `/coderails:merge` skipped the eval check entirely. This is the exact bypass [[pr_95_slash-command-loop-detection|PR #95]] shipped through: it satisfied the review-pr gate (a `code-reviewer`-clean `/pr-review-toolkit:review-pr` ran and is recorded) but produced zero eval artifact of any kind, and nothing forced one.

**New function `gate_eval_artifact_for_merge`**, appended after `enforce_required_step` in the gate chain and run once more before `exit 0`:

- Scoped to `subcommand = merge` only — `git_merge`/`git_push` have no PR number to resolve a SHA-bound artifact against and are explicitly left uncovered (documented residual, not a bug).
- Skips immediately if the review-pr gate (Gate 6) already denied this invocation (`[ "$step_found" -eq 0 ] 2>/dev/null && return 0`), so only one deny message is ever emitted per call, and the cheap transcript-only check always gets first refusal before this network-dependent one runs.
- `cd`'s into the payload's `cwd` in-process (not a subshell — a subshell would lose the `PR_EVAL_TIER`/`PR_TRUST_FETCH_FAIL_REASON` globals the deny-message branches below need) before calling `repo()`/`pr::*` helpers, which are CWD-dependent.
- Resolves the PR number (`$pr_num`, falling back to `pr::num "$(branch)"`) and head SHA (`pr::head_sha`), then calls `pr::has_coderails_eval_for_head`.
- **rc handling mirrors `scripts/merge.sh`'s own eval gate:** `rc=2` (gh fetch failed) → deny fail-closed, reason keyed off `PR_TRUST_FETCH_FAIL_REASON` (`identity`/`permission`/`tempfile`/default); `rc≠0` otherwise → deny, tier-aware NO-GO message if `PR_EVAL_TIER` is set, else "no coderails eval artifact for current head"; `rc=0` → allow (any tier's `GO`, including tier 0).
- The hook now sources `scripts/lib/git-common.sh` (pulling in `eval-artifact.sh` + `review-artifact.sh`) at the top of the file; its colour vars are `_GIT_COMMON_COLORS_LOADED`-guarded, so double-sourcing alongside `merge.sh`'s own in-process sourcing elsewhere is safe.
- NO_CONFIG posture unaffected: `gate_config_present` (Gate 4) still stands aside before any of Gate 7's code runs.

**Tests:** the whole suite (`hooks/scripts/tests/enforce_pr_workflow.test.sh`) gained a global `gh` mock (a fake executable, `MOCKGH_DIR`, placed first on `PATH`) so every pre-existing `gh pr merge` ALLOW case — none written with this new gate in mind — is transparently satisfied by a default `GO tier=1` marker. A dedicated new test section exercises the gate's own branches directly: no marker → deny, `GO` tier 1 → allow, `GO` tier 0 → allow, `NO-GO` tier 2 → deny naming the tier, comments-fetch failure → rc=2 fail-closed with a retry hint, `NO_CONFIG` → allow, and gate ordering (review-pr denial fires first). Suite went from 86 to 93 passing cases.

See [[task-evals-gate]] "Merge gate placement" for how this sits alongside `scripts/merge.sh`'s pre-existing pr-scope gate, and [[evals-gate-enforcement-gap_2026-07-08]] for the investigation that surfaced the gap this closes.

## Gate 8 — smoke-verify gate on gh pr merge ([[pr_279_merge_time_smoke_reexecution|PR #279]], 2026-07-23)

Gate 7 above only ever parses the marker comment's `result=GO` text — it never re-validates the eval artifact's own content. Checks 1-10 in `post_evals::validate_structure` (including check 10's re-execution) only ever ran once, in the posting agent's own `/coderails:post-evals` session — never again against a raw `gh pr merge`. This gate closes that: a hand-written `smoke` object for a `cmd` naming a script that never existed previously passed Gate 7 at rc=0.

**New function `gate_smoke_verify`**, called inline from inside `gate_eval_artifact_for_merge`'s own `else` branch — i.e. only once Gate 7's eval-artifact check has *already passed* (`eval_rc == 0`, any tier including 0) — as `gate_smoke_verify "$num" "$sha" && gate_tier_review_status "$num" "$sha"` (verified: `hooks/scripts/enforce_pr_workflow.sh`, the line directly after Gate 7's rc-handling `if/elif/else`). This is a change to how Gate 8 (below) is invoked too: previously called directly from the `elif`; now chained after Gate 8's own predecessor via `&&`, so a smoke-verify failure short-circuits before the tier-review check ever runs.

Extracts the fenced JSON embed from the same trusted comment Gate 7 already matched (`pr::coderails_eval_embed_for_head`), writes it to a tempfile, and calls `post_evals::smoke_verify` — checking out the trusted head SHA into a detached `git worktree` and re-executing every scripted eval's `cmd`/`negative_control` there. Denies (one JSON deny per invocation, same contract as every other gate in this hook) on: a `gh` fetch failure resolving the embed (fail-closed, reason keyed off `PR_TRUST_FETCH_FAIL_REASON`); no embed found despite Gate 7 having passed (an internal-consistency error, names it as such); a local `mktemp` failure; or `post_evals::smoke_verify` itself failing (environmental cmd/negative_control, or a negative_control observed passing — see [[task-evals-gate]] and [[pr_279_merge_time_smoke_reexecution]] for the full re-execution mechanism and the four bypasses closed in review). No config opt-out — same fail-closed posture as Gate 7.

## Gate 9 — tier-review status gate on gh pr merge ([[pr_232_tier-review-gate|PR #232]], 2026-07-17)

Mirrors `scripts/merge.sh`'s own step-4b tier-review check (see [[merge]]) into this hook's `gh pr merge`/`scripts/merge.sh` path, so a raw `gh pr merge <N>` gets the same tier-review requirement `scripts/merge.sh` already enforces — closing the same class of "wrapper enforces it, raw command bypasses it" gap Gate 7 above closed for the eval-artifact check.

**New function `gate_tier_review_status`**, called after Gate 8 (`gate_smoke_verify`) succeeds — originally (before [[pr_279_merge_time_smoke_reexecution|PR #279]]) called directly from `gate_eval_artifact_for_merge`'s `elif` branch where `PR_EVAL_TIER == "0"`; now reached only once Gate 8 has also passed. Still only fires for a tier-0 result, never for tier 1/2. Config-keyed and inactive by default: stands aside entirely (`return 0`, no deny) unless `tier_review.machine_user` is set in `workflow.config.yaml`, via a local `coderails::_tier_review_machine_user` extractor duplicated from `merge.sh`'s own (no shared lib file was in scope for this change, so both hooks carry their own copy rather than a new shared dependency).

When active, fetches the newest `tier-review` commit status for the SHA (`gh api .../statuses --jq 'select(.context == "tier-review")'`) and denies (one JSON deny per invocation, same contract as every other gate in this hook) on any of: a `gh` fetch failure (fail-closed, retry-hint message); no status found yet (daemon hasn't judged this SHA); `state != success`; or `creator.login != machine_user` (misconfiguration-or-forgery signal, never bypassed). Unlike `merge.sh`'s own check, this gate's deny does **not** re-check the status description for `verdict=legitimate` — the state+creator check alone is what's mirrored here (verified against the actual `enforce_pr_workflow.sh` diff: no `tr_desc`/`verdict` string check appears in `gate_tier_review_status`, only in `merge.sh`'s corresponding block).

Same "belt-and-braces, not primary control" framing as `merge.sh`'s copy: explicitly redundant once a GitHub branch-protection ruleset requiring the `tier-review` status check is live; kept as the only *local* check that catches a machine-user misconfiguration before GitHub itself would.

## The `merge.sh` matcher: closing a real, previously-ungated bypass (PR #146, 2026-07-12)

Before this PR, `gate_in_scope`'s classification only matched literal `gh pr merge` (plus `git merge`/`git push`) — `scripts/merge.sh <N>`, the repo's own sanctioned merge wrapper (which calls `gh pr merge` internally), matched none of the `elif` arms. A hand-rolled `scripts/merge.sh <N>` invocation therefore sailed past the review-pr and eval-artifact gates **entirely**. This is a real, previously-ungated bypass discovered by a "loop engineering" diagnosis — not a regression; `merge.sh` was simply never covered by this hook until now.

**The matcher.** A new `elif` arm in `gate_in_scope` recognises `merge.sh` — optionally path-prefixed, optionally `bash`/`sh`-prefixed, optionally quoted — as `subcommand="merge"`, reusing the identical downstream gating `gh pr merge` already gets (Gate 6 review-pr check, Gate 7 eval-artifact check). Word-boundary anchored, same precedent as the `git merge-base` fix (PR #42, see above): a name merely *containing* `merge.sh` (`auto_merge.sh`, `some-merge.shim`) must not match. `enforce_required_step`'s PR-number extraction for this form mirrors the existing `gh pr merge` extraction (scan tokens left-to-right for the first bare integer, stripping surrounding quotes) rather than inventing a second parser style.

**Two review-found gaps closed before merge, in the same PR:**

1. **Decoy-number hijack.** The original PR-number extraction (both the `gh pr merge` and the new `merge.sh` form) scanned the raw `$cmd` string end-to-end via `grep`, so a PR number merely *mentioned* in an earlier, non-executed shell segment (e.g. `echo "run merge.sh 999 first" && scripts/merge.sh 140`) could donate its number to the later, real invocation — letting an **unreviewed** PR (140) merge under a **reviewed** decoy's number (999), a silent full bypass. Fixed by capturing the matched segment itself (`matched_seg`, new variable, same idiom as the pre-existing `push_target_dir` capture) in `gate_in_scope`, and scoping both PR-number extraction blocks to `matched_seg` instead of the raw `$cmd`.
2. **`--dry-run` passthrough bypass.** `gate_safe_passthrough`'s global `--dry-run`/`--help` exemption assumes those flags are always inert — true for `gh pr merge --dry-run` (which `gh` rejects outright as an unknown flag) and `gh pr create --help` (prints usage, never creates). But `scripts/merge.sh`'s arg parser (`merge::main`) reads only `$1` as the PR number/branch and silently ignores every trailing token — so `scripts/merge.sh 140 --dry-run` would have stood aside from this gate entirely and then performed a **real** merge of PR 140. Fixed by excluding `merge.sh` invocations from the `--dry-run`/`--help` passthrough.

**Documented, tested, non-regression residuals**: `bash -x scripts/merge.sh 140` and `command bash scripts/merge.sh 140` remain ungated — the same "not every shell form is parsed" posture the hook already stood on for `gh pr merge` before this PR, now extended and tested for `merge.sh` too.

Test suite grew from 96 to well over 110 cases across the two commits (matcher + follow-up fixes), including dedicated decoy-hijack and mismatched-PR-deny cases for both invocation forms. See [[pr_144-149_agentic-loop-hardening-from-loop-engineering]] for the full source record.

## Stdin read convention (PR #76)

This hook reads its payload via `IFS= read -r -d '' -t 5 input || true`. Context: this hook's processes were the visible symptom of the fan incident that motivated PR #76 investigation — 21 orphaned instances at 8–10% CPU for 3+ hours. However, their root cause was the config.sh walk-up infinite loop (PR #72), not `input=$(cat)`. PR #76 is defence-in-depth for the separate stdin-block risk. See [[pr_76_harden-hook-stdin-read]] and [[pr_72_config-walkup-symlink-hang]].

## Why it exists

Before this hook, the workflow chain (`/push → /review-pr → /merge`) was advisory: Claude could invoke `gh pr create` or `gh pr merge` directly, bypassing the mandated push and review steps. This hook converts those two checkpoints from advisory to mechanical. Closes review finding #C. (verified — PR #30)

## Auto-chmod

This hook is auto-chmod'd by `install.sh`'s hooks.json-derivation (PR #28). No manual chmod step needed when adding new hooks that follow the `hooks.json` registration pattern.

## Environment variables

- `CLAUDE_DISCIPLINE_LOG` — path to the shared discipline log
- `workflow.config.yaml` — presence/absence is the NO_CONFIG opt-in gate

## See also

[[enforcement-model]] — the hook/command distinction; why this is a hook not a command  
[[no_edit_on_main]] — the companion PreToolUse enforcement hook (same PR wave)  
[[discipline-loop]] — broader discipline hook composition  
[[push]] — the command this hook requires ran before `gh pr create`  
[[workflow]] — the full chain this hook enforces  
[[finishing-a-development-branch]] — the skill whose local-merge option motivated the git-merge gate  
[[skills-hooks-seam]] — the cross-reference convention this hook participates in; the merge-base regex footgun  
[[task-evals-gate]] — the dual-scope eval-artifact design; this hook is its second (hook-level) pr-scope consumer as of PR #97  
[[evals-gate-enforcement-gap_2026-07-08]] — the investigation that surfaced the raw-`gh-pr-merge` bypass PR #97 closes  
[[pr_232_tier-review-gate]] — PR #232 (2026-07-17): Gate 9, this hook's mirror of `merge.sh`'s tier-review status check; also the daemon that posts the status both gates read
[[pr_279_merge_time_smoke_reexecution]] — PR #279 (2026-07-23): Gate 8, this hook's mirror of `merge.sh`'s smoke-verify gate; makes checks 9/10's re-execution property binding at merge
[[pr_96-98_evals-gate-uniform-enforcement_2026-07-08]] — source page for PR #97 (this hook's Gate 7) and its companion PR #96/#98  
[[offload_push_guard]] — Stop/SubagentStop nudge hook (PR #108) that redirects an agent away from telling the user to push past this hook's gate from their own shell, back toward clearing it in-session with `/pr-review-toolkit:review-pr`  
[[pr_144-149_agentic-loop-hardening-from-loop-engineering]] — PR #146 source record: the `merge.sh` matcher, decoy-number hijack fix, and `--dry-run` passthrough exclusion  
[[agentic-loop]] — Phase 4b's review tier ladder (added by the same PR #146): tier-0 may skip the separate `/security-review` pass only when the diff has no hooks/scripts/auth/exec/network paths  
`coderails/hooks/scripts/enforce_pr_workflow.sh`
