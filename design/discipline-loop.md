---
title: Discipline Loop
type: design
created: 2026-05-30
last_updated: 2026-07-24
sources:
  - sources/pr_295_discipline_common_inner_shape_guards.md
  - sources/pr_290_sse_teardown_and_jq_object_guard.md
  - sources/pr_108_2026-07-08_offload-push-guard.md
  - sources/pr_238_crack-on-prose-gate.md
  - sources/pr_63_remove-failure-log.md
  - hooks/scripts/check_confidence_labels.sh
  - hooks/scripts/check_verify_loop.sh
  - CLAUDE.md
  - sources/pr_206_208_loop-state-common-docs-and-robustness.md
  - instructions/self-checking-discipline.md
  - sources/session_2026-05-31_prompting-doc-alignment.md
  - sources/pr_57-62_subagent-enforcement-gate-hardening.md
  - sources/pr_78_hooks-json-timeout-floor.md
  - sources/pr_15-17_loop-hardening-registration-eval-freeze-ledger-dry.md
  - sources/pr_70-71_2026-07-07_dashboard-input-fix-and-voice-announcements.md
  - sources/pr_95_slash-command-loop-detection.md
  - sources/pr_99_unregistered-loop-guard-nudge-once.md
  - sources/pr_156_dnv-presence-check.md
  - sources/pr_159_retire-catchup-add-telemetry.md
  - sources/pr_155-158_ceremony_noise_envelope_anchoring.md
  - sources/pr_163-168_dashboard-rethink.md
  - sources/pr_175-176_crack-on-gate-and-inbox-brief-button.md
tags:
  - discipline
  - hooks
  - confidence-labels
  - did-not-verify
  - enforcement
---

# Discipline Loop

The self-checking discipline system: what it is, what it enforces mechanically, and what it only requests.

## The Founding Thesis

