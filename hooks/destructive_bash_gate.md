---
title: "Hook: destructive_bash_gate"
type: hook
created: 2026-05-31
last_updated: 2026-06-29
sources:
  - hooks/scripts/destructive_bash_gate.sh
  - sources/session_2026-05-31_prompting-doc-alignment.md
  - sources/pr_57-62_subagent-enforcement-gate-hardening.md
  - sources/pr_76_harden-hook-stdin-read.md
tags: [hook, pretooluse-hook, enforcement, destructive-bash, block]
---

# Hook: destructive_bash_gate

A `PreToolUse (Bash)` lifecycle hook that permanently blocks a fixed set of destructive shell commands before they run, with no approval path. Extended in PR #59 to cover additional force-clean, truncate, shred commands plus a best-effort branch-aware gate on in-Bash source file edits.

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

### Original permanent blocklist (main regex)

```
\brm +(-[rRfF]+|--recursive|--force)
|\bgit +push +.*(--force|-f\b|--force-with-lease)
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
| `git push --force`, `git push -f`, `git push --force-with-lease` | Force-push overwriting remote history |
| `git reset --hard` | Hard history rewind, discarding local changes |
| `DROP TABLE/DATABASE/SCHEMA` | Destructive SQL DDL |
| `TRUNCATE TABLE` | Destructive SQL DML |
| `dd if=` | Raw disk write |
| `mkfs.*` | Filesystem formatting |
| `chmod -R 777` | World-write permission on a tree |
| `git commit --no-verify` | Hook bypass — `--no-verify` anywhere in a `git commit` invocation |

The grep is case-insensitive (`-i`), so `DROP table` matches. Word boundaries (`\b`) prevent false positives on substrings. (verified: destructive_bash_gate.sh)

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
    "permissionDecisionReason": "Destructive pattern detected: <matched fragment>\nFull command: <cmd>\nThis command is permanently blocked. To allow it, add a Bash permission rule to settings.json or use a non-destructive alternative."
  }
}
```

There is no approval path built into the hook. The reason field tells Claude to add a `settings.json` permission rule or choose a safer alternative — but that requires a deliberate human configuration change, not just re-asking.

**Why JSON deny and not exit 2:** For `PreToolUse`, both mechanisms block the tool call, but the JSON form carries `permissionDecisionReason`, delivering a useful explanation rather than a bare stderr string. JSON output is only processed at exit 0, so the hook exits 0 after emitting JSON. See [[hook-exit-codes]] for the full rationale.

## Branch-aware in-Bash source-edit gate (added PR #59)

A best-effort gate blocks in-Bash edits to source files (`.py`, `.ts`, `.tsx`, `.js`, `.jsx`, `.go`) and plugin source (`skills/*/SKILL.md`, `commands/*.md`) on `main`/`master`. This closes the gap that `no_edit_on_main.sh` misses — `no_edit_on_main` only intercepts `Write`/`Edit`/`MultiEdit` tool calls, not bash shell commands that write to files.

Blocked forms: `sed -i`, `perl -i`, shell redirects (`>`/`>>`), `tee`, `cp <src> FILE`, `mv <src> FILE`, `dd of=FILE`.

Branch detection uses target-repo resolution (the file's own git repo's branch), mirroring [[no_edit_on_main]]. Falls back to cwd-branch when the target path is not resolvable.

**Enforcement ceiling:** this gate is **best-effort**. Variable filenames, quoted paths with spaces, here-docs, process substitution, and `python -c open(...)` writes remain uncaught. This is documented as a deliberate ceiling, not a bug. See [[pr_57-62_subagent-enforcement-gate-hardening]].

## No logging

This hook does not append to `$CLAUDE_DISCIPLINE_LOG`. (verified: destructive_bash_gate.sh — no reference to `$CLAUDE_DISCIPLINE_LOG` or `discipline.log`)

## Environment variables

None for the original blocklist. The in-Bash source-edit gate reads `.cwd` from the hook payload and falls back to `$PWD`. (verified: destructive_bash_gate.sh)

## Related

## Stdin read convention (PR #76)

This hook reads its payload via `IFS= read -r -d '' -t 5 input || true`. See [[pr_76_harden-hook-stdin-read]] for the full convention.

- [[hook-exit-codes]] — why PreToolUse hooks use `permissionDecision: "deny"` + exit 0 rather than exit 2
- [[enforcement-model]] — hooks vs. commands; this is the clearest example of mechanical enforcement
- [[discipline-loop]] — the full set of coderails hooks
