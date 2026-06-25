---
title: "/coderails:verify"
type: command
created: 2026-06-25
last_updated: 2026-06-25
sources: [commands/verify.md]
tags: [command, discipline, introspection, verification, advisory]
---

# /coderails:verify

Re-derive a specific claim from session sources only — no recall, no inference, just evidence — and state precisely what is missing if the claim cannot be fully sourced.

## Invocation

```
/coderails:verify <claim>
```

`<claim>` is the specific assertion to verify. Pass it as the argument; it becomes `$ARGUMENTS` in the command body.

Example:

```
/coderails:verify "prep.md reads the epic field from config.jira.epic_field"
```

## What it does

1. Takes the claim passed as `$ARGUMENTS`.
2. Re-derives it using **only** sources available in the current session: tool results, file contents already read, user statements, git output. No recall. No inference. No "I believe" or "should be."
3. For each derivation step, cites the source: `file:line`, tool output, exact quote.
4. If the claim **can** be fully sourced, states that it is verified and gives the citation chain.
5. If the claim **cannot** be fully sourced, states precisely:
   - What is missing.
   - What would be needed to verify it (e.g., "read `commands/prep.md` line 48").

The output is a derivation proof, not a judgment call. Either the claim follows from evidence in the session, or it does not.

## When to invoke it

Use `/verify` when the stakes of a specific claim are high enough to warrant re-derivation from sources:

- A specific file path, line number, config field name, or function behaviour was cited in a recommendation and you want to confirm the citation is accurate.
- Something was asserted with confidence earlier in the session and subsequent context makes you doubt it.
- After `/notchecked` identifies a gap: once you know which claim was unverified, `/verify` is the tool to check it.
- Before acting on a claim that, if wrong, would cause significant rework (e.g., "this field is safe to rename" or "this hook fires on Stop, not PreToolUse").

It is most useful for **narrow, specific claims** — a line number, a field name, a function signature. For broad assumption audits, use `/assumptions`. For auditing a whole response's claim set, use `/notchecked`.

## How it relates to the discipline loop

`/verify` is the **targeted manual counterpart** to what [[check_verify_loop]] enforces automatically — and the sharpest tool in the introspection set for confirming individual claims. (inferred)

[[check_verify_loop]] enforces that items in a `## Did Not Verify` section are either resolved or tagged `(unverifiable: <reason>)`. It operates at the response level and on the DNV section.

`/verify` operates on **a single named claim** and demands a full derivation. It does not enforce anything automatically; it runs when the user explicitly requests it. This makes it advisory — but when used, it produces the highest standard of evidence: a step-by-step citation chain, or an honest statement that the chain is incomplete.

The natural sequence: run `/notchecked` to find which claims lacked evidence, then run `/verify <claim>` on the ones that matter most. This two-step flow closes the gap between "confidence was expressed" and "confidence was earned."

No existing hook enforces claim-level re-derivation automatically — that level of checking is too context-dependent to mechanise. `/verify` is the human-in-the-loop mechanism for it. See [[discipline-loop]] and [[enforcement-model]].

## Config fields read

None. This command reads no `workflow.config.yaml` fields.

## Scripts invoked

None. This is a pure prose instruction to Claude — no shell scripts are called.

## Preconditions

- A specific claim to verify must be passed as an argument. Without `$ARGUMENTS`, the command has no claim to derive.
- The verification is bounded to sources in the current session. Claims that require reading files not yet read in this session will require tool calls to satisfy; the command will name what is needed.

## See also

- [[discipline-loop]] — the discipline framework: confidence labels, DNV sections, advisory commands, mechanical hooks
- [[enforcement-model]] — why this command is advisory and hooks are mechanical
- [[check_verify_loop]] — the Stop hook that enforces explicit DNV tagging; mechanical counterpart at the response level
- [[notchecked]] — audit which claims in recent responses lacked evidence; use before `/verify` to find what to check
- [[assumptions]] — surface hidden assumptions across the full session
- [[disconfirm]] — argue against Claude's most recent recommendation
