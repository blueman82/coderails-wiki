---
title: "PR #289 — install.sh warns when other tier-gate daemons share its install root"
type: source
origin: "coderails PR #289 (merged 0f2605b, 2026-07-24; head 993bb08)"
created: 2026-07-24
last_updated: 2026-07-24
sources: [scripts/tier-gate/install.sh, hooks/scripts/tests/tier_gate_install.test.sh]
tags: [tier-review, install, shared-state, cross-repo, multi-instance, launchd, defense-in-depth]
---

# PR #289 — install.sh warns when other tier-gate daemons share its install root

<!-- Ingested by /wiki-ingest after merge. This is an immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR number | #289 |
| Merged | 2026-07-24 (00:19:37Z) |
| Merge commit | `0f2605b3a0c24a0445415d1d2838424f2cded45d` |
| Head commit | `993bb08fcbc1073a0b492c604ab58468ab1f057e` |
| JIRA ticket | — |

## The architecture fact this PR surfaces

`scripts/tier-gate/install.sh` promotes three things to a root-owned install
root: the runner script, the judge-prompt, and credentials. Two env vars
govern where:

```bash
TGI_INSTALL_ROOT="${TGI_INSTALL_ROOT:-/etc/coderails-tier-gate}"
TGI_PLIST_DEST="${TGI_PLIST_DEST:-/Library/LaunchDaemons/com.coderails.tier-gate.plist}"
```

Only `TGI_PLIST_DEST` varies per repo instance (e.g.
`com.coderails.tier-gate.assistant-agent.plist` for
`TIER_GATE_REPO=blueman82/assistant-agent`). `TGI_INSTALL_ROOT` does **not** —
every tier-gate daemon on the machine, regardless of which repo it judges,
shares the same runner/judge-prompt/credentials directory by default. So
running `install.sh` for any one repo overwrites the runner, judge-prompt,
and credentials used by **every other installed tier-gate daemon** on that
machine, not just the one being (re)installed. This is intentional (one
codebase, one credential set) and safe at runtime — no shared mutable state
at execution time, since the only per-run temp file is a unique `mktemp` in
`tier-gate-runner.sh` — but it means a fix landed for one repo's daemon goes
live for all of them on the next reinstall, silently, with no per-repo
opt-out. (verified — `install.sh` diff, `gh pr view 289 --json files,body`)

## What triggered this

Discovered while investigating whether a reinstall for `blueman82/coderails`
had wiped the separate `blueman82/assistant-agent` tier-gate daemon. It
hadn't — separate plist, separate `Label`, separate log, confirmed untouched
via `launchctl`. But the investigation exposed that nothing in `install.sh`
itself would have *warned* if it had, short of reading the code comment
first.

## The fix

New `tgi_other_instance_labels <plist_dest> <plist_glob>`: globs
`/Library/LaunchDaemons/com.coderails.tier-gate*.plist`, skips the plist this
run is about to write (`-ef` same-file check against `plist_dest`), and
echoes the `Label` (via `PlistBuddy -c "Print :Label"`) of every other
matching plist found. Echoes nothing on no matches or an unreadable Label —
treated as "no other instance found," not a preflight failure, since this is
a WARN, not a gate.

The confirmation prompt in the entry-point block now calls this before the
existing diff-before-promote check, and folds a non-empty result into the
same `diff_clean=0` path that already forces an explicit `[y/N]` confirm —
so a shared-root install surfaces at the moment of risk (the prompt itself),
not only in a code comment someone has to read first. (verified — `install.sh`
diff, entry-point block)

## Tests

3 new tests in `hooks/scripts/tests/tier_gate_install.test.sh` (69/69 passing
per the PR body), covering: excludes `plist_dest` and echoes only the other
Label; only `plist_dest` matches the glob → no output; no matching plists at
all → no output. Skips cleanly if `PlistBuddy` is unavailable on the host.

## Hit its own self-edit denylist — force-merged by the owner

This PR touches `scripts/tier-gate/`, on the daemon's own
[[tier-gate-path-denylist-dashboard_2026-07-21|self-edit denylist]]. The
daemon posted `verdict=self_edit tier=1` on the head SHA
(`993bb08`) and refused to judge — correct behaviour by design, not a
defect: a change to the judge's own install path can't be self-approved by
the mechanism it changes. Force-merged by the owner past the block, the same
documented human-disposition escape hatch
[[pr_274_tier_gate_observability_fixes|PR #274]] and
[[pr_256_runner-transcript-persistence|PR #256]] used for the same denylist
entry. (verified — `gh api .../commits/993bb08.../status`: single status,
`state=failure`, `description="verdict=self_edit tier=1 host=Mac"`; merge
commit `0f2605b3` present on `origin/main`, merged by `blueman82`)

## How this extends the install-root narrative

[[pr_274_tier_gate_observability_fixes|PR #274]] already established that
the tier-gate daemon runs only from an **installed root copy**, promoted by
`install.sh`, never from `origin/main` directly — and that a stale local
checkout can cause `install.sh` to promote outdated code while still
printing "INSTALL COMPLETE." This PR adds a second, orthogonal axis to that
same install-root risk surface: even a **fresh, correct** install can have a
wider blast radius than the operator expects, because the install root is
shared **across repos**, not just across time. Both are instances of the
same underlying fact — a source-tree/PR-merged property is not automatically
a running-daemon property — but #274's instance is temporal (this repo's
own daemon running stale code) while #289's is spatial (one repo's install
silently touching every other repo's daemon).

## Wiki pages updated

- `index.md` — new Sources entry
