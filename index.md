---
title: Coderails Wiki Index
type: index
created: 2026-05-30
last_updated: 2026-06-30
sources: []
tags: [index, catalog]
---


# Coderails Wiki — Content Catalog

This is the first file Claude reads when answering queries about the coderails plugin.
Schema and maintenance protocols are in `/Users/harrison/Github/coderails/AGENTS.md`.

## What this plugin is

`coderails` is a Claude Code plugin (not an application) that ships three things:
1. A workflow command chain: `prep → push → merge → wiki`
2. Skills: `agentic-loop`, `test-driven-development`, `writing-plans`, `handoff`, `improve-prompt`, `planning-sequence`, `premortem`, `wiki-ingest`, `wiki-init`, `wiki-lint`, `wiki-query`, `using-coderails`, `using-git-worktrees`, `requesting-code-review`, `receiving-code-review`, `finishing-a-development-branch`, `dispatching-parallel-agents`, `systematic-debugging`, `verification-before-completion`, `writing-skills`, `subagent-driven-development`, `executing-plans`, `brainstorming`
3. A discipline loop: hooks that enforce confidence labels, verify-loop resolution, destructive-bash blocking, main-branch protection, and PR-workflow sequencing. **superpowers is no longer a dependency** — all core dev-workflow skills are vendored into the `coderails:` namespace. See [[self-containment]].

Source: `coderails/CLAUDE.md`. Plugin manifest: `coderails/.claude-plugin/plugin.json`.

---

## Commands

Slash commands in `coderails/commands/`. Advisory only — Claude chooses to invoke them.
See [[enforcement-model]] for why hooks enforce and commands do not.

| Command | File | Status |
|---|---|---|
| `/coderails:workflow` | `commands/workflow.md` | [[workflow]] |
| `/coderails:prep` | `commands/prep.md` | [[prep]] |
| `/coderails:push` | `commands/push.md` | [[push]] |
| `/coderails:merge` | `commands/merge.md` | [[merge]] |
| `/coderails:init` | `commands/init.md` | [[init]] |
| `/coderails:test-gate-setup` | `commands/test-gate-setup.md` | [[test-gate-setup]] |
| `/coderails:assumptions` | `commands/assumptions.md` | [[assumptions]] |
| `/coderails:disconfirm` | `commands/disconfirm.md` | [[disconfirm]] |
| `/coderails:notchecked` | `commands/notchecked.md` | [[notchecked]] |
| `/coderails:verify` | `commands/verify.md` | [[verify]] |

Config resolution pattern shared by `workflow.md`, `prep.md`, `push.md`, `init.md`: [[config-resolution]]

---

## Hooks

Hook event wiring in `coderails/hooks/hooks.json`. Scripts in `coderails/hooks/scripts/`.
See [[discipline-loop]] for the design rationale across all hooks.

