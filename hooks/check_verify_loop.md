---
title: "Hook: check_verify_loop"
type: hook
created: 2026-05-30
last_updated: 2026-07-14
sources:
  - hooks/scripts/check_verify_loop.sh
  - sources/session_2026-05-31_verify-loop-hardening.md
  - sources/session_2026-06-01_verify-loop-total-enforcement.md
  - sources/pr_57-62_subagent-enforcement-gate-hardening.md
  - sources/pr_76_harden-hook-stdin-read.md
  - sources/pr_156_dnv-presence-check.md
  - sources/pr_159_retire-catchup-add-telemetry.md
  - sources/pr_155-158_ceremony_noise_envelope_anchoring.md
  - sources/pr_163-168_dashboard-rethink.md
tags: [hook, stop-hook, subagentstop-hook, discipline, enforcement, did-not-verify, presence-check, turn-scoped, headless-exemption]
---

# Hook: check_verify_loop

A `Stop` and `SubagentStop` lifecycle hook with two independent enforcement paths: (1) blocks when the last response leaves **any** `## Did Not Verify` bullet untagged (total enforcement — a bullet not explicitly marked uncheckable is treated as something the model could have resolved and chose to defer), and (2) as of PR #156 (2026-07-13), blocks on the `Stop` path when the response has **no DNV header at all** after a turn that edited `>= 3` files (the presence check — closes the inversion where omitting the section entirely passed silently while an honest section with one untagged bullet blocked). Wired to both events as of PR #57; the presence check is Stop-only (see below for why). As of PR #167 (2026-07-14), the `Stop` event also exempts entirely when `CODERAILS_HEADLESS_RUN=1` is present in the process env — see "Headless-run exemption" below.

Source: `hooks/scripts/check_verify_loop.sh`

## What it enforces

Every `## Did Not Verify` bullet must be either **resolved** (read the file, run the check) before the response ends, or **explicitly tagged** as genuinely uncheckable. There is no middle state. A plain-prose claim ("the dedup test catches the bug") blocks exactly as a filename does — both are things the model could have confirmed. (verified: check_verify_loop.sh:2–12, 118–122)

The only bullet that passes is one whose leading clause is an explicit `(unverifiable: <reason>)` tag. (verified: check_verify_loop.sh:135)

## Ordered checks

The script detects `hook_event_name` and takes a different path depending on the event.

