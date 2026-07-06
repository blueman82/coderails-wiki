---
title: "PRs #7–10 (blueman82/coderails) — task-evals follow-ups: wiki-first prerequisite, comment-spoofing closure, install/docs refresh, tier_justification everywhere"
type: source
created: 2026-07-06
last_updated: 2026-07-06
sources: []
tags: [task-evals, evals-json, anti-gaming, merge-gate, loop-gate, comment-spoofing, gh-pagination, tier-justification, install-sh, installation-md]
---

# PRs #7–10 — task-evals follow-ups

<!-- Ingested by /wiki-ingest after merge. This is an immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR numbers | #7, #8, #9, #10 |
| Repo | `blueman82/coderails` |
| Branches | `task-evals/wu2-skill-wiki-first`, `task-evals/wu1-wu5-author-check`, `task-evals/wu3-wu6-install-docs`, `task-evals/wu8-tier-justification` |
| Merged | 2026-07-06 (all four, same day, sequentially) |
| Merge SHAs | #7 `e487b54` (head `5602b9b`), #8 `62ad18d` (head `395c416`), #9 `ffe5ccc` (head `431c48e`), #10 `238f5e1` (head `376e9fc`) |
| JIRA ticket | — |

## Summary

Four follow-up PRs closing gaps that [[pr_1-4_task-evals-feature]] (PRs #1–4, plus the #5–#6 addendum) had explicitly surfaced-but-not-fixed, plus one new owner directive. All four dogfooded the gate they extend: every PR needed both a review artifact and an eval artifact, both SHA-bound, before merge — the eval discipline gating its own follow-up work. Two of the four fix rounds were themselves caught by negative controls and independent verifier subagents during grading (a vacuous stub and a false caching comment, both described below).

- **PR #7** — `task-evals` SKILL.md gains a context-gathering prerequisite: read the project wiki before generating evals, codebase only where the wiki doesn't cover it, dispatched to a sonnet agent rather than done inline.
- **PR #8** — closes the comment-spoofing hole in both PR-gate comment readers (`scripts/lib/git-common.sh`): adds an author-identity trust filter and replaces the 100-comment-capped `gh pr view --json comments` fetch with a paginated `gh api` call.
- **PR #9** — refreshes INSTALLATION.md and fixes an `install.sh` chmod-list gap that had left two library scripts unmanaged by the installer's own exec-bit sweep.
- **PR #10** — owner directive: `tier_justification` is now required at every tier, not just tier 0, closing a bypass where a GO-graded loop artifact could lack any stated justification.

## PR #7 — wiki-first context-gathering prerequisite (WU2)

`skills/task-evals/SKILL.md`'s existing "Prerequisite: gather context before generating evals" section (added by [[pr_1-4_task-evals-feature|PR #1]]) is tightened, not newly added — review findings sharpened the wording across two commits (verified: `git show 5602b9b`):

- "the goal state, prior decisions, and known gotchas" → **"the invariants and constraints the goal state must respect, prior decisions, and known gotchas"** — more precise about what a wiki read is expected to surface before eval-writing starts.
- **No-wiki fallback clause, new**: "If the project has no wiki (`config.wiki_path` is null), the context read is codebase-only." Closes a gap where the prerequisite step had no defined behaviour for a project without a wiki at all.
- **Agentic-loop Phase 2 carve-out, new**: "Inside an agentic loop, the orchestrator's Phase 2 pre-flight wiki read already satisfies this prerequisite — reuse its findings rather than re-reading per invocation." Avoids a redundant wiki read every time `task-evals` fires inside a loop that already did one at Phase 2.
- The "why dispatch to a sonnet agent" justification was reworded from standing alone to an explicit analogy: "the same delegation pattern `agentic-loop` Phase 2 uses for its pre-flight checks."

Net effect: the prerequisite step is unchanged in shape (still wiki-first, codebase-fallback, sonnet-dispatched, non-substitutive for the five anti-gaming rules) but now has defined behaviour for the wiki-less case and an explicit reuse rule inside loops. See [[task-evals]] for the skill's full description — its "Prerequisite" framing already reflects this tightened wording.

## PR #8 — comment-spoofing closure + gh pagination (WU1+WU5)

Closes two gaps [[pr_1-4_task-evals-feature]] explicitly flagged as "surfaced, not fixed": **comment-spoofing trust model** and **`gh` comment pagination**. Both PR-gate readers in `scripts/lib/git-common.sh` — `pr::has_coderails_review_for_head` and `pr::has_coderails_eval_for_head` — are rewritten to share a new trust-filtering fetch (verified: `git diff ffe5ccc..62ad18d -- scripts/lib/git-common.sh`).

### The fix: trusted-author filter, fail-closed

Two new functions in `scripts/lib/git-common.sh`:

- **`pr::_trusted_login()`** — echoes the `gh`-authenticated user's login (`gh api user -q .login`), caching it in `_PR_TRUSTED_LOGIN` for the lifetime of the calling subshell only — the variable does **not** survive across process/subshell boundaries, so each top-level reader call still fetches fresh. Before use, the login is validated against GitHub's login charset (`^[A-Za-z0-9-]+$`) **regardless of whether it came from cache or a fresh fetch** — because it's spliced directly into a `jq --jq` program string, and an unvalidated value (e.g. a pre-seeded env var like `x" or true`) could break out of the string literal and turn the trust filter into a tautology. A value that fails validation is treated as unset (fail-closed), not passed through.
- **`pr::_trusted_comment_bodies <num>`** — fetches **all** comments for the PR via `gh api "repos/$(repo)/issues/${num}/comments" --paginate` (no 100-comment cap, unlike the old `gh pr view --json comments` GraphQL fetch this replaces), keeps only comments whose `user.login` matches the trusted identity **and** whose `author_association` is `OWNER`, and echoes bodies base64-encoded (one per line, so multi-line bodies survive as single reader lines). Untrusted comments are dropped **before** any marker matching — they can neither win nor suppress a match. Returns non-zero (fail-closed) if either the identity or comments fetch fails.

Both `pr::has_coderails_review_for_head` and `pr::has_coderails_eval_for_head` now source their comment bodies from `pr::_trusted_comment_bodies` instead of the old direct `gh pr view --json comments -q '.comments[].body'` call, decoding each base64 line before the existing marker-matching logic runs unchanged.

### Adopted design: Shape B (repo-owner login + fixed OWNER association)

Rejected alternative: a **config allowlist** of trusted logins — judged YAGNI for a personally-owned repo. Flip-condition recorded in the design spec: a CI/service-account artifact poster (a bot account distinct from the owner's own login) would justify introducing an allowlist later; not needed today.

**Known limitation, documented in code**: `OWNER` association assumes a personally-owned repo. On an org-owned repo, the same authenticated user's own comments carry `MEMBER`/`COLLABORATOR` instead of `OWNER` — so this gate would **fail closed** there (every comment untrusted, both readers report "no artifact found"), not silently pass. Widening the trust floor for org repos is a separate, not-yet-made owner decision.

### Review-round fixes (findings caught during grading, before merge)

Four issues found and fixed in the same PR, all verified against the merged diff:

1. **Mutation-proven stub tautology** — an early test stub for the trusted-fetch path always returned true regardless of input, making its own test vacuous. Fixed so the stub actually executes production's `--jq` filter expression against fixture data, rather than hardcoding the expected result.
2. **`repo()` dotted-name truncation** — the pre-existing `repo()` helper's regex (`[^/.]+` for the repo-name capture group) truncated at the first dot, so `owner/my.repo.git` resolved to `owner/my` instead of `owner/my.repo`. Fixed to a greedy `(.+)$` capture with trailing-slash and `.git`-suffix stripped **after** the match (`${name%/}`, `${name%.git}`) instead of excluded from it, so a dotted repo name is never truncated (verified: `git diff ffe5ccc..62ad18d -- scripts/lib/git-common.sh`, `repo()` function).
3. **Pre-seeded `_PR_TRUSTED_LOGIN` jq-injection** — the charset validation described above was added specifically to close this: a cached or externally-seeded login value that failed the `^[A-Za-z0-9-]+$` check would otherwise reach the `--jq` program string unescaped.
4. **False caching comment** — an early code comment claimed `_PR_TRUSTED_LOGIN` "caches the identity across calls"; corrected to state plainly that it does **not** survive across process/subshell boundaries (each top-level reader call fetches fresh) — the cache only helps within a single subshell's own lifetime.
5. **Vacuous clean-break guard literal** — a test asserting the old `gh pr view --json comments` call pattern was fully removed originally searched for a literal string that never matched the actual old source text, so it passed trivially whether or not the clean-break had happened. Fixed to an `-E` pattern that does match the old call shape, with a companion control assertion proving the pattern fails against the (deliberately reverted) old code — a negative control for the negative control.

## PR #9 — INSTALLATION.md refresh + install.sh chmod symmetry (WU3+WU6)

Two independent fixes bundled in one PR (verified: `git show ffe5ccc --stat`, `git diff e487b54..ffe5ccc -- install.sh`):

### INSTALLATION.md refresh

- The "what you get" table now covers all 28 skills, family-grouped (rather than an older, shorter enumeration).
- `/coderails:post-review` and `/coderails:post-evals` documented as commands.
- `enforce_pr_workflow` described as **config-gated**, with `/pr-review-toolkit:review-pr` named as the specific step that unblocks the `git push`/`gh pr create` gate.
- `no_edit_on_main`'s settings.json-blocking clause documented as applying on **any branch**, not just main — matching the hook's actual any-branch behaviour for `.claude/settings.json`/`.claude/settings.local.json` (see [[no_edit_on_main]]).

### install.sh chmod-list gap closed

`install.sh`'s literal script-chmod inventory (the `for script in ...` loop that arms exec bits post-install) previously listed `scripts/push.sh scripts/merge.sh scripts/lib/git-common.sh scripts/lib/eval-artifact.sh scripts/post_evals.sh` plus glob-expanded `$_lib_scripts`/`$_hook_scripts`/`$_skill_scripts`. The gap: **there is no `scripts/lib/*.sh` glob** in that loop — only scripts named literally reach it, and two library files (`scripts/lib/review-artifact.sh`, `scripts/lib/config.sh`) were missing from the literal list despite being core, exec-bit-relevant scripts. Fixed by adding both to the literal `for script in` list (verified: `install.sh:332` diff, one-line change adding both paths). This closes the same class of install-inventory gap [[pr_1-4_task-evals-feature]]'s addendum already recorded being closed for `eval-artifact.sh`/`post_evals.sh` themselves — the sweep still has no `lib/*.sh` glob, so any future `scripts/lib/*.sh` addition needs the same manual literal-list update.

### Mode-sweep test hardening

`hooks/scripts/tests/install_mode_sweep.test.sh` gains index-mode preconditions and a NOGIT-symmetry check — ensuring the mode-aware sweep behaviour PR #96 introduced (git-index-mode branching: 100755→+x, 100644→-x, untracked→legacy +x) is exercised consistently whether or not the test harness's fixture repo has a `.git` directory.

## PR #10 — tier_justification required at every tier (WU8, owner directive)

**Owner directive** (explicit, recorded verbatim in the design spec diff): `tier_justification` must be present and non-blank at **every** tier — tier 0 justifies the exemption itself (unchanged from PR #1's original rule); tier 1/2 must now state **which tier predicate fired**. Previously, only tier 0 required a justification; a tier-1/2 artifact with an empty `tier_justification` field passed both gates as long as its evals graded GO.

### Writer side: `post_evals::validate_structure` check 2 (scripts/post_evals.sh)

Check 2 rewritten from a tier-0-only conditional to an unconditional check (verified: `git diff ffe5ccc..238f5e1 -- scripts/post_evals.sh`): every artifact, regardless of tier, must carry a `tier_justification` that is non-blank **after trimming** — `jq -r '.tier_justification // "" | gsub("^\\s+|\\s+$"; "")'` — so a whitespace-only string no longer slips through as "non-empty." The error message names the actual tier (`post_evals: tier %s requires a non-blank tier_justification`) rather than assuming tier 0.

### Reader side: `als_read_loop_evals_result()` gains `UNJUSTIFIED` (hooks/scripts/lib/loop_state_common.sh)

The loop-scope reader's result vocabulary grows from `GO | TIER0 | NO-GO | ABSENT` to **`GO | TIER0 | NO-GO | UNJUSTIFIED | ABSENT`** (verified: `git diff ffe5ccc..238f5e1 -- hooks/scripts/lib/loop_state_common.sh`). The check for a non-blank, trimmed `tier_justification` now runs **first**, before checking `result`/`tier` at all — a blank justification short-circuits straight to `UNJUSTIFIED` regardless of what `result` or `tier` say. This closes a specific bypass the source comment spells out: `eval_artifact::compute_go` (the one place `.result` is derived) never inspects `tier_justification` at all, so without this reader-side check, a `GO`-graded artifact with a blank justification would silently satisfy the loop gate — the writer-side check alone (post_evals.sh, pr scope) doesn't protect the loop-scope path, which reads `evals.json` directly rather than going through `post_evals.sh`.

`UNJUSTIFIED` is kept **distinct from `NO-GO`** deliberately, so `loop_state_guard.sh`'s block message can name the actual defect (missing `tier_justification`) instead of misattributing it to a failed eval run. The guard also gained a `*` catch-all case logging `reason=unrecognised_evals_result` and failing closed on any value the reader function doesn't recognise (verified: `git diff ffe5ccc..238f5e1 -- hooks/scripts/loop_state_guard.sh`).

### Behaviour flip, explicitly logged in the source comment

"Legacy flip: pre-existing GO loop artifacts written before this check existed, and lacking tier_justification, now block (owner directive 2026-07-06) rather than silently passing as before." This is a deliberate breaking change to already-graded artifacts, not merely new-artifact-forward — the source comment names it explicitly rather than leaving it implicit.

`SKILL.md` and the design spec (`docs/coderails/specs/2026-07-03-task-evals-design.md`) were both updated to state the all-tiers requirement; the two copies were kept byte-identical per the existing convention.

## Cross-PR theme: the loop dogfooded its own gates

Every one of the four PRs required both a review artifact (`/pr-review-toolkit:review-pr` + `/coderails:post-review`) and an eval artifact (`/coderails:task-evals` + `/coderails:post-evals`), both SHA-bound, before merge — the same gates [[pr_1-4_task-evals-feature]] introduced were live and blocking for its own immediate follow-ups. Grading used independent verifier subagents per [[task-evals-gate]]'s verifier-agent contract (fresh context, no implementation conversation). Two vacuous checks were caught specifically because of the negative-control discipline: PR #8's mutation-proven stub tautology and its vacuous clean-break guard literal (see above) — both are cases where "the test passes" and "the test tests anything" had silently diverged, caught only because a negative control was required to prove each check could actually fail.

## Files changed

**PR #7**: `skills/task-evals/SKILL.md`

**PR #8**: `scripts/lib/git-common.sh`, `hooks/scripts/tests/git-common.test.sh`

**PR #9**: `INSTALLATION.md`, `install.sh`, `hooks/scripts/tests/install_mode_sweep.test.sh`

**PR #10**: `docs/coderails/specs/2026-07-03-task-evals-design.md`, `hooks/scripts/lib/loop_state_common.sh`, `hooks/scripts/loop_state_guard.sh`, `hooks/scripts/tests/loop_state_guard_evals.test.sh`, `hooks/scripts/tests/post_evals.test.sh`, `scripts/post_evals.sh`, `skills/task-evals/SKILL.md`

## Wiki pages updated

- New: this page (cluster source record for PRs #7–10)
- Updated: [[task-evals-gate]] (fail-closed/fail-open matrix gains `UNJUSTIFIED`; comment-spoofing and pagination gaps marked closed)
- Updated: [[task-evals]] (prerequisite wording tightened; tier_justification requirement extended to all tiers)
- Cross-referenced: [[engineering-principles]] (created earlier the same day, unrelated cluster — linked here only for vault navigation, no content dependency)

## Caveats / gotchas

- **Comment-spoofing closure is scoped to personally-owned repos.** `OWNER` association fails closed on an org-owned repo — this is documented as a known limitation in the code itself, not a bug. Widening it (e.g. an allowlist for CI/service-account posters) is a distinct, not-yet-made owner decision.
- **`install.sh`'s chmod sweep still has no `scripts/lib/*.sh` glob.** PR #9 closed the immediate gap for `review-artifact.sh`/`config.sh` by adding them to the literal list; any future `scripts/lib/*.sh` addition needs the same manual step repeated, not a structural fix.
- **The tier_justification flip is retroactive for already-graded artifacts.** A GO-result loop artifact written before PR #10, without a `tier_justification`, will now block at the loop gate — this was a deliberate owner choice recorded directly in the source, not an oversight.
- **PR #7's no-wiki fallback and loop-reuse clauses are prose-only.** Neither is independently hook-enforced; both rely on the invoking agent reading and following the updated SKILL.md text, same structural-unenforceability class AGENTS.md already documents for skill-level instructions in general.
