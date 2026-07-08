---
title: "PR 95 — slash-command loop detection (als_count_invocations undercounted slash-started loops)"
type: source
created: 2026-07-08
last_updated: 2026-07-08
sources: []
tags: [source, loop-state-common, als_count_invocations, loop_stop_counts, slash-command, agentic-loop, hook-detection]
---

# PR #95 — slash-command loop detection

<!-- Ingested by /wiki-ingest after merge. This is an immutable record of what changed. -->

## PR metadata

| Field | Value |
|---|---|
| PR number | #95 |
| Branch | `fix/loop-invocation-count-slash-command` |
| Merged | 2026-07-08T10:52:31Z |
| Merge SHA | `bcd572bd19dd8151c060158066b1b5d33dd800f5` |
| Base | main |
| JIRA ticket | — |

## Summary

`als_count_invocations` (`hooks/scripts/lib/loop_state_common.sh`) only detected the **programmatic** invocation form of a loop — an assistant `Skill` tool_use with `input.skill` matching `agentic-loop`. A loop started via the **slash-command** form, `/coderails:agentic-loop`, is recorded in the transcript as a `user`-role message whose `.message.content` is a plain **string** carrying `<command-name>/coderails:agentic-loop</command-name>` — never an assistant tool_use. The count was therefore 0 for every slash-started loop, `als_gate_require_active_loop` treated the session as "not a loop," and every downstream consumer (`als_load_progress`, `gate_loop_stop_declared`, `bump_loop_stop_count`) never ran. The fix extends the jq filter to also scan user `<command-name>` messages, anchored on the same `(^|:)agentic-loop$` name test after stripping the leading `/`, benefiting all four consumers of the shared function (`loop_stall_guard`, `loop_state_guard`, `voice_announce`, `unregistered_loop_guard`).

**Root cause chain (verified against the reproduction session `3f897875`):** all 54 log lines for that session read `invocations=0 active=0 blocked=0` — never `declared=1`. This was NOT a keying bug (ALS_PATH was fine) and NOT a tail-window blind spot (both were earlier, now-disproven hypotheses) — it was a wholesale detection miss: the counting function structurally could not see the slash-command transcript shape at all.

## Files changed

- `hooks/scripts/lib/loop_state_common.sh` — `als_count_invocations()` jq filter gains a second form (`# Form 2: user slash-command message`) alongside the pre-existing `# Form 1: assistant Skill tool_use`.
- `hooks/scripts/tests/loop_stall_guard.test.sh` — new coverage (see Tests below).

## The fix (verbatim jq shape, verified against merge SHA)

```jq
def loop_name: test("(^|:)agentic-loop$");
[ .[]?
  # Form 1: assistant Skill tool_use.
  | ( select(.type == "assistant")
      | .message.content[]?
      | select(.type == "tool_use" and .name == "Skill")
      | (.input.skill // "")
      | select(loop_name) ),
    # Form 2: user slash-command message (content is a string carrying
    # <command-name>). Scan EVERY command-name tag ("g" flag), not just
    # the first. Strip leading "/" and trim whitespace before the
    # anchored loop_name test.
    ( select(.type == "user")
      | .message.content
      | select(type == "string")
      | ( [ match("<command-name>/?([^<\\n]+)</command-name>"; "g")
            | .captures[0].string ][]? )
      | gsub("^\\s+|\\s+$"; "")
      | select(loop_name) )
]
| length
```

## Review-driven hardening (same PR)

1. **Scan every `<command-name>` tag, not just the first** — `match(...; "g")` rather than a single match. A loop tag appearing after a non-loop tag in the same message would otherwise undercount to 0.
2. **Trim whitespace on the captured name before the anchored `$`-test** — a padded tag (`.../agentic-loop  `) would otherwise fail the anchor and silently re-hide the exact bug this PR fixes.
3. **Deliberately NO cross-form dedup** (documented inline in the code): a loop both slash-started *and* programmatically Skill-invoked in the same session counts 2 — matching the pre-existing behavior for two separate Skill tool_uses. This is a conscious non-fix: the primary re-arm signal for a loop is Phase -2's stub-first `progress.json` overwrite, not the invocation count, so double-counting across forms was judged harmless.

## Tests added

- End-to-end (`loop_stall_guard.test.sh`): scoped slash form counts (`/coderails:agentic-loop`), bare slash form counts (`/agentic-loop`), a non-loop slash command does NOT count, and a multi-tag message where the loop tag is not first still counts.
- Direct unit tests for `als_count_invocations`: an array-content (non-string) `tool_result` message is not miscounted, a trailing-whitespace command-name tag still counts, and a mixed slash+tool_use session counts 2 (documenting the no-dedup decision above).
- Full hook test suite: 37/37 passing post-fix.

Reviewed via `pr-review-toolkit:review-pr` — `code-reviewer` returned no findings; `pr-test-analyzer` findings were addressed in the same PR (the hardening items above).

## Wiki pages updated

- [[loop-progress-fields]] — `loop_stop_counts` section gains a note on why the field could silently stay null for a slash-started loop.
- [[discipline-loop]] — the "key off main-agent loop invocation count" line near the Stop-hook-composition section gains a correction: the count historically missed the slash-command form.
- [[evals-gate-enforcement-gap_2026-07-08]] — new investigation page (see Caveats below), documenting a workflow-uniformity gap surfaced by this PR's own debugging path.

## Caveats / gotchas

- **This bug was totally silent.** No hook fired an error; `loop_stall_guard` simply behaved as if no loop was active, for the entire duration of every slash-started loop. The 54-line all-zero log for session `3f897875` was the only evidence trail; there was no exception, no stderr line, nothing that would have surfaced without directly grepping the guard's own audit log.
- **Two earlier hypotheses were investigated and disproven before the real root cause was found**: an `ALS_PATH` keying mismatch (memory: `project_loop_stop_counts_null_bug.md`) and a tail-window blind spot in the transcript scan. Neither explains the observed all-zero `invocations=0` — only a structural detection miss across the entire transcript does. Do not re-open either of those as live hypotheses for this symptom class.
- **Workflow-uniformity gap surfaced during this fix.** The `task-evals` gate is hook-enforced only inside `agentic-loop` (via `loop_state_guard`, at ≥3 work-units). For a single-PR path, no hook forces `task-evals` at all — only `review-pr` invocation is gated (via `enforce_pr_workflow`'s `PreToolUse` hook). Critically, `systematic-debugging` — the natural entry skill for "there's a bug, go fix it" — has **zero mention of evals** (verified: `grep -niE "eval" skills/systematic-debugging/SKILL.md` returns nothing); its only real outbound chain is TDD → verify → verification-before-completion. **Verified directly against GitHub** (`gh pr view 95 --json comments` against `blueman82/coderails`): PR #95 carries **zero PR comments** — neither a review-artifact marker nor an eval-artifact marker. Since `scripts/merge.sh`'s own review- and eval-artifact gates are both unconditional, fail-closed checks against exactly such comment markers, this PR could not have been merged via `/coderails:merge` — it was merged by a path that bypasses that script (and both its artifact gates) entirely, most likely a direct `gh pr merge` once `enforce_pr_workflow`'s invocation-evidence check (not a comment-marker check) was satisfied. This PR's own fix was debugging-framed and routed around the evals gate in exactly the way the investigation describes; the concrete evidence trail (zero comments) additionally shows the comment-based artifact gates were bypassed structurally, not merely that a skill failed to prompt for them. See [[evals-gate-enforcement-gap_2026-07-08]] for the full investigation (a Thread-B fix for this gap is being handed off separately, not part of this PR).
