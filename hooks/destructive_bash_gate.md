---
title: "Hook: destructive_bash_gate"
type: hook
created: 2026-05-31
last_updated: 2026-07-17
sources:
  - hooks/scripts/destructive_bash_gate.sh
  - sources/session_2026-05-31_prompting-doc-alignment.md
  - sources/pr_57-62_subagent-enforcement-gate-hardening.md
  - sources/pr_76_harden-hook-stdin-read.md
  - sources/pr_69_2026-07-08_substitution-bypass-audit.md
  - sources/pr_72-75_2026-07-08_force-with-lease-allowlist.md
  - sources/pr_84_2026-07-08_git-global-option-bypass.md
  - sources/pr_92_2026-07-08_reference-drift-and-lookalike-fp.md
  - sources/pr_201_202_203_routine-followups.md
  - sources/pr_216_217_safe-routes-and-cost-miner-diagnostics.md
  - sources/pr_224_231_233_235_loop-tooling-hardening.md
tags: [hook, pretooluse-hook, enforcement, destructive-bash, block, pattern-id, mention-safe]
---

# Hook: destructive_bash_gate

A `PreToolUse (Bash)` lifecycle hook that permanently blocks a fixed set of destructive shell commands before they run, with **one narrow conditional-allow carve-out** (`git push --force-with-lease`, opt-in, see below). Extended in PR #59 to cover additional force-clean, truncate, shred commands plus a best-effort branch-aware gate on in-Bash source file edits. Hardened again in a same-day, five-PR arc on 2026-07-08 (#69, #72+#75, #84, #92) that closed three independent bypasses and one false-positive — see "2026-07-08 adversarial-hardening arc" below. Deny messages gained a per-pattern named safe route on 2026-07-17, first for 3 patterns (PR #203), then extended to **all ~13 blockable patterns** the same day (PR #216) — see "Deny messages name a concrete safe route" below. Each route arm also gained a mention-safe `patternId` output field the same day (PR #233) — see "Mention-safe `patternId` per case arm" below.

Source: `hooks/scripts/destructive_bash_gate.sh`

## Event and mode

| Field | Value |
|---|---|
| Event | `PreToolUse (Bash)` |
| Mode | **block** (`permissionDecision: "deny"`, exit 0) |
| Timeout | 5 seconds (hooks.json) |

## Logic summary

The hook reads the Bash `tool_input.command` string from the payload via `jq`. If the field is absent or empty, it exits 0 immediately (verified: destructive_bash_gate.sh:9–11). Otherwise it runs a single `grep -qiE` against a conservative destructive-pattern regex. (verified: destructive_bash_gate.sh:16)

The hook has no skip gates beyond the empty-command guard. It is stateless — no transcript read, no file count, no session check.

## Blocked patterns

The hook runs two tiers of checks:

### Extended blocklist (added PR #59)

Handled with dedicated arg-extraction logic before the main regex:

| Command | What it catches | Notes |
|---|---|---|
| `git clean -f`, `git clean --force`, `git clean -fd` etc. | Force-clean working tree | Allows dry-run (`-n`/`--dry-run`) and interactive (`-i`); denies any force flag in combined or separated form |
| `find ... -delete` / `find ... --delete` | Recursive deletion via find | Cross-separator anchored so the pattern doesn't cross `&&`/`;`/`\|` boundaries |
| `truncate -s` / `truncate --size` | File-content truncation | Blocks size-destructive truncate |
| `shred` | Secure file overwrite/deletion | |

### `git push --force`/`-f`/`--force-with-lease` — carved out of the main regex (2026-07-08)

As of the 2026-07-08 hardening arc (below), force-push detection is no longer part of the monolithic main regex — it moved to its own block earlier in the script, because POSIX ERE alternation (`grep -E`, no `BASH_REMATCH`) can't report *which* alternative matched, and the gate needs to distinguish "naked force" (always deny) from "force-with-lease" (conditionally allow). See "Force-with-lease allowlist carve-out" below for the current logic. The main regex below no longer includes any `git push` pattern.

