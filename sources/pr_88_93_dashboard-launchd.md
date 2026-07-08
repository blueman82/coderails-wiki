---
title: "PR #88 + #93 — dashboard launchd agent + uninstall bootout race"
type: source
created: 2026-07-08
last_updated: 2026-07-08
sources: []
tags: [source, dashboard, launchd, reboot-persistence, boot-persistence]
---

# PR #88 + #93 — dashboard launchd agent + uninstall bootout race

Ingested as one cluster, theme: **dashboard reboot survival via launchd**. Both merged to `origin/main` 2026-07-08 (verified via `gh pr view --json mergedAt,mergeCommit`).

## PR metadata

| Field | Value |
|---|---|
| PR number | #88 |
| Title | feat/dashboard launchd agent |
| Merge SHA | `ec46132dca4d3f73b266bc0837b4fd9d75d39ad2` |
| Merged | 2026-07-08T08:50:58Z |
| PR number | #93 |
| Title | dashboard uninstall bootout race |
| Merge SHA | `f630a9836611157688cc40259178bf6b8cf2ab71` |
| Merged | 2026-07-08T09:18:12Z |

## Summary

**PR #88** gives the observability dashboard the same reboot-survival mechanism [[routines]] already has: a launchd LaunchAgent instead of a manually-started, pidfile-tracked process. New files: `launchd/com.coderails.dashboard.plist` (`RunAtLoad`+`KeepAlive`+`ThrottleInterval 60`), a thin exec wrapper `skills/dashboard/runner/bin/dashboard-server.sh`, `launchd/install-dashboard-agent.sh` / `uninstall-dashboard-agent.sh`, a static guard suite `hooks/scripts/tests/dashboard_agent.test.sh`, and a new `SKILL.md` "Surviving reboots (launchd)" section.

**PR #93** fixes a race in the uninstaller found immediately after #88 shipped: `launchctl bootout` on a running `KeepAlive` job is asynchronous (~2s observed), so a single immediate `launchctl print` re-check spuriously reported "still loaded" and bailed before removing the `~/Library/LaunchAgents/` copy. The uninstaller now polls up to 10×1s before declaring failure.

## Files changed

