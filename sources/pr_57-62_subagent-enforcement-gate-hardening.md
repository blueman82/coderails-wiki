---
title: "PRs #57–#62 — Subagent enforcement and gate hardening"
type: source
created: 2026-06-26
last_updated: 2026-06-26
sources: []
tags: [source, hooks, subagent, enforcement, hardening, discipline]
---

# PRs #57–#62 — Subagent enforcement and gate hardening

<!-- Ingested by /wiki-ingest after merge. This is an immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR numbers | #57, #58, #59, #60, #61, #62 |
| Theme | Subagent enforcement + gate hardening |
| Merged | 2026-06-26 (verified: gh pr list --state merged) |
| Merge SHAs | #57 `06a2bf94`, #58 `73a4c69f`, #59 `02f8a908`, #60 `502f7508`, #61 `e4274fe1`, #62 `6b7d058d` (verified: gh pr view) |

## PR #57 — SubagentStop discipline

**Title:** feat(hooks): extend discipline checks to SubagentStop

`check_confidence_labels.sh` and `check_verify_loop.sh` are now also wired to the `SubagentStop` hook event in `hooks.json`. Both scripts detect `hook_event_name` and take a separate path on SubagentStop.

**Critical distinction:** `transcript_path` on a SubagentStop payload is the **parent** session transcript, not the subagent's. Reading it would check the wrong content. Both scripts instead read `.last_assistant_message` directly — the subagent's actual final output — on `SubagentStop`. (verified: check_confidence_labels.sh, check_verify_loop.sh)

For `check_verify_loop`, `file_count` is not used as a skip gate on the SubagentStop path: the subagent's message IS the authoritative output, and an untagged DNV bullet in it is proof of deferred work regardless of whether the agent transcript is readable. (On Stop, `file_count` is retained in the log line but no longer gates the check either — see PR #61.)

`loop_state_guard` and `loop_stall_guard` remain **Stop-only**: they key off main-agent loop invocation count and session-owned `progress.json`, which are parent-session concepts. A subagent has no `progress.json` to validate.

`discipline_common.sh` gained a `dc_file_count()` helper (extracted from the Stop path of `check_verify_loop`) so both SubagentStop and Stop branches share the same jq query. A fixture `subagentstop_payload.json` was added for test cases. TDD: 6 SubagentStop tests added to `check_confidence_labels.test.sh` (cases 8–13), 5 to `check_verify_loop.test.sh` (cases 10–15), including regressions proving the scripts read the right field. All 14 test suites green.

## PR #58 — enforce_pr_workflow hardened

**Title:** feat/enforce pr per pr review

Four hardening changes to `enforce_pr_workflow.sh`:

**Change A — subagent transcript support:** when `.agent_transcript_path` is present and readable in the hook payload, the transcript scan reads both `.transcript_path` and `.agent_transcript_path` for required evidence. Closes the gap where a subagent ran `/push` or `/review-pr` and the parent session had no record of it.

**Change B — per-PR / consume-on-use review evidence:**
- `gh pr merge <N>` now requires a `/pr-review-toolkit:review-pr` invocation whose `args` **starts with** the same PR number (leading-token match, so incidental occurrence of the number in prose does not satisfy the gate).
- `git merge` uses consume-on-use: review-pr must have run SINCE the last `git merge` in the transcript, not merely any time this session.
- Bare `gh pr merge` (no PR number) still accepts any review-pr (legacy behaviour preserved).

**Change C — positional `git push origin main` gated:** `gate_targets_main` now also gates `git push origin main` from any branch (bare positional bare-branch target). Previously only colon-refspec forms (`HEAD:main`, `feature:master`) were caught.

**Change D — flag-boundary tightening:** `--dry-run` / `--help` passthrough in `gate_safe_passthrough` now uses word-boundary anchors `(^|[^-[:alnum:]])(--dry-run|--help)([^-[:alnum:]]|$)` so that flags like `--dry-run-data` or `--helpfulness` don't accidentally bypass the gate.