### Original permanent blocklist (main regex)

```
\brm +(-[rRfF]+|--recursive|--force)
|\bgit +reset +--hard
|\bDROP +(TABLE|DATABASE|SCHEMA)\b
|\bTRUNCATE +TABLE\b
|\bdd +if=
|\bmkfs\.
|\bchmod +-R +777
|\bgit +commit +.*--no-verify
```

| Pattern | What it catches |
|---|---|
| `rm -rf`, `rm -r`, `rm -R`, `rm -f`, `rm --recursive`, `rm --force` | Recursive/force file deletion |
| `git reset --hard` | Hard history rewind, discarding local changes |
| `DROP TABLE/DATABASE/SCHEMA` | Destructive SQL DDL |
| `TRUNCATE TABLE` | Destructive SQL DML |
| `dd if=` | Raw disk write |
| `mkfs.*` | Filesystem formatting |
| `chmod -R 777` | World-write permission on a tree |
| `git commit --no-verify` | Hook bypass — `--no-verify` anywhere in a `git commit` invocation |

The grep is case-insensitive (`-i`), so `DROP table` matches. Word boundaries (`\b`) prevent false positives on substrings. (verified: destructive_bash_gate.sh)

## Force-with-lease allowlist carve-out (added 2026-07-08, PR #72+#75)

The **only** conditional-allow in this entire hook. `git push --force`/`-f` (including combined short-flag clusters like `-uf`/`-fu`/`-ufd`, mirroring the `git clean` force detector above) is **always denied**, even alongside `--force-with-lease` on the same line. A *clean* `--force-with-lease` with no naked force present is allowed only if `.claude/destructive_allowlist` — resolved against the command's own repo root via `git rev-parse --show-toplevel`, gitignored/local-only — contains the exact whole-line keyword `git-push-force-with-lease` (matched with `grep -qxF`, never eval'd or regex-spliced, so a malformed file can only fail to match, never widen what's permitted). Missing/empty/garbage allowlist → denies (fail-closed).

Detection uses `force_cmd_flat`: a two-pass normalisation of `$cmd` that (1) splices backslash-newline *pairs* out entirely via `awk 'BEGIN{RS="\\\\\n"}...'` — matching what bash's own line-continuation does, fusing e.g. `--fo` + backslash-newline + `rce` into the real argv token `--force` — then (2) flattens any remaining bare newlines to spaces. Order matters: a naive `tr '\n' ' '` alone leaves the backslash behind (`--for\ ce`, two tokens) and misses an interior-flag-split bypass entirely (the PR #75 incident — see [[pr_72-75_2026-07-08_force-with-lease-allowlist]]).

