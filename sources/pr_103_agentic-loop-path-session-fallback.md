---
title: "PR 103 — agentic-loop path session-id fallback probe (split-slug fix)"
type: source
created: 2026-07-08
last_updated: 2026-07-08
sources: []
tags: [agentic-loop, progress-json, session-id, git-common-dir, loop-state, migration-gap]
---

# PR 103 — agentic-loop path session-id fallback probe (split-slug fix)

<!-- Ingested by /wiki-ingest after merge. This is an immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR number | #103 |
| Branch | `slug-fix` |
| Merged | 2026-07-08 |
| Merge SHA | `bf02e8a` |
| Head SHA | `f4ce997` |
| JIRA ticket | — |

## Summary

Closes the plugin-version re-keying gap that [[agentic-loop-path-keying]]
documented at "Known caveats" bullet 4 as *filed for the owner, not fixed*.
Implements the first candidate fix named there: a **session-id fallback probe**
in `hooks/scripts/lib/agentic_loop_path.sh`.

## Incident (live-reproduced 2026-07-08)

Session `46d6c1b5` registered its loop's `progress.json` under the **legacy
raw-cwd slug**
(`~/.claude/agentic-loop/-Users-harrison-Github-coderails-.claude-worktrees-routines/<session>/progress.json`)
because the checkout it ran from predated PR #24. The live Stop hooks ran from
the installed plugin cache (1.1.0 = current main), whose helper keys the slug by
`--git-common-dir` (`-Users-harrison-Github-coderails-.git`). The readers
resolved the canonical git-common-dir path, found nothing, and
`unregistered_loop_guard` nudged a COMPLETED, registered loop on every Stop. The
same split earlier cost the loop its `loop_stop_counts` at Phase 13.

## Mechanism (general lesson)

The "sole path authority" is only sole **per copy**. Writer (orchestrator
intake) and readers (Stop hooks) can run *different helper versions*, and a
session's cwd can drift mid-loop. The same session's state can be written under
one slug and read back under another. Any fix requiring version lockstep or a
stable cwd is insufficient — hence keying the recovery on `session_id`, which is
unique per session and stable for the life of the conversation.

## What changed

`agentic_loop_path.sh` resolution is now (slug computation unchanged):

1. Compute the canonical path (git-common-dir slug; cwd-slug fallback outside a
   repo) exactly as before.
2. If `progress.json` **exists** at the canonical path → print it (unchanged).
3. Else **probe** `"$base"/*/"$session_id"/progress.json`. Existing matches are
   deduped by the **physical identity** of their containing dir (`pwd -P`), so
   the orchestrator's workaround symlinks — which alias one real file under
   several slugs — collapse to one. If distinct real files somehow coexist, the
   pick is deterministic (lexicographically smallest). Print the match.
4. Else (no state anywhere) → print canonical, so a fresh loop registers there.

The pre-existing `session_id` sanitisation (`tr '/' '_'`, `sed 's/\.\.//g'`) now
runs **load-bearingly before** the glob — a comment says so — so the
`<session_id>` glob segment cannot expand into a sibling/parent directory.

Constraints preserved: dependency-free (no `source`), pure (prints a path,
creates nothing), bash-3.2-safe (verified under `/bin/bash` 3.2.57 — the
`[ "$a" \< "$b" ]` POSIX string comparison and the empty-glob literal-pattern +
`[ -e ]` guard both behave), no `set -e` surprises on empty globs.

Header comment's path-resolution description and "Accepted limitation" note
reworded honestly: the probe now **heals** the common mid-loop repo-ness-change
split; the one residual gap (slug changes before any file is written → fresh
registration at the new canonical path) is documented. `docs/REFERENCE.md` (two
entries: the file table and the loop-state paths table) updated to match.

## Verification

- **TDD**: 5 new checks added to `hooks/scripts/tests/agentic_loop_path.test.sh`
  (17 → 22): canonical-exists-wins even with a legacy file present; the incident
  (legacy-slug state found when canonical has none); no-state → canonical
  printed; symlinked-duplicate dedupe (one path, references the real file);
  generated fallback session ids (`unknown-…`) resolve to canonical with no
  cross-probe collision. Confirmed RED (incident + symlink cases) before
  implementing, then GREEN.
- **Full suite**: `run_all.sh` green (37/37) under bash 5 and bash 3.2 — no
  other suite that exercises `als_resolve_path` regressed.
- **PR-scope evals**: GO, tier 1, 6/6 P0 asserted against a **fresh clone** of
  the branch head with passing negative controls (the-incident, canonical-wins,
  fresh-registration, symlink-dedupe, non-git-regression+purity, full-suite).
- **Review**: independent code-reviewer pass, no material findings.

## Notes

- The live workaround symlinks under `~/.claude/agentic-loop/` were left in
  place — load-bearing for session `46d6c1b5` until its plugin cache refreshes.
- Plugin version deliberately **not** bumped; plugin-cache refresh is the
  orchestrator's post-merge concern.

## See also

- [[agentic-loop-path-keying]] — the design page this fix updates (the caveat it
  closes + the dated "Update — session-id fallback probe" subsection)
- [[pr_87_agentic-loop-path-session-keying]] — added `session_id` to the key;
  this PR leverages that same key for recovery
- [[pr_23-24_hook-lib-observability-and-repo-keyed-loop-state]] — introduced the
  git-common-dir slug whose version skew caused the split
- [[loop_state_guard]] — the reader that was blinded by the split