**Review-caught Critical (fixed before merge):** the Change-C implementation initially checked only colon-refspec forms; a `git push origin main` from a feature branch was a silent false-allow. Fixed by the positional-target gate in Change C. TDD: enforce suite went to 27+ cases. (verified: PR #58 body, enforce_pr_workflow.sh)

## PR #59 — destructive_bash_gate extended

**Title:** feat/bash gate hardening

Three additions to `destructive_bash_gate.sh`:

**Extended permanent blocklist (Change #4):**
- `git clean -f` / `git clean --force` / `git clean -fd` etc.: matched via arg-extraction logic that allows dry-run (`-n`/`--dry-run`) and interactive (`-i`/`--interactive`) while denying any force flag.
- `find ... -delete` / `find ... --delete`: matched with a cross-separator anchor so `.` can't cross a shell separator (`find . -name "*.py" && rm` is not blocked).
- `truncate -s` / `truncate --size`: blocks content-destructive file truncation.
- `shred`: blocks secure-overwrite.

**Review-caught Critical (fixed before merge):** the initial `git clean` pattern used `grep -qiE '\bgit +clean.*(-f|--force)\b'` which missed combined short flags like `-fd`. Fixed to extract args after `git clean` and test for force in the arg portion. All 14 test suites green.

**Branch-aware in-Bash source edits (Change #3):** a best-effort gate blocks the tools `no_edit_on_main.sh` cannot see — `sed -i`, `perl -i`, shell redirects (`>`/`>>`), `tee`, `cp`, `mv`, `dd of=` — when targeting source files (`.py`, `.ts`, `.tsx`, `.js`, `.jsx`, `.go`) or plugin source (`skills/*/SKILL.md`, `commands/*.md`) on `main`/`master`. Branch detection uses target-repo resolution (the file's own repo branch), mirroring `no_edit_on_main.sh`. Best-effort: variable filenames, quoted paths with spaces, here-docs, process substitution, and `python -c open(...)` writes remain uncaught. This is a documented ceiling (see PR #62 ceilings note).

## PR #60 — no_edit_on_main code arm inverted to allowlist

**Title:** feat/no edit main allowlist

`no_edit_on_main.sh` was previously a **blocklist**: it blocked a hardcoded set of 6 code extensions (`.py`, `.ts`, `.tsx`, `.js`, `.jsx`, `.go`) on main/master and let everything else through.

The code arm is now an **allowlist**: on main/master, the hook blocks edits to everything **except** an explicit set of doc/config types:

| Allowed category | Extensions / filenames |
|---|---|
| Docs | `.md`, `.txt`, `.rst` |
| Config | `.yaml`, `.yml`, `.json`, `.toml`, `.ini`, `.cfg` |
| Special dotfiles | Literal `.gitignore` basename only (not `deploy.gitignore`) |
| Bare filenames | `LICENSE` |

Everything else — `.sh`, `.py`, `.ts`, `.go`, `.rb`, etc. — is now blocked on main. The plugin-source markdown arm (`skills/*/SKILL.md`, `commands/*.md`) is checked FIRST and remains blocked unconditionally (PR #44 behaviour preserved). The `.gitignore` special-case was tightened from `*.gitignore` (which would have allowed `deploy.gitignore`) to a basename match — a TDD-discovered bug fixed in PR #62. PR #52 cross-repo correctness (file's own repo branch, not session cwd) is preserved.

## PR #61 — check_verify_loop file_count gate removed on Stop path

**Title:** feat/verify loop dnv independent

On the **Stop** path of `check_verify_loop.sh`, the `file_count < 1` early-exit gate was removed. A `## Did Not Verify` section that exists must be policed regardless of whether any files were edited this turn — a pure-conversation response can carry deferred verifiable claims too.

`file_count` is still computed and written to the log line, but no longer short-circuits the check. The SubagentStop path (added in PR #57) already had no such gate, so both paths are now consistent: presence of an untagged DNV bullet is sufficient to block.

## PR #62 — docs consolidation: hook-map sync, ceilings, spec, .gitignore fix

**Title:** docs/hardening hookmap ceilings

Four tasks:

**Task A — hook-map sync:** `CLAUDE.md` and `docs/REFERENCE.md` hook-map tables updated to reflect #57–#61 changes: SubagentStop row added, enforce_pr_workflow description updated, destructive_bash_gate description updated, no_edit_on_main description updated, check_verify_loop description updated.

**Task B — "Enforcement ceilings" section** added to `CLAUDE.md`: honest documentation of what the hooks deliberately do NOT fully cover. Ceilings include: bash blocklists are enumerated families (not exhaustive); in-Bash gate is best-effort; wiki/workflow sequence post-merge is advisory; check_verify_loop and loop guards short-circuit at most once per turn; TDD is not enforced test-first; skill invocation is structurally unenforceable; no SubagentStart event exists.

**Task C — spec doc:** recreated the design spec from orphaned commit `8a8d1b6` into `docs/coderails/specs/2026-06-26-subagent-enforcement-and-gate-hardening-design.md`.

**Task D — .gitignore allowlist fix:** `no_edit_on_main.sh` had a `*.gitignore` arm that would have allowed `deploy.gitignore` edits on main. Fixed to match only the literal `.gitignore` basename. TDD-driven: tests for `deploy.gitignore → DENY` and `src/.gitignore → ALLOW` added first.

## Wiki pages updated

[[check_confidence_labels]] — SubagentStop event, last_assistant_message path  
[[check_verify_loop]] — SubagentStop event, file_count gate removal on Stop path  
[[destructive_bash_gate]] — extended blocklist, in-Bash source-edit gate  
[[enforce_pr_workflow]] — per-PR review evidence, consume-on-use, positional push, flag-boundary tightening, subagent transcript support  
[[no_edit_on_main]] — allowlist inversion, .gitignore tightening  
[[discipline-loop]] — SubagentStop composition, enforcement ceilings  
[[enforcement-model]] — updated hook map  

## Caveats / gotchas

- On SubagentStop, `transcript_path` is the **parent** session transcript. Both discipline hooks read `.last_assistant_message` instead — confusing the two fields is the most likely future bug if a contributor adds a new SubagentStop hook.
- The in-Bash source-edit gate (`destructive_bash_gate` Change #3) is **best-effort** by design. Variable filenames, quoted paths, and process substitution are not caught. This is documented as an enforcement ceiling, not a bug.
- `no_edit_on_main`'s allowlist model means `.sh` scripts are now **blocked** on main, which is stricter than before. Any legitimate hotfix to a `.sh` on main requires a settings.json `Edit` permission override.
- Per-PR review evidence in `enforce_pr_workflow` uses a leading-token match on the PR number, not an equality check. `review-pr 42` satisfies the gate for `gh pr merge 42`; `review-pr 420` does not satisfy `gh pr merge 42` (420 does not start with 42 followed by a non-digit). This is by design.