`git_push_re` is option-tolerant (added PR #84): `git`, zero-to-20 bounded git global-option tokens (`-c KEY=VALUE`, `-C path`, `--no-pager`, any other short/long flag), then `push` — so `git -c user.name=x push --force` or `git --no-pager push --force` can't defeat the trigger by breaking `git`+`push` adjacency. Applied symmetrically to both the naked-force trigger and the fwl-exclusion check, so the allowlist path stays reachable when a global option is present. Known residual gap: a `-c`/`-C` value containing a quoted space is not matched — same class as this file's documented "quoted paths with spaces... remain uncaught" ceiling elsewhere.

See [[pr_72-75_2026-07-08_force-with-lease-allowlist]] and [[pr_84_2026-07-08_git-global-option-bypass]] for the full incident history and fixes.

## Command-substitution detection in workflow-script arguments (added 2026-07-08, PR #69)

`push.sh`/`merge.sh`/`post_review.sh`/`post_evals.sh` all take a free-text message/title/body argument. A bare backtick, `$(...)`, or process substitution `<(...)`/`>(...)` inside that argument executes as live shell substitution the instant the Bash tool_input line is interpreted — the gate denies this (same injection class as the `$ARGUMENTS` render-time bug, PR #97).

`subst_re='`|\$\(|<\(|>\('` — the `<(`/`>(` process-substitution forms were added in PR #69 after a confirmed bypass (`<(touch pwned)` executed with zero backticks or `$(` on the line). `cmd_flat` flattens embedded newlines to spaces before scoping, closing a second confirmed bypass where a script mention and its substitution landed on different physical lines (heredoc with an unquoted delimiter, or backslash line-continuation) — `sed`'s and `grep`'s `.` never cross a newline, so the un-flattened command hid the substitution from the scoping logic.

The prose exemption (a mention of the script name that isn't itself an invocation) is deliberately narrow after three prior narrower attempts each admitted a real bypass under adversarial review — it now fires only when the script pattern occurs exactly once, quoted, non-bare, with every substitution character on the line confined to that one quoted segment. More than one mention on a line is never treated as prose.

**Documented conservative tradeoff:** the newline-flattening in this block uses a plain `tr '\n' ' '` (not the backslash-pair-aware `awk` two-pass PR #75 later introduced for the force-push flattener) — so a *quoted-delimiter* heredoc (normally substitution-inert) is also denied if it contains substitution-looking characters. The code notes this gap explicitly as flagged-but-out-of-scope for PR #69. Accepted over-block, not a false-negative.

See [[pr_69_2026-07-08_substitution-bypass-audit]] for the full incident record.

### Why `git commit --no-verify` is destructive

`--no-verify` bypasses all git commit hooks, which in coderails means defeating the discipline loop that `test_gate.sh`, and any user-configured pre-commit scripts, provide. Doc basis: "don't bypass safety checks (e.g. --no-verify)". (verified: [[session_2026-05-31_prompting-doc-alignment]])

### Known false-positive: `--no-verify` in commit message text

The pattern `\bgit +commit +.*--no-verify` uses `.*` which matches the entire argument string including `-m "..."` contents. A command like:

```bash
git commit -m "reminder: don't use --no-verify"
```

would match and be denied. The substring `--no-verify` in the commit message body triggers the pattern. This is a known limitation accepted as an acceptable tradeoff given the low frequency of such commit messages in practice. (inferred: no git-parsing logic in the hook to distinguish flag vs. message content)

## Block condition

Any Bash command matching the regex above is denied. The hook emits a JSON response on stdout at exit 0 (verified: destructive_bash_gate.sh:18–25):

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "deny",
    "permissionDecisionReason": "Destructive pattern detected: <matched fragment>\nFull command: <cmd>\nThis command is permanently blocked. <route>"
  }
}
```

There is no approval path built into the hook. The reason field tells Claude to add a `settings.json` permission rule or choose a safer alternative — but that requires a deliberate human configuration change, not just re-asking.

## Deny messages name a concrete safe route (added 2026-07-17, PR #203; extended to all patterns same day, PR #216)

Before PR #203, `<route>` above was a single generic sentence appended
regardless of which pattern matched ("use a non-destructive alternative"
without naming one) — this caused a real stall in practice (an orchestrator
blocked on `git reset --hard` with no named way forward). `deny()` now looks
up a `route` string keyed on the matched pattern's own text (lowercased,
whitespace-collapsed for the lookup only — the reported `$pat` is unchanged).
PR #203 shipped 3 named routes; **PR #216, merged the same day, extended
every remaining blockable pattern to its own named route** — route arms went
4 → 14 (13 real routes + the generic fallback, kept for anything genuinely
unmapped). The full table, current as of PR #216:

| Blocked pattern | Named safe route |
|---|---|
| `git reset --hard` | `git branch backup/<desc> <ref>` first, then `git reset --keep <ref>` — `--keep` refuses rather than clobbers when it would discard uncommitted changes |
| `rm -rf` / recursive-force-remove | `unlink <file>` for a single file; move to a temp dir (`mkdir -p /tmp/trash && mv <target> /tmp/trash/`) for a directory |
| `git push --force`/`-f` | `--force-with-lease` — **with an explicit disclosure that fwl is itself denied by default**, naming the exact opt-in line (`git-push-force-with-lease` in `.claude/destructive_allowlist`) |
| `git clean -f`/`--force`/`-fd` etc. | `git clean -n` (dry-run/preview, lists targets without deleting) or `git clean -i` (interactive per-file/dir prompt) — **both already ALLOWed by this same gate**; only the force forms are blocked |
| `chmod -R 777` | `chmod -R u+rwX,go+rX <path>` — owner rw(+x on dirs/already-executable files), group/other read, without world-writable/world-executable |
| `git commit ... --no-verify` | Fix the failing pre-commit hook and re-run without `--no-verify`; if the hook itself is broken (not the change), fix the hook, don't skip it |
| `find ... -delete`/`--delete` | **No safe equivalent for the deletion itself** — preview the match set first with `-print` (or `-print0 \| xargs -0 ls`); to allow, add a settings.json permission rule |
| `truncate -s`/`--size` | **No safe equivalent** — truncate destroys content in place; back up first (`cp <file> <file>.bak`) or rotate instead of truncating; to allow, add a settings.json permission rule |
| `shred` | **No safe equivalent** — shred exists specifically to make content unrecoverable; if you only meant to delete (not securely wipe), move to a temp dir instead; to allow, add a settings.json permission rule |
| `DROP TABLE`/`DATABASE`/`SCHEMA` | **No safe equivalent** — take a backup/dump first if the data must be recoverable, confirm you're pointed at the intended database; to allow, add a settings.json permission rule |
| `TRUNCATE TABLE` | **No safe equivalent** — removes all rows, not equivalent in safety to a scoped `DELETE`; take a backup/dump first; to allow, add a settings.json permission rule |
| `dd if=` | **No safe equivalent** — writes raw bytes with no confirmation; double-check the `of=` target isn't a mounted disk; to allow, add a settings.json permission rule |
| `mkfs.*` | **No safe equivalent** — reformats a filesystem, destroying existing data; confirm the target device isn't mounted/in-use; to allow, add a settings.json permission rule |
| No specific mapping (nothing currently falls here — the fallback is retained for any future unmapped pattern) | Generic fallback text — not a false claim of a specific route |

**Six patterns (`find -delete`, `truncate`, `shred`, the three SQL DDL
patterns, and `dd`/`mkfs`) genuinely have no safe alternative.** Their routes
say so explicitly rather than inventing a plausible-sounding one — a
fabricated safe route was judged strictly worse than the honest generic
fallback it replaced, because an invented alternative sends someone down a
false path with false confidence. Only `git reset --hard`, `rm -rf`, `git
push --force`, `git clean`, `chmod -R 777`, and `git commit --no-verify` have
a genuine safer alternative to name.

This remains **message text only** — the paired test file asserts the same
DENY/ALLOW verdict set as before PR #203, unchanged; only
`permissionDecisionReason`'s content changed, both times. **bash 3.2 gotcha
caught by the hook's own test suite during PR #203's work:** the
lowercase-normalisation step was first tried with `${pat,,}` (bash 4+), which
**aborts the hook** on this machine's bash 3.2 (macOS default) — an abort
here is a fail-open (denies nothing). Fixed with
`tr '[:upper:]' '[:lower:]'`. See [[pr_201_202_203_routine-followups]] for
the full incident record, including two other durable lessons from the same
cluster (a handoff memory's claims need live-state re-verification; a frozen
eval command must be smoke-run once at freeze).

### Mention-safe `patternId` per case arm (PR #233)

Each named-route case arm above now also sets a hyphenated `pattern_id`
(e.g. `git-reset-hard`, `chmod-r-777`, `rm-rf`), surfaced in the hook's JSON
output as `hookSpecificOutput.patternId` (`null` for the unmapped generic
fallback arm). **Message-only** — the matcher regexes and DENY/ALLOW verdict
set are unchanged; only the output shape gained a field.

The id is deliberately hyphenated so it can never itself match the gate's
own whitespace-based regexes — `"chmod-r-777"` does not match
`chmod[[:space:]]+-R[[:space:]]+777`. This exists so a test or artifact can
reference a blocked pattern by id instead of quoting its literal text, which
previously risked this same gate denying the line that documents the block
(see [[pr_216_217_safe-routes-and-cost-miner-diagnostics]]'s "Lessons" #4,
the standing mention-vs-invocation tension). Every id is tested both
directions: the id string alone must ALLOW (mention-safety), the real
blocked literal must DENY and carry that same `patternId`.

The "Deliverable B" source-drift tripwire (below) gained a companion check,
`assert_every_route_arm_has_pattern_id`: it walks the `case "$pat_lc" in
... esac` block via `awk` and fails if any non-fallback `route=` assignment
isn't immediately followed by a `pattern_id=` assignment — so a future route
arm added without an id is caught here, not just a future *pattern* added
without a route. See [[pr_224_231_233_235_loop-tooling-hardening]] for the
full source record.

### Sync enforcement — a test, not a scheduled routine (PR #216)

Keeping this table from silently rotting as new patterns are added needed a
decision: a scheduled routine (drift reported days later) vs. a test in the
existing suite (fails the instant a routeless pattern ships). **Resolved: a
test** — the enforcement needs to be as fast as the change that causes the
gap. Two independent mechanisms, because a behavioural test alone only proves
today's pattern list has routes, not that a future one will:

1. **Per-pattern route assertions** via a new `assert_specific_route` test
   helper that drives the real gate and fails on the generic fallback text,
   AND requires pattern-specific needle strings unique to that route's own
   wording — 13 assertions, one per pattern.
2. **A source-drift tripwire** comparing the gate's actual blockable set (the
   `deny "..."` call sites plus the monolithic `pattern=` line) against two
   committed `EXPECTED_*` snapshots, failing with an actionable message the
   moment either changes.

Two discriminating-check defects were found in the first draft and fixed
before merge — both of the class "a check that cannot both pass and fail is
not a check": (a) the six honest ("no safe equivalent") routes originally
shared identical needles, so a copy-paste-swapped route body passed every
assertion (proven by swapping `dd`'s route text into `shred`'s case and
confirming it stayed green); (b) the `git clean` check grepped bare `-n`/`-i`,
two-char substrings matching incidentally inside unrelated words (a
fabricated route mentioning `"manual-n"` and `"4-i"` passed clean). Both
fixed with route-specific needles and full-string literals respectively. See
[[pr_216_217_safe-routes-and-cost-miner-diagnostics]] for the full incident
record.

**Why JSON deny and not exit 2:** For `PreToolUse`, both mechanisms block the tool call, but the JSON form carries `permissionDecisionReason`, delivering a useful explanation rather than a bare stderr string. JSON output is only processed at exit 0, so the hook exits 0 after emitting JSON. See [[hook-exit-codes]] for the full rationale.

## Branch-aware in-Bash source-edit gate (added PR #59)

A best-effort gate blocks in-Bash edits to source files (`.py`, `.ts`, `.tsx`, `.js`, `.jsx`, `.go`) and plugin source (`skills/*/SKILL.md`, `commands/*.md`) on `main`/`master`. This closes the gap that `no_edit_on_main.sh` misses — `no_edit_on_main` only intercepts `Write`/`Edit`/`MultiEdit` tool calls, not bash shell commands that write to files.

Blocked forms: `sed -i`, `perl -i`, shell redirects (`>`/`>>`), `tee`, `cp <src> FILE`, `mv <src> FILE`, `dd of=FILE`.

Branch detection uses target-repo resolution (the file's own git repo's branch), mirroring [[no_edit_on_main]]. Falls back to cwd-branch when the target path is not resolvable.

**Enforcement ceiling:** this gate is **best-effort**. Variable filenames, quoted paths with spaces, here-docs, process substitution, and `python -c open(...)` writes remain uncaught. This is documented as a deliberate ceiling, not a bug. See [[pr_57-62_subagent-enforcement-gate-hardening]].

**`plugin_src` path-boundary anchor (added 2026-07-08, PR #92).** The matcher for `skills/*/SKILL.md`/`commands/*.md` targets was originally unanchored on the left, so it matched the substring anywhere on the line — `tee xcommands/prep.md` false-positive DENYed even though `xcommands/` merely contains the substring `commands/`, not a real path segment. Fixed by requiring a path/token boundary immediately before the match (start-of-string, whitespace, quote, or a preceding `/`): `plugin_src='(^|[[:space:]/'"'"'"])(skills/[^/]+/SKILL\.md|commands/[^/]+\.md)([ '"'"'"]|$)'`. A genuinely nested real path like `vendor/skills/x/SKILL.md` still matches (the `/` before `skills/` is a real separator) — no false-negative introduced. Mirrors `src_ext`'s existing right-anchor. See [[pr_92_2026-07-08_reference-drift-and-lookalike-fp]].

## No logging

This hook does not append to `$CLAUDE_DISCIPLINE_LOG`. (verified: destructive_bash_gate.sh — no reference to `$CLAUDE_DISCIPLINE_LOG` or `discipline.log`)

## Environment variables

None for the original blocklist. The in-Bash source-edit gate reads `.cwd` from the hook payload and falls back to `$PWD`. (verified: destructive_bash_gate.sh)

## 2026-07-08 adversarial-hardening arc — cross-cutting theme

A same-day, five-PR arc (#69, #72+#75, #84, #92) that hardened this single line-oriented ERE gate against three independent bypasses (process substitution, multi-line substitution, git-global-option adjacency-break) and fixed one false-positive (`plugin_src` lookalike). The pattern across all five: **each narrowing of a regex risks reopening an adjacent hole** — the PR #72 tab-separator regression (fixed in #75) is the concrete instance, where the fix for one gap (adding the allowlist carve-out) shipped with a live boundary bug in an unrelated part of the same regex. The corrective discipline that emerged: every fix to this gate needs (a) a fresh adversarial pass, not just a fix for the reported shape, and (b) a **positive-control test** — asserting the allowlisted path actually ALLOWs on a clean payload, not just that denied shapes stay denied — because a fail-closed default can silently mask a broken allow-path the same way it correctly masks a bypass. See each source page for the specific incident: [[pr_69_2026-07-08_substitution-bypass-audit]], [[pr_72-75_2026-07-08_force-with-lease-allowlist]], [[pr_84_2026-07-08_git-global-option-bypass]], [[pr_92_2026-07-08_reference-drift-and-lookalike-fp]].

The gate's documented enforcement ceiling (`AGENTS.md`) still applies unchanged after this arc: quoted paths with embedded spaces, variable filenames, and `python -c` writes remain best-effort-uncaught.

## Related

## Stdin read convention (PR #76)

This hook reads its payload via `IFS= read -r -d '' -t 5 input || true`. See [[pr_76_harden-hook-stdin-read]] for the full convention.

- [[hook-exit-codes]] — why PreToolUse hooks use `permissionDecision: "deny"` + exit 0 rather than exit 2
- [[enforcement-model]] — hooks vs. commands; this is the clearest example of mechanical enforcement
- [[discipline-loop]] — the full set of coderails hooks
- [[pr_69_2026-07-08_substitution-bypass-audit]] — process-substitution + multi-line substitution bypass fixes
- [[pr_72-75_2026-07-08_force-with-lease-allowlist]] — force-with-lease allowlist carve-out + tab-separator incident
- [[pr_84_2026-07-08_git-global-option-bypass]] — option-tolerant git_push_re
- [[pr_92_2026-07-08_reference-drift-and-lookalike-fp]] — REFERENCE.md drift fix + plugin_src anchor fix
- [[pr_201_202_203_routine-followups]] — PR #203, first 3 named safe routes in deny messages + the bash 3.2 `${pat,,}` fail-open incident
- [[pr_216_217_safe-routes-and-cost-miner-diagnostics]] — PR #216, extends named routes to all ~13 blockable patterns + the source-drift tripwire + two discriminating-check fixes
- [[pr_224_231_233_235_loop-tooling-hardening]] — PR #233, the mention-safe `patternId` field documented above + the tripwire's companion id-presence check