The empirical failure that motivated block-mode (originally recorded in the seeded `templates/failure_log.md`, which was **removed in PR #63** — see [[pr_63_remove-failure-log]]; the record now lives only in the maintainer's live `~/.claude/failure_log.md`, user data rather than a repo artifact):

> Multiple turns this session lacked the DNV section while warn-mode hooks fired silent reminders that I ignored. The user explicitly observed "another claude session just ignored the hooks prompt" — same failure mode. **Warn-mode + memory-only enforcement is mechanically insufficient** — confirms the Shingo prediction quoted during the build.

This is the founding data point. The discipline system existed in warn-mode first. It was tested in a live session. It failed. Claude ignored the hooks. The lesson: a rule that only reminds does not produce the behaviour. A rule that blocks does.

## Two Disciplines, Two Enforcement Levels

### Advisory: the CLAUDE.md prose rules

`instructions/self-checking-discipline.md` installs rules into `~/.claude/CLAUDE.md` via `install.sh`. These cover:

- Confidence labels: tag non-trivial assertions `(verified)`, `(inferred)`, or `(guess)`
- Did Not Verify section: after multi-file changes, list what was NOT checked
- Ask on ambiguity: use `AskUserQuestion` rather than silently filling in interpretations
- Verify memory before acting: re-read/grep the file rather than recalling

These are prose instructions to Claude. Claude *should* follow them. They are not enforced by any gate.

### Mechanically Enforced: the Stop hooks

Two Stop hooks promote the most important of those rules into block-mode gates.

**`check_confidence_labels.sh`** — blocks responses ≥200 chars that carry no confidence label.

From the script header (verified): *"BLOCK-MODE: exits 2 when confidence labels are missing (promoted from warn-mode 2026-05-05)."*

Logic at lines 44–66 (verified):
- Reads the last assistant text block from the transcript, retrying up to 5 times with 0.3s backoff to handle the transcript-flush race.
- If text length < 200 chars: passes (short responses are not substantive claims).
- If text contains `(verified`, `(inferred`, or `(guess`: passes.
- Otherwise: blocks with `exit 2` and a message naming the failure.

**`check_verify_loop.sh`** — **total enforcement** (as of 2026-06-01) on two independent checks: (1) blocks when *any* `## Did Not Verify` bullet is left untagged (sole escape: an explicit `(unverifiable: <reason>)` tag on the bullet's leading clause), and (2) as of PR #156 (2026-07-13), blocks on the Stop path when the response has **no DNV header at all** after a turn that touched `>= 3` files — closing the inversion where omitting the section entirely used to pass silently while an honest section with one untagged bullet blocked.

As of PR #61, the `file_count < 1` gate on the bullet-tagging check's Stop path was removed — an *existing* DNV section is policed regardless of whether files were edited. As of PR #57, the hook also fires on SubagentStop, reading `.last_assistant_message` directly. As of PR #156, `file_count` is TURN-scoped (records after the last genuine user prompt, not session-cumulative) and gates the *separate* presence check instead — see [[check_verify_loop]] for the full turn-scoping mechanics.

As of PR #155 (2026-07-13, a concurrent session's work), both checks demote from a hard block to a model-visible `additionalContext` warn on `Stop` events inside an active, incomplete [[agentic-loop]] session — `SubagentStop` never demotes, so worker output stays fully block-enforced.

As of PR #167 (2026-07-14, [[pr_163-168_dashboard-rethink]]), both hooks also **exempt entirely** (exit 0, no warn text, `skipped=headless` logged) on the `Stop` event when `CODERAILS_HEADLESS_RUN=1` is present in the process env — a third condition alongside "outside a loop → block" and "inside an active incomplete loop → warn." This is narrower than the loop-warn demotion: it has exactly one legitimate set-site (the dashboard's `POST /api/run` spawn call, for a headless `claude -p` run with no interactive turn left to satisfy a repair-turn block) and, like the loop-warn demotion, never applies to `SubagentStop` — worker output stays block-enforced regardless of the flag. See [[enforcement-model]]'s "headless-run exemption" section for the ceiling framing.

Gate chain (verified: [[check_verify_loop]]):
1. No transcript file (Stop path only) — allow stop.
2. `stop_hook_active == true` — allow stop (loop-guard: already blocked once this turn).
3. Last response has no text — allow stop.
4. No `## Did Not Verify` header AND turn `file_count >= 3` (Stop only) — **block** (or loop-demoted warn) — PR #156.
5. No `## Did Not Verify` header, fewer than 3 files this turn — allow stop, nothing to enforce.
6. Header present, zero bullets (prose-only, "nothing outstanding") — allow stop, compliant empty section.
7. Any DNV bullet **not** tagged `(unverifiable: …)` — **block** (or loop-demoted warn) with `exit 2`.

The earlier source-token regex (matching `.md`, `.sh`, etc. extensions) and the `meta_pattern` allowlist were both removed in the 2026-06-01 escalation. Now prose claims block exactly as filename claims do — any untagged bullet is treated as something the model could have confirmed. See [[check_verify_loop]] for the full history. (inferred: [[session_2026-06-01_verify-loop-total-enforcement]])

## What This Means in Practice

Writing a DNV section that says:

```markdown
## Did Not Verify
- Whether the CI pipeline will pass
- User intent on the edge case
```

passes both hooks. Neither bullet names a file.

Writing:

```markdown
## Did Not Verify
- prep.md:96 — the exact config field name
- The output will be correct when run
```

both block (as of 2026-06-01). The first names a file; the second is prose — but neither carries the `(unverifiable: …)` tag, so both are treated as deferrals the model could have resolved.

The only passing form for a genuinely uncheckable item:

```markdown
- (unverifiable: external-system behaviour) Whether the CI pipeline will pass
```

This is the design intent: nothing is silently deferred. Genuinely unverifiable things (future runtime, external systems, user intent) may stay in DNV only if the tag makes the deferral explicit and greppable.

## Advisory vs. Enforced: Summary

| Rule | Where it lives | Enforcement |
|---|---|---|
| Confidence labels on non-trivial claims | `~/.claude/CLAUDE.md` prose | Advisory (demotes to loop-warn per PR #155 inside an active loop) |
| ≥200-char response must have one label | `check_confidence_labels.sh` | **Block (Stop hook)**, demoted to warn inside an active incomplete loop (PR #155); SubagentStop always blocks |
| Did Not Verify section after file-editing responses | `~/.claude/CLAUDE.md` prose | Enforced as of PR #156 (2026-07-13) — see next row. Previously advisory-only; the DNV-section-presence gap was the actual live enforcement hole this closed |
| Any untagged `## Did Not Verify` bullet, OR a missing `## Did Not Verify` header after `>= 3` files edited this turn | `check_verify_loop.sh` | **Block (Stop + SubagentStop hook)**; Stop-path blocks demote to warn inside an active incomplete loop (PR #155/#156); SubagentStop always blocks |
| Ask on ambiguity, verify memory before acting | `~/.claude/CLAUDE.md` prose | Advisory |

The prose rules are not redundant. They cover the cases the hooks don't — short responses, conversational turns, runtime claims. The hooks cover the high-value case where Claude might stop with a false claim of completeness.

## Prose as Standard, Hook as Floor (key design invariant)

This is the central layering principle of the discipline loop, made explicit in `instructions/self-checking-discipline.md:10–11` (verified) as of 2026-05-31.

The **prose rules** (in `~/.claude/CLAUDE.md`) describe the standard Claude *should* aim for:
- Label every substantive claim with a confidence tag
- Write a Did Not Verify section after any response that edits one or more files

The **Stop hooks** enforce a *floor* that is intentionally lower than the prose standard:
- `check_confidence_labels.sh` only blocks responses ≥200 chars with no label at all — it does not require every claim be labelled, just that the response isn't entirely unlabelled (verified: [[check_confidence_labels]])
- `check_verify_loop.sh` (total enforcement as of 2026-06-01) blocks when any DNV bullet is untagged — the floor is: every deferral must carry an explicit `(unverifiable: …)` tag, or be resolved. As of PR #156 (2026-07-13), the hook **also** requires the section to exist at all once a turn has edited `>= 3` files — closing the gap this section used to describe as "the hook does not require a DNV section on every response." That gap is now narrower: no header is still allowed under 3 files edited this turn, but a compliant empty header ("nothing outstanding") is always allowed regardless of file count. (verified: [[check_verify_loop]])

**Why the floor is lower than the standard:** mechanical hooks cannot encode nuanced judgment about what "substantive" means in context, or when a DNV section genuinely adds value vs. boilerplate. The prose standard asks Claude to apply judgment; the hooks catch the case where that judgment fails in a high-stakes way (a long response with no accountability markers, or a DNV that lists an unread file as a known gap, or — as of PR #156 — a response that silently drops the section after real edits). See [[hook-exit-codes]] for the block mechanisms.

**What this means in practice:** following the hook floor is necessary but not sufficient. A response can satisfy both hooks (≥1 label, no untagged DNV bullet, header present if required) while still falling short of the prose standard (many unlabelled claims, a DNV section that exists but under-covers what was actually deferred). The prose standard is the real target. The hooks are the backstop for clear failures — and PR #156 moved the floor closer to the standard for the specific "omitted the section entirely" failure mode.

## The Stop hook composition (8 Stop hooks; 3 SubagentStop hooks)

**Correction (2026-07-22, wiki-lint):** the two array walkthroughs below (originally written PR #57/#71-era and last touched PR #175, 2026-07-14) fell out of date without a matching content edit — `crack_on_prose_gate.sh` (PR #238, 2026-07-17) and `offload_push_guard.sh` (PR #108, predates this page's last edit) are both live in `hooks/hooks.json` but were never added to either enumerated list. Verified directly against `hooks/hooks.json` on origin/main (2026-07-22): the `Stop` array has **8** entries, the `SubagentStop` array has **3**. Both are corrected below in place; the surrounding prose (gate-chain mechanics, enforcement-ceiling table, cross-references) was already accurate for these two hooks elsewhere on this page and needed no change.

**Retirement note (PR #159, 2026-07-13):** `discipline_catchup.sh`, the sole `UserPromptSubmit`-event discipline hook, was retired clean-break — file and test both deleted, `hooks.json`'s `UserPromptSubmit` array is now `inject_context.sh` only. Reason: a flat 23–26% first-attempt miss rate held steady since block-mode shipped 2026-05-05 — the warn-mode catchup nudge measurably added nothing on top of the two block-mode Stop hooks below. See [[discipline_catchup]] (now marked retired) and [[pr_159_retire-catchup-add-telemetry]]. The composition below is unaffected — it was always Stop/SubagentStop-only.

**New `UserPromptSubmit` + `PreToolUse` addition (PR #175, 2026-07-14):** [[crack_on_gate]] adds a second `UserPromptSubmit` hook alongside `inject_context.sh` (stamps a session-scoped `crack_on_active` flag on a raw-prompt match for "crack on") plus a new `PreToolUse` matcher scoped to `AskUserQuestion` (denies while the flag is stamped). This is a different discipline axis from the Stop/SubagentStop confidence-label and DNV gates documented in this page's core sections below — it governs *when the model may ask the human a question at all*, not what a completed response must contain. See [[crack_on_gate]] for the full raw-prompt-only detection rationale and the deliberate divergence from `agentic_loop_path.sh`'s resolver.

The coderails Stop hook array has eight hooks, running in order:

1. `voice_announce` — **observe-only, always exits 0** — speaks a loop lifecycle event (complete / waiting / stopped / stall) via macOS `say`; runs first specifically because it cannot affect the other gates and must not be short-circuited by one of them (added PR #71, [[pr_70-71_2026-07-07_dashboard-input-fix-and-voice-announcements]]; see [[voice_announce]])
2. `check_confidence_labels` — confidence labels gate; demotes to loop-warn inside an active incomplete loop as of PR #155
3. `check_verify_loop` — DNV resolution gate (untagged-bullet check) plus DNV-presence gate (missing header after `>= 3` files this turn, PR #156, 2026-07-13); both demote to loop-warn inside an active incomplete loop
4. `crack_on_prose_gate` — **block** (exit 2) a final message that hands a question to the user in prose while the session's crack-on flag is stamped; the prose-half counterpart to [[crack_on_gate]]'s tool-deny on `AskUserQuestion`, deterministic two-tier heuristic, fail-closed, capped at 3 blocks per turn (added PR #238, 2026-07-17; see [[crack_on_prose_gate]])
5. `loop_state_guard` — `progress.json` presence/ownership gate ([[agentic-loop]] sessions only)
6. `loop_stall_guard` — `LOOP-STOP` declaration gate (agentic-loop sessions only)
7. `unregistered_loop_guard` — nudge (never blocks) when a loop looks unregistered: ≥3 distinct agent-dispatch turns, no `progress.json`, no `agentic-loop` Skill invocation (added PR #17, [[pr_15-17_loop-hardening-registration-eval-freeze-ledger-dry]]). **Nudges at most once per session (PR #99, [[pr_99_unregistered-loop-guard-nudge-once]]):** the original version re-emitted the nudge on every Stop for a session that kept meeting the trip conditions, which self-perpetuated for a genuinely one-off dispatch sequence (nudge → honest "no action needed" turn → Stop → nudge again, observed live 2026-07-08). The fix greps the existing discipline log for a prior `nudged=1` line for this session id (BRE-escaped before interpolation, closing a wildcard-match regression) before emitting again; the first nudge for a session is unaffected, and a missing/unreadable log still fails open (nudges rather than wrongly suppresses).
8. `offload_push_guard` — nudge (never blocks) when the final message both names a `git push` targeting main/master AND carries an offload-to-user cue ("run this yourself", a leading `! `, etc.) — the case where a session hands the user a push that `enforce_pr_workflow.sh` would have gated, sidestepping the gate by proxy; runs last in the array (added PR #108, [[pr_108_2026-07-08_offload-push-guard]]; see [[offload_push_guard]])

The SubagentStop hook array has three hooks, both `check_confidence_labels`/`check_verify_loop` wired PR #57 and `offload_push_guard` added later (PR #108) — all always block/nudge-enforced regardless of loop state (the PR #155/#156 loop-demotion is Stop-only; a nudge hook has no block state to demote):
1. `check_confidence_labels` — reads `.last_assistant_message`; same MIN_LEN/label logic as Stop
2. `check_verify_loop` — reads `.last_assistant_message`; untagged-bullet DNV enforcement same as Stop; the presence check (PR #156) structurally cannot fire on this path — `file_count` is never computed for SubagentStop and is always `0`, so its `>= 3` condition never trips
3. `offload_push_guard` — same nudge logic as the Stop-path entry above, reading `.last_assistant_message` directly on this path (same rationale as `check_confidence_labels.sh`)

Both hooks log `event=SubagentStop` on this path as of PR #159 (2026-07-13), distinguishing these log lines from the Stop-path equivalents.

The three loop-state hooks (C1/C2 plus the new unregistered-loop nudge) remain **Stop-only** — they key off main-agent loop invocation count, session-owned `progress.json`, or (the new hook) the *absence* of one. A subagent has no `progress.json` to validate.

**Correction (PR #95, 2026-07-08):** the "loop invocation count" this section describes historically missed loops started via the **slash-command form** (`/coderails:agentic-loop`), which the transcript records as a `user`-role message with a string `.message.content` carrying `<command-name>...</command-name>` — not an assistant `Skill` tool_use. `als_count_invocations` (`hooks/scripts/lib/loop_state_common.sh`) only matched the tool_use form, so a slash-started loop counted 0 invocations and every shared consumer (`als_gate_require_active_loop`, and downstream `als_load_progress`/`gate_loop_stop_declared`/`bump_loop_stop_count`) treated the session as not-a-loop. The count now covers **both** forms: the assistant `Skill` tool_use form and the user `<command-name>` slash form (anchored on the same `(^|:)agentic-loop$` name test after stripping the leading `/`). See [[pr_95_slash-command-loop-detection]] for the fix mechanics and hardening details.

`loop_state_guard` and `loop_stall_guard` pass immediately when no agentic-loop session is active, adding zero overhead to normal single-PR sessions; `unregistered_loop_guard` is the one loop-state hook designed to fire specifically when a loop-shaped session has NOT registered — see [[unregistered_loop_guard]] for why that's a nudge, not a block, unlike its two Stop-only siblings. See [[spec-plan-progress-artifact-chain]] for the two-hook (C1/C2) guard architecture that predates it.

## Enforcement ceilings (documented PR #62)

These are deliberate limits, not bugs. See `CLAUDE.md` "Enforcement ceilings" section for the canonical list.

- **Bash blocklists are enumerated families, not exhaustive.** `destructive_bash_gate` and the in-Bash source-edit gate catch known patterns; obfuscated forms, variable filenames, quoted paths with spaces, here-docs, process substitution remain uncaught.
- **`no_edit_on_main` allowlist breadth is intentional (fail-safe).** `.sh` is blocked on main while `.json`/`.yaml` config stays editable — an accepted classification. Legitimate overrides use settings.json `Write`/`Edit` permission rules.
- **Wiki/workflow sequence past merge is advisory.** The `/workflow` chain (wiki-ingest + wiki-lint) after merge is a slash command — no hook enforces it.
- **check_verify_loop and the two loop guards short-circuit on `stop_hook_active=true` (at most once per turn).** check_confidence_labels does NOT read `stop_hook_active` and can re-block.
- **TDD is not enforced test-first.** `test_gate` only checks that tests pass at commit time.
- **Skill invocation is structurally unenforceable.** A hook cannot observe or mandate internal reasoning steps.
- **No `SubagentStart` event exists.** `inject_bootstrap.sh` cannot inject `using-coderails` into subagents; orchestrators must include it in subagent prompts explicitly.

## Stdin read convention (all hooks, PR #76)

All 10 hook scripts read their payload via `IFS= read -r -d '' -t 5 input || true` (replacing the old `input=$(cat)`). The 5-second timeout is an in-process backstop for the orphaned-hook scenario: if the parent process dies without closing stdin, the read times out, `input` is empty, and the hook exits 0 (fail-open). Fail-open is safe in this scenario — a dead parent means no live tool call to gate. The `|| true` is mandatory because `read -d ''` returns exit 1 on normal (non-timeout) EOF. See [[pr_76_harden-hook-stdin-read]] for details and test coverage.

**Timeout-floor invariant guard (PR #78):** Every hooks.json `timeout` for `hooks/scripts/` entries must be >= 5 (the `read -t` floor). If a timeout were declared below 5, the harness would kill the hook before the in-process backstop could fire. The invariant now has a machine check: `hooks/scripts/tests/hooks_json_timeout_floor.test.sh` parses `hooks.json` via jq and fails if `min(declared timeouts) < 5`. Also guards against the fractional-timeout silent-skip bug (jq comparison, not bash `-lt`) and empty-result vacuous-pass (explicit FAIL on zero extracted timeouts). See [[pr_78_hooks-json-timeout-floor]]. (verified: PR #78 body + script)

**Same test hardened again (2026-07-13, commit 109987b, same-day follow-up to PR #159):** the `UserPromptSubmit` guard previously checked only `[0].hooks`, so a hook re-added as a second matcher object (`[1]`) would pass unnoticed — this is exactly the shape a regression re-adding `discipline_catchup.sh` as a second matcher (rather than a second entry in the existing matcher's `hooks` array) would take. The guard now also asserts the matcher array itself has length 1. The same commit fixed a vacuous-pass in the test's version-lockstep check (see [[install-and-cache-trap]]) and added stderr-content coverage to `check_confidence_labels.test.sh`'s blocked-path case.

## Shared library: discipline_common.sh (added 2026-06-25, PR #29; discipline_catchup.sh consumer retired PR #159)

The discipline hooks previously duplicated the transcript text-extraction jq expression and retry loop. PR #29 extracted this into `hooks/scripts/lib/discipline_common.sh`, mirroring the pattern of `lib/loop_state_common.sh`. Behaviour-preserving (proven against origin/main pre-refactor). A TDD test was added. (verified — PR #29)

At the time of PR #29 there were three consumers: `check_confidence_labels.sh`, `check_verify_loop.sh`, and `discipline_catchup.sh`. **`discipline_catchup.sh` was retired PR #159 (2026-07-13)** — see [[discipline_catchup]] and the retirement note above; the library now has two consumers.

**jq-slurp fragility family, part 1 — the MALFORMED-line parse (PR #156, 2026-07-13):** `dc_file_count()` was the last of three known instances of a bare `jq -s` slurp that would zero its result entirely if any single transcript line was malformed or truncated (the other two — `dc_extract_last_text`'s text extraction and `unregistered_loop_guard`'s dispatch-turn count — were covered by [[pr_112-113_2026-07-08_jq-slurp-residuals-round2|PRs #112/#113]]). PR #156 applied the per-line tolerant two-stage parse (`jq -R 'fromjson? // empty' | jq -s ...`) to `dc_file_count`, and simultaneously re-scoped it from session-cumulative to TURN-scoped (see [[check_verify_loop]] for the mechanics). Both changes shipped in the same PR because they touch the same function.

**Part 2 — the VALID-SCALAR object guard (PR #290, 2026-07-24), a different hazard:** the two-stage parse above defends against a line that *doesn't* parse. It does **not** defend against a line that is valid JSON but a bare **scalar** (`42`, `"a bare json string"`): `fromjson?` **keeps** those, they reach `.type`, and jq errors out (`Cannot index string with string "type"`). Because stderr is discarded, that error is silent — the count returns 0 or the extraction returns empty, and the caller reads it as "no files touched" / "no text yet" rather than as a failure. [[pr_290_sse_teardown_and_jq_object_guard|PR #290]] added `select(type == "object")` to **both** functions (`map(select(type == "object")) as $lines` in `dc_file_count`; a `select(type == "object")` stage in `dc_extract_last_text`), with tests (k) and (l) interleaving scalar lines against genuine assistant turns. The same guard `als_extract_last_text` (`loop_state_common.sh`) received in [[pr_206_208_loop-state-common-docs-and-robustness|PR #208]]; #208 deliberately left `dc_extract_last_text` as "a separate, already-tracked, parked concern," and #290 is the PR that landed it.

> ✅ **CONTRADICTION RESOLVED 2026-07-24 by [[pr_290_sse_teardown_and_jq_object_guard|PR #290]].** From 2026-07-17 until that merge, this page carried a flag stating that "hardened in earlier PRs" was inaccurate for `dc_extract_last_text` — correctly, because the object guard was genuinely absent. The flag's own clearing condition ("do not treat `dc_extract_last_text` as fixed until a PR actually lands the guard") is now met. Re-verified against `origin/main` at ingest: `select(type == "object")` is present in `dc_file_count` (line 39) and `dc_extract_last_text` (line 74) — the only two functions #290 touched. (verified) **Scope, stated precisely because the two hazards are easy to conflate:** `unregistered_loop_guard.sh`'s `ulg_count_dispatch_turns` has its own *malformed-line* tolerant parse from [[pr_112-113_2026-07-08_jq-slurp-residuals-round2|PR #113]] but **no** object guard, so this is not family-wide object-guard closure. **Assessed during the #290 ingest and the hazard was LIVE there** — see the open-defect note on [[pr_290_sse_teardown_and_jq_object_guard]]: a single valid-scalar line made its stage-2 slurp jq-error and return 0, and the trailing `case` coercion left `ULG_PARSE_REASON` empty, so it reported "no dispatch turns" instead of a parse failure (verified — reproduced on a 3-line fixture at ingest). Nudge-only hook, so the impact was a missed nudge, not a gate failure. **✅ Closed 2026-07-24 by PR #293** (MERGED `2026-07-24T01:02:56Z`, `4340c4b`, branch `fix/context-trend-panel-styles` — a dashboard-CSS-titled PR that also landed the guard): `select(type == "object")` is now present at `unregistered_loop_guard.sh:105`, and the same 3-line fixture returns **2** against current `origin/main` (verified — executed 2026-07-24). So the object guard now covers all three functions in the family. Nor does #290 close the memory handoff `project_jq_slurp_round2_handoff`: read directly at ingest, that handoff's "2 remaining" are `dc_extract_last_text` and `ulg_count_dispatch_turns` (not `dc_file_count`), and its fix shape is the malformed-line parse — work PRs #112/#113 already completed on 2026-07-08. The reason the original claim mis-fired is worth keeping: the "hardened in earlier PRs" phrasing was accurate about the guard those PRs *added* — [[pr_112-113_2026-07-08_jq-slurp-residuals-round2|PR #112]] genuinely did give `dc_extract_last_text` malformed-line tolerance — and inaccurate only in implying a second, different guard came with it. Two hazards that share a pipeline are easy to describe as one.

**Part 3 — the WRONG-INNER-SHAPE hazard (✅ CLOSED in `discipline_common.sh` 2026-07-24 by [[pr_295_discipline_common_inner_shape_guards|PR #295]]), a third distinct failure:** `select(type == "object")` is necessary but not sufficient. A line can be a valid JSON **object** and still abort the slurp if its inner shape is wrong — `{"type":"assistant","message":"oops"}` passes the object guard, then `.message.content` indexes a bare string and jq errors out (`Cannot index string with string "content"`), taking the whole slurp with it. Same silent signature as parts 1 and 2: stderr discarded, count coerced to 0, extraction returns empty. **This was live in `dc_file_count` and `dc_extract_last_text` until 2026-07-24** — a fixture of one genuine assistant edit turn plus one wrong-inner-shape object gave `dc_file_count` → **0** (should be 1) and `dc_extract_last_text` → **empty** (should be the text) (verified — executed 2026-07-24 against the then-current `origin/main`). `ulg_count_dispatch_turns` was further along, and by a **trigger-independent** mechanism rather than a second shape guard: PR #293 gave it an rc capture on stage 2, and a nonzero rc sets `ULG_PARSE_REASON="jq_parse_error"`, echoes it to stderr, prints `0` and **returns at `unregistered_loop_guard.sh:131-136` — before the `case "$n" in (''|*[!0-9]*) n=0;;` laundering on `:137`** (verified — read on `origin/main`). So a wrong-inner-shape abort there is attributed as an untrustworthy count rather than reported as a quiet session. The function's own comment states the trade: a partial stage-2 skip is now over-attributed as total loss, which is the correct direction for a hook whose job is to distinguish those two states, because over-attribution is visible and under-attribution is not. **✅ Closed in `discipline_common.sh` 2026-07-24 by [[pr_295_discipline_common_inner_shape_guards|PR #295]]** (MERGED `2026-07-24T08:33:04Z`, merge commit `4a5c237`, head `87a0647`, branch `fix/discipline-common-rc-capture`), which took a **two-layer** approach rather than mirroring #293's single rc capture (verified — read on `origin/main` at ingest):

- **Layer 1 (recovery, primary)** — inline shape guards where the slurp indexes: `((.message | type) == "object")` before `.message.content` in both functions and inside `is_genuine_user`, plus `select(type == "object")` on each *content element* before indexing `.type`/`.name`. **Per-line**: the slurp completes and the count/text is recovered from surviving lines, so one bad line costs one line.
- **Layer 2 (net, explicitly not primary)** — stage 2 split into a captured `$tolerant` intermediate so `agg_rc=$?` is readable at all, the same *capture-first* shape [[pr_274_tier_gate_observability_fixes|#274]]'s fix 3 named. **Whole-slurp**: on abort the surviving lines are lost too, a stated trade, aimed at unenumerated hazards.

**Layer 2 here writes nothing to stderr and sets no reason global, deliberately — and this is the transferable constraint.** `dc_file_count` is called unconditionally on **every** Stop-hook turn from `check_verify_loop.sh`, *ahead of* that hook's own block-message write to the **same** stderr stream (`>&2` + `exit 2`). An attribution echo would land concatenated ahead of the model-facing block message on every blocked turn where the hazard existed anywhere in the transcript. So the identical fix shape was right in `unregistered_loop_guard.sh` — a nudge-only hook that writes nothing else to stderr, and whose sole caller reads the reason — and wrong here: **an attribution channel is worth adding only where something reads it and where it doesn't collide with a channel already load-bearing.**

> ⚠️ **Consequence, found at ingest and worth keeping: Layer 2 as shipped is behaviourally INERT.** With the stderr write dropped, the abort branch reduces to `printf '0'; return` (and `printf ''` in `dc_extract_last_text`) — byte-identical to what the fall-through already yields, since `n` is empty on abort and the trailing `case "$n" in (''|*[!0-9]*) n=0;; esac` laundering coerces it to `0` anyway. Verified by mutation, not by reading: deleting **both** early-return branches outright and re-running the exact fixtures the PR's tests (p)/(q) use gives identical output before and after — `dc_file_count` → `0`, `dc_extract_last_text` → empty (verified — executed 2026-07-24 against merged `origin/main`). Tests (p)/(q) therefore assert the right *behaviour* (fail open, no stderr) but **do not discriminate the mechanism they are named for**; they would pass with the Layer 2 scaffolding removed. Not a live defect — fail-open is the correct outcome either way, which is exactly why it went unnoticed. **Disposition owed**: remove the dead scaffolding and keep Layer 1 alone, or give Layer 2 an observable effect that doesn't collide with `check_verify_loop.sh`'s stderr. Contrast `ulg_count_dispatch_turns`, where the same net **is** live because it returns before the laundering `case` *and* sets `ULG_PARSE_REASON`.

**Family status after #295 — three functions, three different treatments; "the family is closed" would be an over-claim.** `dc_file_count` and `dc_extract_last_text` carry Layer 1 inner-shape guards **and** a (currently inert) Layer 2. `ulg_count_dispatch_turns` carries the scalar object guard (#293) and a **live** Layer 2, but **no** Layer 1 inner-shape guards — so a wrong-inner-shape line there still costs the whole slurp, attributed rather than silent. The effective protection in `discipline_common.sh` today is Layer 1 alone.

This means: edits to transcript-extraction logic now go in `discipline_common.sh`, not in each hook individually.

## Cross-References

- [[enforcement-model]] — why hooks can enforce things that commands cannot
- [[check_confidence_labels]] — the confidence-label Stop hook in detail
- [[check_verify_loop]] — the verify loop hook in detail (total enforcement as of 2026-06-01; DNV-presence check added PR #156, 2026-07-13)
- [[discipline_catchup]] — retired PR #159 (2026-07-13); page kept as historical record
- [[pr_156_dnv-presence-check]] — the presence-check PR, turn-scoped file_count, malformed-line jq-slurp tolerance (part 1 of the family above)
- [[pr_290_sse_teardown_and_jq_object_guard]] — part 2: the valid-scalar `select(type == "object")` guard on both `dc_file_count` and `dc_extract_last_text`, which resolved this page's 2026-07-17 CONTRADICTION
- [[pr_295_discipline_common_inner_shape_guards]] — part 3: the wrong-inner-shape Layer 1 guards + the Layer 2 rc-capture net, and why Layer 2 stays silent (and is currently inert)
- [[pr_159_retire-catchup-add-telemetry]] — the retirement + `event=` telemetry + version-lockstep PR
- [[pr_155-158_ceremony_noise_envelope_anchoring]] — PR #155's full warn-demotion mechanism (predicate truth table, lazy evaluation, fail-toward-blocking `jq` emission) and the retro-mining log-line metric (`blocked=1` count, not `would_block=1`), plus the accepted cosplay-loop residual
- [[pr_163-168_dashboard-rethink]] — PR #167's `CODERAILS_HEADLESS_RUN` Stop-only exemption on both hooks, the third condition alongside plain-block and loop-warn-demote
- [[crack_on_gate]] — the newest `UserPromptSubmit` + `PreToolUse` (AskUserQuestion) hook (PR #175, 2026-07-14); a different discipline axis (gating human-asks during a crack-on envelope) from the Stop/SubagentStop content gates this page centers on
- [[crack_on_prose_gate]] — the prose-half of `crack_on_gate`'s tool-deny; Stop-only, blocks a final message that asks the user a question in prose during a crack-on envelope (PR #238, 2026-07-17)
- [[offload_push_guard]] — nudge-only Stop + SubagentStop hook catching a push-to-main handed off to the user by proxy (PR #108, 2026-07-08)
- [[loop_state_guard]] — `progress.json` presence/ownership Stop hook (C1, added 2026-06-24)
- [[loop_stall_guard]] — `LOOP-STOP` declaration Stop hook (C2, added 2026-06-24)
- [[unregistered_loop_guard]] — unregistered-loop nudge Stop hook (added 2026-07-06, PR #17; nudges at most once per session as of PR #99)
- [[voice_announce]] — observe-only voice-lifecycle Stop hook, first in the array (added 2026-07-07, PR #71)
- [[pr_95_slash-command-loop-detection]] — fixes the loop-invocation-count blind spot for slash-started loops (2026-07-08)
- [[pr_99_unregistered-loop-guard-nudge-once]] — fixes the self-perpetuating nudge and a grep-metachar false-suppression regression (2026-07-08)
- [[spec-plan-progress-artifact-chain]] — the two-hook loop-state guard design
- [[hook-exit-codes]] — which hook events block on exit 2 vs. permissionDecision: deny
- [[install-and-cache-trap]] — hook edits in the repo do not take effect until cache is re-synced