- `launchd/com.coderails.dashboard.plist` — new
- `launchd/install-dashboard-agent.sh` — new
- `launchd/uninstall-dashboard-agent.sh` — new (#88), race-fixed (#93)
- `skills/dashboard/runner/bin/dashboard-server.sh` — new
- `skills/dashboard/SKILL.md` — new launchd section
- `hooks/scripts/tests/dashboard_agent.test.sh` — new (#88), extended (#93)
- `hooks/scripts/tests/routine_runner_bin_targets.test.sh` — one exclusion line added

## Architecture: why a thin exec wrapper, not the existing scripts

Three shapes were on the table; the loop's resolved forks:

- **(a) Thin exec wrapper** (chosen) — `dashboard-server.sh` execs `npm run start` in the foreground, sibling to `launchd/install-routines.sh`'s `bin/*.sh` pattern.
- **(b) plist invokes `start-dashboard.sh` directly** — rejected: that script's `nohup`-backgrounding-plus-pidfile model fights `KeepAlive`, which needs the plist's own `ProgramArguments` process to be the one that lives or dies.
- **(c) a `--foreground` flag added to `start-dashboard.sh`** — rejected: mixes two ownership models (self-managed pidfile vs. launchd-managed) in one script; repo precedent (`install-routines.sh`'s bin wrappers) is separate thin wrappers per invocation model, not a flag-branched shared script.

Flip condition, recorded for a future reader: if the two scripts' copied `build-if-stale` logic (~13 lines) drifts, that's the trigger to unify them — until then this is accepted duplication (YAGNI), not an oversight.

The wrapper deliberately diverges from `start-dashboard.sh`'s copy in one way: staleness detection also covers dependency/config files (`package.json`, `package-lock.json`, `next.config.mjs`), not just `src/`, and a missing `src/` dir is treated as a fail-safe rebuild trigger — a daemon has no operator watching it to notice a stale build the way `start-dashboard.sh`'s interactive caller would. `dashboard_agent.test.sh` asserts this divergence directly (`find package.json package-lock.json next.config.mjs -newer .next`).

The wrapper execs `npm` via an exported `PATH` (not a `$SCRIPT_DIR/../`-relative node target) because launchd's environment carries no `PATH` at all (same finding [[routines]] already documents for the routine-sweeper plists, re-verified here) — `routine_runner_bin_targets.test.sh`'s node-target guard explicitly excludes this wrapper on that basis, with `dashboard_agent.test.sh` covering the equivalent risk via its own TARGET/staleness/npm-ci checks.

## Boot persistence — the load-bearing mechanism (same pattern as routines, PR #85)

`install-dashboard-agent.sh` copies the plist into `~/Library/LaunchAgents/com.coderails.dashboard.plist` (mode 0644) and bootstraps `launchctl bootstrap gui/$(id -u)` **from that copy**, not from the repo path. This mirrors the exact fix `install-routines.sh` needed after the 2026-07-08 reboot loss (`routines.md`'s "Boot persistence" section: 03:00 run fine, reboot at 07:34, every `com.coderails` job silently gone) — `launchctl bootstrap` from an arbitrary path survives only until logout/reboot; launchd only auto-loads plists that live in `~/Library/LaunchAgents/`. The dashboard installer adopts the already-discovered fix rather than re-discovering the bug.

Other install-time details: a `lsof -nP -iTCP:4173 -sTCP:LISTEN` pre-flight refuses to bootstrap if the port is already held (avoiding an EADDRINUSE crash-loop against a manually-started server); the log directory `~/.claude/coderails-dashboard/` is created at `mkdir -p -m 0700` and then unconditionally `chmod 700`'d (the chmod covers a dir left at a looser mode by an earlier manual run — `mkdir -p` only sets mode on creation).

## The stop-command gotcha

Once the agent is installed, **`stop-dashboard.sh` cannot stop it** — the agent-owned server process has no pidfile for that script to find. The documented stop command is:

```
launchctl bootout gui/$(id -u)/com.coderails.dashboard
```

`SKILL.md` also documents the reverse ordering requirement: stop any manually-started server **before** running `install-dashboard-agent.sh`, since the installer's port pre-flight refuses to bootstrap over a live manual server (clean failure), but installing the agent first and then hand-running `start-dashboard.sh` also fails cleanly (its own lsof guard refuses). The real risk is neither of those — it's installing the agent while a manual server already holds the port, which produces a `ThrottleInterval`-rate-limited (60s) respawn crash-loop until one side is stopped.

## PR #93 — the uninstall bootout race

`launchctl bootout` returns before a running `KeepAlive` job has actually unloaded — observed live 2026-07-08, ~2 seconds between `bootout` returning and `launchctl print` reporting the job gone. The pre-fix uninstaller checked once, immediately, and on a still-loaded false positive it both errored out *and* still ran `rm -f "$DEST"` unconditionally afterward in the original code path shape — PR #93's diff reorders so removal happens only after the loaded-check passes.

Fixed shape: poll `launchctl print` inside a `for _ in $(seq 1 10); do ... sleep 1; done` loop (10×1s ceiling) before declaring failure; on failure, the bootout's own stderr is captured and surfaced (`bootout_err="$(launchctl bootout ... 2>&1 >/dev/null || true)"`), and the message states explicitly that the `~/Library/LaunchAgents/` copy is deliberately left in place (not safe to remove while the job might still be loaded).

The mutation-tested guard added to `dashboard_agent.test.sh` doesn't just check a loop keyword exists — it asserts via `awk` that `launchctl print` appears *inside* the loop body between the `bootout` call and the failure-check line, specifically to catch a loop with an empty/no-op body or an unrelated earlier loop satisfying a looser textual check while the underlying race bug remained. A second guard asserts the `rm -f "$DEST"` line appears strictly after the still-loaded failure-check line, guarding against a future refactor reintroducing the remove-before-confirming-unloaded ordering bug.

## Live verification

Both PRs' authors report the agent installed and exercised on this machine: `KeepAlive` respawn proven (dashboard process killed, respawned within 42s), and a full uninstall→reinstall cycle proven post-#93-fix. `(inferred from PR body/commit messages — not independently re-run for this ingest)`.

## Wiki pages updated

- [[dashboard]] — new "Surviving reboots (launchd)" section
- [[routines]] — cross-reference added; the dashboard agent explicitly reuses the boot-persistence fix routines discovered first
- `index.md` — new source entry, hooks/skills table note if applicable

## Caveats / gotchas

- **Not portable** — same caveat as the routines plists: `com.coderails.dashboard.plist`'s `ProgramArguments` and log paths are this machine's absolute paths, baked in at authoring time, not derived from `install-dashboard-agent.sh`'s own `SCRIPT_DIR` or `$HOME`. Copying into `LaunchAgents` does not change that.
- **The `build-if-stale` duplication between `dashboard-server.sh` and `start-dashboard.sh` is deliberate (YAGNI), not an oversight** — recorded above as the loop's own flip condition, not raised here as an unfixed finding.
