---
title: "/coderails:test-gate-setup"
type: command
created: 2026-06-25
last_updated: 2026-06-25
sources: [commands/test-gate-setup.md]
tags: [command, setup, testing, test-gate, hooks]
---

# /coderails:test-gate-setup

Detects the project's test runner and writes `.claude/test_command` — the opt-in file that arms the `test_gate.sh` hook so every `git commit` runs tests first.

## Invocation

```
/coderails:test-gate-setup
```

No arguments.

## What it does

1. **Checks for existing config** — if `.claude/test_command` already exists, reads and reports it, then asks whether the user wants to change it.
2. **Auto-detects the test runner** by checking for known project files in this order (verified from source):
   - `package.json` with a `test` script → `npm test`
   - `Cargo.toml` → `cargo test`
   - `pyproject.toml` / `setup.py` / `setup.cfg` → `pytest -x`
   - `go.mod` → `go test ./...`
   - `Makefile` with a `test` target → `make test`
   - `mix.exs` → `mix test`
   - `Gemfile` → `bundle exec rspec`
3. **Proposes the detected command** to the user. If multiple indicators are detected, asks which to use. If none are detected, asks the user to supply the command manually.
4. **Creates `.claude/test_command`** — creates the `.claude/` directory if needed, then writes the single-line test command.
5. **Runs the test command once** to verify it works. Reports the result.
6. **Confirms** with: "Test gate is active. Every `git commit` in this project will run `<command>` first. Remove `.claude/test_command` to disable."

## File written

`.claude/test_command` — a single line containing the test command string (e.g. `npm test`, `pytest -x`).

This file is the activation switch for [[test_gate]]. The hook reads it on every `git commit` `PreToolUse` event; if the file is absent, the hook no-ops. (verified from hook design: "opt-in via `.claude/test_command`").

## Scripts invoked

None directly. The command runs the detected test command once via Bash to verify it works, but does not shell out to a dedicated script.

## Preconditions

- Must be run in the project directory that contains the test runner indicators.
- The test runner itself must be installed and working (e.g. `npm` on PATH, `pytest` in the virtualenv).
- `.claude/` directory need not pre-exist — the command creates it.

## Opt-in / opt-out model

The test gate is **off by default**. Running this command turns it on for the current project. To disable: `rm .claude/test_command`. The hook (`test_gate.sh`) checks for the file's presence on every `git commit`; no file means no gate. This is an opt-in per-project mechanism, not a global setting.

## Relationship to the hook

`test-gate-setup` is the setup command; [[test_gate]] is the enforcement hook. They are separate concerns:

- This command: interactive, advisory, runs once per project setup.
- `test_gate.sh`: mechanical, automatic, runs on every `git commit` if the file exists.

See [[enforcement-model]] for the general distinction between commands (advisory) and hooks (mechanical).

## See also

- [[test_gate]] — the `PreToolUse` hook that reads `.claude/test_command` and blocks commits when tests fail
- [[enforcement-model]] — why the gate is a hook (mechanical enforcement) not a command