**SubagentStop path (PR #57):** reads `.last_assistant_message` directly. The `file_count` gate is NOT applied — the subagent's message is the authoritative output; an untagged DNV bullet in it is proof of deferred work regardless of whether the agent transcript is readable. `transcript_path` on a SubagentStop payload is the parent session transcript, not the subagent's, so reading it would check the wrong content.

**Stop path:** reads the last assistant text block from `transcript_path`. As of PR #61, the `file_count < 1` early-exit gate on the **bullet-tagging** check is **removed** — a `## Did Not Verify` section that exists is policed regardless of whether files were edited this turn. As of PR #156 (2026-07-13), `file_count` gates a *separate* check instead: the **presence** check (see below), which fires when the header is missing entirely and the turn edited `>= 3` files. This makes the Stop path's bullet-tagging behaviour consistent with SubagentStop, while giving `file_count` a new, narrower job.

Checks run top-to-bottom after the path branching. The first that matches decides. All but the last two allow the stop; an untagged DNV bullet or a missing header after enough edits blocks. (verified: check_verify_loop.sh)

| Check | Condition | Outcome |
|---|---|---|
| headless (Stop only, PR #167) | `CODERAILS_HEADLESS_RUN=1` in process env | allow stop — exit 0, no warn text, `skipped=headless` logged |
| no transcript (Stop) | No transcript path in payload, or file does not exist | allow stop |
| loop-guard | `stop_hook_active == true` (already blocked once this turn) | allow stop |
| no text | Last assistant response has no text | allow stop |
| **presence** (Stop only, PR #156) | No `## Did Not Verify` header at all AND turn `file_count >= 3` | **exit 2** (or loop-demoted warn) |
| no DNV, file_count < 3 | No DNV header, but fewer than 3 files edited this turn | allow stop — nothing to enforce |
| header present, zero bullets | DNV header exists but has no bullets (prose-only, "nothing outstanding") | allow stop — compliant empty section |
| **block** | Any DNV bullet **not** tagged `(unverifiable: …)` | **exit 2** (or loop-demoted warn) |
| all tagged | DNV header present, every bullet carries `(unverifiable: …)` | allow stop |

Note: the previous "conversation only" gate (`file_count < 1`) was removed from the **bullet-tagging** path on Stop in PR #61 — that removal is unrelated to the presence check, which is a different code path added later (PR #156) that reintroduces a `file_count` gate, just for a different question ("does a header exist at all," not "are its bullets tagged"). The SubagentStop path never had either gate, and the presence check cannot fire on SubagentStop regardless — `file_count` is never computed there (always `0`), so the `>= 3` condition never trips. (verified: check_verify_loop.sh header comment + lines 134-164)

## file_count is now TURN-scoped, not session-cumulative (PR #156)

`dc_file_count()` (in `hooks/scripts/lib/discipline_common.sh`) used to count `Write`/`Edit`/`MultiEdit` targets across the whole transcript. As of PR #156 it finds the last "genuine user" record — a `user`-type record whose `message.content` is a non-empty string, or an array containing a text block (a tool-result-only array does not count as genuine) — and only counts files touched after that cutoff. This matches the CLAUDE.md self-checking-discipline wording, which is per-response ("after any response that edits files"), not per-session: a session that edited 5 files across turns 1–2 and then has a pure-conversation turn 3 no longer gets nagged for a missing DNV section on turn 3. Falls back to counting the whole transcript when no genuine user record exists (test fixtures). Also hardened with the same per-line tolerant jq parse (`jq -R 'fromjson? // empty' | jq -s ...`) the text extractors already used — a single malformed transcript line no longer zeroes the count for the rest of the turn. (verified: discipline_common.sh:11-24)

## The escape hatch — the only way past

A bullet passes only if its leading clause is an explicit tag:

```
- (unverifiable: <reason>) <the item>
```

The detection regex anchors the tag to the start of the bullet (verified: check_verify_loop.sh:135):

```
^- *\(unverifiable:
```

Anchoring to the leading clause (right after `- `) is deliberate: the tag cannot be sprinkled mid-sentence to dodge a real claim. The reason is meant for the genuinely-uncheckable: a REPL-only action, external-system behaviour, prod-only observation, or user intent. Because the tag is greppable, overuse is visible on review.

Untagged bullets are counted by inverting the tag match, then counting bullets that carry any non-whitespace content — a bare `- ` is not a claim and does not block. Any count `> 0` blocks. (verified: check_verify_loop.sh:136–139, 143)

## Honest boundary

This hook forces every DNV item to be **resolved or explicitly tagged**. It **cannot** force the tag to be truthful — tagging a checkable item is cheaper than checking it, and the hook has no way to tell the difference. The guarantee is therefore *"nothing is silently deferred,"* not *"everything was actually verified."* The `(unverifiable: …)` tag is the auditable seam: it makes every deferral a deliberate, greppable, reviewable declaration. (verified: check_verify_loop.sh:10–12, 131–134)

## Transcript-flush race mitigation

The hook retries `extract_last_text` with configurable backoff (`MAX_ATTEMPTS`, `SLEEP_S`) until the extracted length stabilises at a positive value. Each assistant entry is reduced to a joined string of its text blocks; a non-text entry contributes `""` and can never shadow a real text block. (verified: check_verify_loop.sh:62–92)

## Loop-guard

`stop_hook_active` from the hook payload is checked before any expensive transcript parsing. When the model re-runs after a block and trips this hook again, the loop-guard exits 0, allowing the second stop. This is what makes total enforcement safe: it blocks once, then lets the re-stop through, so there is no deadlock. (verified: check_verify_loop.sh:56–60)

## Loop-scope demotion (PR #155 bullet path; PR #156 wires presence through the same predicate)

On a `Stop` event inside an active, incomplete [[agentic-loop]] session, both the bullet-tagging block and the presence block demote to a model-visible `additionalContext` warn (`[discipline-warn(loop)] ...`) instead of `exit 2`, via the shared `als_loop_active_incomplete` predicate from `lib/loop_state_common.sh`. `SubagentStop` never demotes — worker output stays fully block-enforced regardless of loop state. (verified: check_verify_loop.sh:150-156, 225-238)

PR #155 (a concurrent session's work, referenced here not duplicated) added the demotion for the bullet path first. PR #156 wired the newer presence check through the *same* predicate rather than adding a separate in-loop hard block — the stated reasoning is consistency: once #155 landed, a hard presence-block in-loop would have been the only remaining in-loop hard block in the file. The demotion evaluation is lazy (only runs once a block is imminent), so non-loop sessions never pay the cost of scanning the transcript for loop state.

## Logging

Appends a structured key=value line to `$CLAUDE_DISCIPLINE_LOG` (default `~/.claude/discipline.log`) at multiple points (empty-text skip, presence-check outcome, bullet-tagging outcome), including `session_id`, `text_len`, `attempts`, `files`, `dnv_items`, `resolvable_dnv_items`, and — as of PR #159 (2026-07-13) — `event=<hook_event_name>` (`Stop` or `SubagentStop`), enabling main-agent-vs-subagent segmentation of the log that was previously structurally impossible to reconstruct after the fact. Loop-demoted lines also carry `would_block=1 warned=1 blocked=0` so a demotion is distinguishable from a genuine pass. (verified: check_verify_loop.sh:69, 89, 122, 153, 158, 162, 215, 235, 240)

## History

The hook went through four generations:

1. **"Ran a tool" check** (original): asked whether any Read/Grep/Bash tool call appeared after the DNV section. It carried a jq operator-precedence bug that made the branch evaluate incorrectly. Both the weak check and the buggy expression were deleted. (inferred: prior session knowledge — absent from the current file, repo carries no git history)
2. **Source-token regex + meta-bullet exclusion** (2026-05-31): blocked only bullets naming a `file.ext` or `file:line` token, with a `meta_pattern` allowlist of leading-clause phrases ("nothing outstanding", "scoped out", etc.) to drop false positives. This left prose claims that named no file completely unpoliced. See [[session_2026-05-31_verify-loop-hardening]].
3. **Total enforcement** (2026-06-01): the source-token regex and the `meta_pattern` allowlist were both removed. Now *any* untagged bullet blocks, prose or filename, and the single `(unverifiable: …)` tag is the only escape. This closed the gap where a checkable prose claim slipped through because it named no file. See [[session_2026-06-01_verify-loop-total-enforcement]].
4. **SubagentStop + file_count removal** (2026-06-26): wired to SubagentStop (PR #57) with `last_assistant_message` as the text source, and the `file_count < 1` gate removed on the Stop path (PR #61) so pure-conversation turns with a DNV section are also policed. See [[pr_57-62_subagent-enforcement-gate-hardening]].
5. **Presence check + turn-scoped file_count + loop demotion** (2026-07-13, PR #156, building on PR #155's concurrent bullet-path demotion): closed the inversion where omitting the `## Did Not Verify` header entirely passed silently while an honest section with an untagged bullet blocked. Added a `file_count >= 3` + no-header presence block on the Stop path, re-scoped `file_count` from session-cumulative to TURN-scoped (honouring CLAUDE.md's per-response wording), hardened `dc_file_count`'s jq parse to be per-line tolerant (closing the last of the jq-slurp fragility family), and wired the new check through the same loop-demotion predicate PR #155 added for the bullet path. See [[pr_156_dnv-presence-check]].

## Stdin read convention (PR #76)

This hook reads its payload via `IFS= read -r -d '' -t 5 input || true`. The 5-second timeout is an in-process backstop for the orphaned-hook scenario (parent dead, stdin never closed). On timeout, `input` is empty → the hook exits 0. The `|| true` is mandatory: `read -d ''` returns exit 1 on normal EOF. See [[pr_76_harden-hook-stdin-read]].

## Related

- [[enforcement-model]]
- [[discipline-loop]]
- [[install-and-cache-trap]]
- [[pr_156_dnv-presence-check]] — the presence-check PR, turn-scoping, jq hardening
- [[pr_159_retire-catchup-add-telemetry]] — the `event=` telemetry PR
- [[agentic-loop]] — the loop-demotion predicate's consumer context
- [[pr_155-158_ceremony_noise_envelope_anchoring]] — PR #155's full mechanism writeup (predicate truth table, lazy evaluation, fail-toward-blocking `jq` emission), the retro-mining `blocked=1` metric guidance, and the accepted cosplay-loop residual (a never-completing loop holds discipline at warn indefinitely; contained by `SubagentStop` always blocking + the log staying visible for retro mining)
