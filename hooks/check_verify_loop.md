---
title: "Hook: check_verify_loop"
type: component
created: 2026-05-30
last_updated: 2026-06-01
sources:
  - hooks/scripts/check_verify_loop.sh
  - sources/session_2026-05-31_verify-loop-hardening.md
  - sources/session_2026-06-01_verify-loop-total-enforcement.md
tags: [hook, stop-hook, discipline, enforcement, did-not-verify]
---

# Hook: check_verify_loop

A `Stop` lifecycle hook that blocks Claude Code (exit 2) when the last response leaves **any** `## Did Not Verify` bullet untagged. Total enforcement: a bullet that is not explicitly marked uncheckable is treated as something the model could have resolved and chose to defer.

Source: `hooks/scripts/check_verify_loop.sh`

## What it enforces

Every `## Did Not Verify` bullet must be either **resolved** (read the file, run the check) before the response ends, or **explicitly tagged** as genuinely uncheckable. There is no middle state. A plain-prose claim ("the dedup test catches the bug") blocks exactly as a filename does — both are things the model could have confirmed. (verified: check_verify_loop.sh:2–12, 118–122)

The only bullet that passes is one whose leading clause is an explicit `(unverifiable: <reason>)` tag. (verified: check_verify_loop.sh:135)

## Ordered checks

Checks run top-to-bottom. The first that matches decides. All but the last allow the stop; only an untagged DNV bullet blocks. (verified: check_verify_loop.sh:14–21)

| Check | Condition | Outcome |
|---|---|---|
| no transcript | No transcript path in payload, or file does not exist | allow stop |
| conversation only | No files edited this turn (Write/Edit/MultiEdit targets) | allow stop |
| loop-guard | `stop_hook_active == true` (already blocked once this turn) | allow stop |
| no text | Last assistant response has no text | allow stop |
| no DNV | No `## Did Not Verify` (or `## Not Verified`) bullets in the response | allow stop |
| **block** | Any DNV bullet **not** tagged `(unverifiable: …)` | **exit 2** |

File-edit threshold: the hook skips when `file_count -lt 1`, i.e. zero files edited (verified: check_verify_loop.sh:52). A single edited file brings the response in scope. This was lowered from `< 3` so single-file sessions are no longer unguarded (see [[install-cache-trap_2026-05-30]]).

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

## Logging

Appends a structured key=value line to `$CLAUDE_DISCIPLINE_LOG` (default `~/.claude/discipline.log`) at two points: once after the DNV-item count is known, and again on block, both including `session_id`, `text_len`, `attempts`, `files`, `dnv_items`, and `resolvable_dnv_items`. (verified: check_verify_loop.sh:108, 141, 144)

## History

The hook went through three generations:

1. **"Ran a tool" check** (original): asked whether any Read/Grep/Bash tool call appeared after the DNV section. It carried a jq operator-precedence bug that made the branch evaluate incorrectly. Both the weak check and the buggy expression were deleted. (inferred: prior session knowledge — absent from the current file, repo carries no git history)
2. **Source-token regex + meta-bullet exclusion** (2026-05-31): blocked only bullets naming a `file.ext` or `file:line` token, with a `meta_pattern` allowlist of leading-clause phrases ("nothing outstanding", "scoped out", etc.) to drop false positives. This left prose claims that named no file completely unpoliced. See [[session_2026-05-31_verify-loop-hardening]].
3. **Total enforcement** (2026-06-01): the source-token regex and the `meta_pattern` allowlist were both removed. Now *any* untagged bullet blocks, prose or filename, and the single `(unverifiable: …)` tag is the only escape. This closed the gap where a checkable prose claim slipped through because it named no file. See [[session_2026-06-01_verify-loop-total-enforcement]].

## Related

- [[enforcement-model]]
- [[discipline-loop]]
- [[install-and-cache-trap]]