| Script | Event | Mode | Status |
|---|---|---|---|
| `inject_bootstrap.sh` | `SessionStart` | silent — bootstraps session with `coderails:using-coderails` | [[inject_bootstrap]] |
| `inject_context.sh` | `UserPromptSubmit` | silent — injects `[ctx]` date/cwd/branch | [[inject_context]] |
| `discipline_catchup.sh` | `UserPromptSubmit` | warn — re-injects discipline nudge if prior response missed labels | [[discipline_catchup]] |
| `check_confidence_labels.sh` | `Stop` + `SubagentStop` | **block** (exit 2) — ≥200-char response with no label; reads `last_assistant_message` on SubagentStop (PR #57) | [[check_confidence_labels]] |
| `check_verify_loop.sh` | `Stop` + `SubagentStop` | **block** (exit 2) — any untagged `## Did Not Verify` bullet; file_count gate removed on Stop path (PR #61); reads `last_assistant_message` on SubagentStop (PR #57) | [[check_verify_loop]] |
| `loop_state_guard.sh` | `Stop` | **block** (exit 2) — agentic loop active but `progress.json` absent / session-mismatch (presence + ownership) | [[loop_state_guard]] |
| `loop_stall_guard.sh` | `Stop` | **block** (exit 2) — agentic loop active + incomplete + no `LOOP-STOP: <category>` declaration | [[loop_stall_guard]] |
| `destructive_bash_gate.sh` | `PreToolUse (Bash)` | **block** (permissionDecision: deny) — extended blocklist (git clean -f, find -delete, truncate, shred) + in-Bash source edits on main (PR #59) | [[destructive_bash_gate]] |
| `test_gate.sh` | `PreToolUse (Bash)` | **block** on `git commit` if tests fail — opt-in via `.claude/test_command` | [[test_gate]] |
| `enforce_pr_workflow.sh` | `PreToolUse (Bash)` | **block** — `gh pr create` without prior `/push`; `gh pr merge`/`git merge`/`git push` on/targeting main without prior `/review-pr` (per-PR + consume-on-use + positional push, PR #58; NO_CONFIG opt-in) | [[enforce_pr_workflow]] |
| `no_edit_on_main.sh` | `PreToolUse (Write\|Edit\|MultiEdit)` | **block** — allowlist model (PR #60): everything except doc/config/special dotfiles on main/master; plugin-source markdown still blocked; `.claude/settings.json`/`.local.json` blocked on **any** branch (PR #70) | [[no_edit_on_main]] |

Stop hook order: `check_confidence_labels` → `check_verify_loop` → `loop_state_guard` (C1) → `loop_stall_guard` (C2). SubagentStop hook order (added PR #57): `check_confidence_labels` → `check_verify_loop` (loop guards are Stop-only). The two loop-state hooks share `hooks/scripts/lib/loop_state_common.sh` (vocab, active-loop detection, and named `als_gate_*` functions for Gates 1–4; extracted PR #49) and `hooks/scripts/lib/agentic_loop_path.sh` (sole path authority). The three discipline hooks share `hooks/scripts/lib/discipline_common.sh` (transcript extraction + retry loop + `dc_file_count()` helper; added PR #29, extended PR #57). See [[spec-plan-progress-artifact-chain]].

---

## Skills

Skills in `coderails/skills/*/SKILL.md`.

| Skill | Trigger summary | Status |
|---|---|---|
| `agentic-loop` | Multi-PR autonomous sessions; sits above `/workflow` | [[agentic-loop]] |
| `test-driven-development` | Worker construction discipline (red-green-refactor); code-guarded | [[test-driven-development]] |
| `writing-plans` | Turn a resolved spec into a durable task-by-task `plan.md` (Phase 2.8) | [[writing-plans]] |
| `handoff` | Generates memory file + continuation prompt for next session | [[handoff]] |
| `improve-prompt` | Rewrites underspecified prompts using 7 diagnostic foundations; Phase -1 of agentic-loop | [[improve-prompt]] |
| `planning-sequence` | Pre-Parade → Premortem → Red Team on a plan | [[planning-sequence]] |
| `premortem` | Backwards-failure-mode analysis on a plan or decision (standalone; embedded in planning-sequence Stage 2) | [[premortem]] |
| `wiki-ingest` | Primary wiki write operation; translates a PR or decision into permanent vault entries | [[wiki-ingest]] |
| `wiki-init` | One-time vault bootstrap: creates vault, seeds AGENTS.md, wires Obsidian + qmd | [[wiki-init]] |
| `wiki-lint` | Wiki structural integrity auditor: contradictions, stale pages, orphans, missing cross-refs | [[wiki-lint]] |
| `wiki-query` | Wiki read operation; answers questions with citations, files hard answers back as investigations | [[wiki-query]] |
| `using-coderails` | Session orientation; auto-loaded at SessionStart via inject_bootstrap hook | [[using-coderails]] |
| `using-git-worktrees` | Git worktree mechanics for parallel agent isolation | [[using-git-worktrees]] |
| `requesting-code-review` | PR review submission discipline; sits at push→review boundary | [[requesting-code-review]] |
| `receiving-code-review` | Acting on review findings; triage by severity | [[receiving-code-review]] |
| `finishing-a-development-branch` | Final verification before merge; branch wrap-up | [[finishing-a-development-branch]] |
| `dispatching-parallel-agents` | Fan-out to parallel subagents with worktree isolation | [[dispatching-parallel-agents]] |
| `systematic-debugging` | Hypothesis-then-verify loop before touching code | [[systematic-debugging]] |
| `verification-before-completion` | Structured checklist against requirements before declaring done | [[verification-before-completion]] |
| `writing-skills` | Meta-skill for authoring new coderails skills | [[writing-skills]] |
| `subagent-driven-development` | Delegation pattern: implementer prompts → subagent → verify loop | [[subagent-driven-development]] |
| `executing-plans` | Executing a `plan.md` task list with scope discipline | [[executing-plans]] |
| `brainstorming` | Structured ideation + Decision Ledger before committing to a design | [[brainstorming]] |

---

## Design

Architectural decisions and invariants.

- [[enforcement-model]] — hooks vs. commands; the central design distinction
- [[discipline-loop]] — how the six discipline hooks compose into a coherent loop
- [[config-resolution]] — walk-up `workflow.config.yaml` resolver (`scripts/lib/config.sh`) shared by all workflow commands + the merge-gate hook
- [[install-and-cache-trap]] — idempotency contracts for `install.sh`/`uninstall.sh` and the reload-plugins caveat
- [[hook-exit-codes]] — which hook events block on exit 2; why coderails uses two block mechanisms
- [[repo-hosting]] — where the plugin + wiki repos live (private, `blueman82`), clone locations, `.gitignore` rules
- [[spec-plan-progress-artifact-chain]] — the agentic-loop's full spec→plan→progress durable-artifact chain + the two-hook (C1 presence/ownership, C2 anti-stall) loop-state guard architecture
- [[self-containment]] — the 2026-06-25 decision to vendor 12 superpowers skills into coderails; superpowers now uninstallable
- [[skills-hooks-seam]] — the cross-reference obligation between skills (advisory) and hooks (mechanical); the merge-base regex footgun; added PR #42

---

## Investigations

Point-in-time filed investigations (`<topic>_<YYYY-MM-DD>.md` naming).

- [[install-cache-trap_2026-05-30]] — initial investigation filed at wiki bootstrap
- [[install-bash32-bad-substitution_2026-06-01]] — `${answer,,}` (bash 4) broke install.sh on macOS bash 3.2; fixed with `tr` lowercase
- [[skill-testing-state_2026-06-26]] — coderails has ZERO automated skill testing; the lone planning-sequence evals.json was a dormant skill-creator quality-eval artifact (deleted, PR #55); skill-creator (external) has two eval systems (trigger `run_eval.py` + agent-graded quality); decision NOT to build a runner (corpus=1); candidate reminder-hook deferred

---

## Sources

Ingested PR and change records (`sources/pr_<N>_*.md`; session sources for direct-edit work).

- [[session_2026-05-31_verify-loop-hardening]] — verify-loop rework (meta-exclusion, threshold, string-coercion), cache-trap, exit-2 findings
- [[session_2026-05-31_prompting-doc-alignment]] — prompting-doc alignment: --no-verify gate, agentic-loop context/delegate guidance, discipline scope+floor clauses, workflow parallel-calls
- [[session_2026-06-01_verify-loop-total-enforcement]] — verify-loop escalated to total enforcement: source-token regex + meta_pattern removed, any untagged DNV bullet blocks, `(unverifiable: …)` is the sole escape
- [[session_2026-06-01_agentic-loop-delegate-all-impl]] — agentic-loop: main context becomes pure orchestrator; every code change (even single-file) goes to sonnet agent for impl+verify; two-rung delegation ladder; new Phase 3a (plugin commit 3c33f99)
- [[session_2026-06-25_agentic-loop-upgrade-arc]] — the agentic-loop upgrade arc (PRs #12–#18): Specs A (clean-migration), C1 (progress.json lifecycle), C2 (anti-stall), B (slim), D (vendored TDD), E (spec→plan artifacts), + #18 tool-agnostic cleanup
- [[pr_19-30_self-containment-and-hardening]] — self-containment + plugin hardening cluster (PRs #19–#30): 12 skills vendored from superpowers, SessionStart bootstrap hook, no_edit_on_main + enforce_pr_workflow enforcement hooks, install.sh chmod derivation, discipline_common.sh extraction
- [[pr_39_agentic-loop-slim-v2]] — agentic-loop slim v2 (reference-not-embody): PR #39 normalised 13 bare skill refs to `/coderails:` form; two independent passes against a four-part test found ZERO further passages to slim — the embodied content is irreducible autonomy supersets. Do not re-attempt
- [[pr_40_hook-hardening]] — PR #40 hook hardening (2026-06-25): enforce_pr_workflow gains a `git merge` gate on main/master (closes local-merge bypass); inject_bootstrap jq refactor (escape_for_json removed, clean-break); new run_all.sh aggregate test runner with zero-test guard (13 suites)
- [[pr_41_phase25-brainstorming-xref]] — PR #41 (2026-06-25): Phase 2.5 design-fork resolver now applies brainstorming's design-quality discipline (YAGNI, design-for-isolation) by reference, without brainstorming's human-approval gates
- [[pr_42_skills-hooks-seam]] — PR #42 (2026-06-25): merge-base regex exclusion bug fix; reordered git-merge hint; skills↔hooks seam convention; finishing-a-development-branch Option 1 hook note
- [[pr_43_rough-edges]] — PR #43 (2026-06-25): failure_log template cleaned (3 stale dev-history rows removed); GitHub-only constraint made user-facing in README; merge.sh enforcement-gap notice when enforce_pr_workflow is inactive
- [[pr_44_no-edit-plugin-source]] — PR #44 (merged 2026-06-25): no_edit_on_main now gates plugin-source markdown (`skills/*/SKILL.md`, `commands/*.md`) on main, narrowing the docs carve-out; deliberate decision NOT to gate `git push` (edit-time is the right seam); `docs/superpowers` historical tree removed
- [[pr_46_gate-git-push-on-main]] — PR #46 (merged 2026-06-26, 7a4906c): enforce_pr_workflow gains a `git push` gate mirroring the `git merge` gate — blocks pushes that land on main/master (current branch OR explicit destination refspec like `HEAD:main`) without prior `/review-pr`. Destination-refspec model (not current-branch only); metachar-anchored against `HEAD:main;echo` evasion; bare positional `git push origin main` is a documented unparsed limitation. Review caught a Critical current-branch-only false-allow; tests 14→27. Distinct from #44's no_edit_on_main push decision
- [[pr_47_strictcode-skill-config]] — PR #47 (merged 2026-06-26, f00ed54): `strictcode_skill` promoted from hardcoded `/strictcode-python` to a configurable `workflow.config.yaml` field. `/coderails:init` now prompts with language auto-detection (go.mod → /strictcode-go; package.json+*.ts → /strictcode-ts; else /strictcode-python). [[push]] and [[workflow]] read `config.strictcode_skill` (default `/strictcode-python`). `workflow.md` allowed-tools expanded to pre-authorise `/strictcode-go` and `/strictcode-ts`. Fully backward-compatible
- [[pr_50_planning-sequence-gate]] — PR #50 (merged 2026-06-26, 03b5b11): [[writing-plans]] gains a mandatory `/coderails:planning-sequence` gate — after the self-review gate and before implementation hand-off, the written plan runs through Pre-Parade → Premortem → Red Team and findings fold back inline. Advisory (no hook). Also relocated engineering-principles spec/plan into `docs/coderails/{specs,plans}/` and landed design docs for the not-yet-implemented strictcode→engineering-principles vendoring. [[planning-sequence]] reframed: now a required *downstream* gate, not only an optional upstream step
- [[pr_49_gate-function-rename]] — PR #49 (merged 2026-06-26, 0426312): positional `# Gate N` comments replaced with named bash functions in `enforce_pr_workflow.sh` (7 functions: `gate_has_command`, `gate_safe_passthrough`, `gate_in_scope`, `gate_config_present`, `gate_targets_main`, `gate_have_transcript`, `enforce_required_step`). Gates 1–4 byte-identical between `loop_state_guard.sh` and `loop_stall_guard.sh` extracted into new shared lib `hooks/scripts/lib/loop_state_common.sh` (`als_gate_*` / `als_load_progress`); each guard keeps own Gates 5–6. Known limitation documented: `enforce_pr_workflow` checks invocation evidence not completion — fix deferred. CLAUDE.md hook conventions paragraph corrected (scoped to 3 scripts; removed false claim about `check_verify_loop.sh`). `.worktrees` added to `.gitignore`
- [[pr_54_engineering-principles-vendoring]] — PR #54 (merged 2026-06-26, 2f1ad1c): vendored the global `strictcode` family into coderails as the **`engineering-principles`** family (4 skills), stripping Scout/SlimCode (kept Serena + Grep fallback). Renamed config `strictcode_*` → `engineering_principles_*` and default `/strictcode-python` → `/engineering-principles-python` across [[push]]/[[workflow]]/[[prep]]/[[init]]/[[config-resolution]]. Added the PR-review touchpoint (`workflow` Phase 3 `2b` /engineering-principles + `2c` /simplify). README 23→27 / four groups. Globals removed. **Supersedes [[pr_47_strictcode-skill-config]]**
- [[pr_55_remove-dead-evals]] — PR #55 (merged 2026-06-26, ce499d7): deleted the orphaned `skills/planning-sequence/evals/evals.json` — a dormant skill-creator quality-eval artifact referenced by nothing in coderails, falsely implying a test harness existed. See [[skill-testing-state_2026-06-26]]
- [[pr_56_writing-skills-using-coderails-example]] — PR #56 (merged 2026-06-26, 0f0969e): [[writing-skills]] doc fixes — stale `using-skills`→`using-coderails` naming example corrected, and the new `examples/CLAUDE_MD_TESTING.md` worked test-campaign added (closes a dangling reference in `testing-skills-with-subagents.md`). Documents the *manual* skill-testing ritual only — no automation added
- [[pr_57-62_subagent-enforcement-gate-hardening]] — PRs #57–#62 (merged 2026-06-26): subagent enforcement + gate hardening cluster. #57: SubagentStop discipline for check_confidence_labels + check_verify_loop (reads last_assistant_message, not parent transcript). #58: enforce_pr_workflow hardened — per-PR review evidence, consume-on-use, positional git push origin main, flag-boundary tightening, subagent transcript. #59: destructive_bash_gate extended — git clean -f/--force, find -delete, truncate, shred + in-Bash source-file edits on main (best-effort). #60: no_edit_on_main code arm inverted to allowlist (everything except doc/config/special dotfiles blocked on main). #61: check_verify_loop file_count gate removed on Stop path (DNV enforcement independent of file edits). #62: docs sync + Enforcement ceilings section + .gitignore allowlist over-match fix
- [[pr_63_remove-failure-log]] — PR #63 (merged 2026-06-26, 18392fb): deleted the orphaned `templates/failure_log.md` + all plumbing (install/uninstall/CLAUDE.md/INSTALLATION.md). It was the data input for claude-guardrails' removed weekly calibration ritual — consumer gone, zero entries in two months. An automation plan (`/failure` + `/failure-review`) was DECLINED: empty log = demand signal, not friction signal. Records the caught-vs-escaped failure distinction (only caught failures are hook-capturable). Fixed the [[discipline-loop]] Founding-Thesis dead pointer (quote preserved in-wiki). Kept: docs/coderails-review.md (historical) + README "no calibration ritual" line
- [[pr_64_loop-review-via-skill]] — PR #64 (merged 2026-06-26, ee44aa0): agentic-loop Phase 4b changed to invoke `/pr-review-toolkit:review-pr <PR#>` as a Skill (with PR number as arg) instead of hand-rolling six reviewer agents. Root cause: enforce_pr_workflow only recognises Skill invocations as merge evidence; manual agent fanout leaves no recognisable record. CLAUDE.md skills↔hooks seam convention extended with one sentence naming this constraint
- [[pr_69_no-edit-message-worktree]] — PR #69 (merged 2026-06-29, 86e701a): [[no_edit_on_main]] deny message + header comment reworded to recommend an isolated **worktree + branch** (`/coderails:prep` or `git worktree add <path> -b <name>`) instead of a plain feature branch. Prose-only — block conditions and the 20/20 test suite unchanged. Records a gotcha: `scripts/merge.sh` exits 128 (post-merge `git checkout main`) when run from inside a linked worktree, though the merge itself lands
- [[pr_70_gate-settings-json-edits]] — PR #70 (merged 2026-06-29, e52e541; supersedes closed #68): [[no_edit_on_main]] gains a **permission-file arm** blocking `.claude/settings.json`/`.claude/settings.local.json` edits on **any** branch — the first non-main arm. Closes the self-escape door: the `permissions.allow` rules pre-approve commands upstream of every gate, so editing them dismantles the discipline layer. Friction not lock (Bash-path edits + plugin-disable still open; branch protection is the real lock). Collateral: retargeted `ceiling_note.test.sh` CLAUDE.md→AGENTS.md (stale after #67). Process gotcha: force-push to rebase #68 was blocked by [[destructive_bash_gate]], so a fresh branch was used — the gate forcing its own safe path
- [[pr_72_config-walkup-symlink-hang]] — PR #72 (merged 2026-06-29, 3862614): fixed an **infinite-loop hang** in `scripts/lib/config.sh::coderails::config_path()`. Walk-up terminated only on `d == git_root` string equality, but `git rev-parse` returns symlink-resolved paths (macOS /tmp→/private/tmp); on the NO_CONFIG path the loop spun at `dirname /` forever. Surfaced as a hang in [[enforce_pr_workflow]]'s "no workflow.config.yaml" test case, wedging `run_all.sh`. Bug from #67 (walk-up) + #71 (extracted `config.sh` with no test). Fix: `pwd -P` canonicalisation + hard `/` floor. Added `config.test.sh` (watchdog-guarded reproducer; fails pre-fix, passes post-fix). Suite 14→15, now 307 ok / 0 FAIL. Updated stale [[config-resolution]] (still documented the pre-#67 dual-path lookup). Lesson: absence of a FAIL line ≠ pass when the process was killed mid-hang
- [[pr_74_config-test-watchdog-stall]] — PR #74 (merged 2026-06-29, 55e0d63): the watchdog #72 added to `config.test.sh` was correct but **slow** — every `resolve()` call paid the full 5 s `sleep` (~25 s of the ~45 s suite). Root cause: `kill $w` orphans the watchdog's `sleep` child, which keeps the `$()` pipe write-fd open, so `$()` blocks the full 5 s — a pipe-fd lifetime issue, **not** the "ignores SIGTERM" the first comment claimed. Fix: `pkill -P $w` kills the sleep directly (closes fd); `{ } 2>/dev/null` swallows the job-control notice. Suite ~45 s→~21 s. Review (comment-analyzer) caught the false comment; code-reviewer self-retracted a wrong narrower-fix. Lesson: a comment that misexplains the mechanism is worse than none
- [[pr_75_main-branch-fallback]] — PR #75 (merged 2026-06-29, feaefe9): the `main()` helper in `scripts/lib/git-common.sh` returned a **blank string** instead of falling back to literal `main` when `git symbolic-ref refs/remotes/origin/HEAD` was unset. Root cause: in `git ... | sed ... || echo main`, the `||` binds to the **pipeline's** exit status, which is `sed`'s — `sed` exits 0 on empty input, so the fallback never fired. Fix: capture the result, `echo "${m:-main}"` (key the fallback off empty **output**, the real symptom). Latent — the marker is set by `git clone`/`gh`, so only bare-`git init` test scaffolding ever hit the blank; starved `sync::main_branch` (added by un-ingested #73) into a silent no-op. 3 reviews (code/tests/silent-failure) clean, 0 findings. Same pipeline-mechanism family as #74. Lesson: `cmd | filter || fallback` keys the fallback off `filter`'s exit, not `cmd`'s
- [[pr_76_harden-hook-stdin-read]] — PR #76 (merged 2026-06-29, 37459a3): replaced `input=$(cat)` with `IFS= read -r -d '' -t 5 input || true` in all 10 hook scripts. Defence-in-depth for a latent stdin-blocking risk (orphaned hook blocking forever if parent dies without closing stdin). NOT the fix for the fan incident (config.sh walk-up loop, PR #72). On read timeout, input empty → exit 0 (fail-open is safe: dead parent = no live pipeline). `|| true` mandatory (`read -d ''` exits 1 on normal EOF). Added `stdin_bounded_read.test.sh` (guard + fidelity tests, TDD). Security: CLEAN 9/10. ~~Known gap: no automated `min(hooks.json timeout) >= 5` guard~~ → **Closed by PR #78**
- [[pr_77_agentic-loop-sync-docs-step]] — PR #77 (merged 2026-06-29, cefb47c): agentic-loop Phase 9 gains a `/sync-docs` docs-drift check at the loop boundary. Wiki ingest (external KB) and `/sync-docs` (in-tree docs) are now explicitly separated as the two loop-boundary doc steps. `/sync-docs` runs without Serena (`--semantic` optional), is delegated to a spawned agent, and follows the same finding-triage discipline as Phase 5 (fix only loop-introduced drift; surface pre-existing drift to the user)
- [[pr_78_hooks-json-timeout-floor]] — PR #78 (merged 2026-06-29, ceabb75): closes the known gap from PR #76 — added `hooks/scripts/tests/hooks_json_timeout_floor.test.sh` asserting `min(declared hooks.json timeout) >= 5` (Half A of the timeout invariant). **Both halves of the invariant are now fully guarded by PR #80** — see [[pr_80_guard-read-t-floor]]
- [[pr_79_sync-docs-drift]] — PR #79 (merged 2026-06-29, 0bcb44e): fixes 4 pre-existing doc-drift findings surfaced by `/sync-docs` (Phase 9, PR #77). F1 (High): `docs/REFERENCE.md` missing 4 engineering-principles skills + false "complete catalogue" claim — added section, reconciled count 23→27. F2 (Medium): `README.md` listed bare `truncate` in destructive_bash_gate blocklist — corrected to `truncate -s/--size`. F3 (Low): `README.md` + `AGENTS.md` omitted inject_context.sh's first-prompt discipline reminder — added. F4 (Low): `AGENTS.md` said "four core gate scripts" omitting test_gate.sh — corrected to "five". Docs-only change; no runtime impact

---

## Templates

Skeletons for each page type live in `templates/`. Use them when creating new pages:

- `templates/command.md`
- `templates/hook.md`
- `templates/skill.md`
- `templates/design.md`
- `templates/investigation.md`
- `templates/source.md`
