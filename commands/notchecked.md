---
title: "/coderails:notchecked"
type: command
created: 2026-06-25
last_updated: 2026-06-25
sources: [commands/notchecked.md]
tags: [command, discipline, introspection, did-not-verify, advisory]
---

# /coderails:notchecked

Review recent responses and list every non-trivial claim made without actual verification — the gap between what was asserted and what was confirmed by evidence.

## Invocation

```
/coderails:notchecked
```

No arguments. Operates on the current session's recent responses.

## What it does

1. Reviews Claude's recent responses in the current conversation.
2. Lists every non-trivial claim that was **not** actually verified — i.e., things written with confidence that were not backed by a tool result, file read, or user-provided fact.
3. For each unverified claim, states:
   - The claim, in one sentence.
   - Why it was not verified (cost, oversight, assumed obvious, etc.).
   - What verification would look like now.
4. Is deliberately ruthless — does not defend the claims, only surfaces the gaps.
5. Does not attempt to verify the claims in the same turn. It is an audit, not a fix.

## When to invoke it

Use `/notchecked` when you want to audit confidence before acting on Claude's recent output:

- Before implementing something Claude described as "straightforward" without reading the relevant files.
- When a response contained a lot of assertions and you want to know which ones rest on evidence vs. recall.
- Mid-session when something has gone wrong and you want to trace whether a wrong assumption caused it.
- Before handing off session output to another agent or to a human reviewer.

The command is most useful when responses have been dense with claims — architecture explanations, diagnoses, "this will work because..." reasoning. Short purely-operational responses (running a command, reading a file) rarely need it.

## How it relates to the discipline loop

`/notchecked` is the **manual, advisory counterpart** to what [[check_verify_loop]] enforces automatically, and to the `## Did Not Verify` discipline in [[discipline-loop]]. (inferred)

[[check_verify_loop]] is a Stop hook: it fires after every response where files were edited and blocks (exit 2) if any `## Did Not Verify` bullet is left untagged. It enforces that deferrals are *explicit*.

`/notchecked` does something complementary: it audits **claims made in the response body itself** — not just items flagged in a DNV section, but assertions embedded in prose that were never tagged or audited. It runs on demand, not automatically.

The gap between them:
- The hook polices the DNV section — what Claude *acknowledged* as unchecked.
- The command surfaces what Claude *did not acknowledge* as unchecked — the unguarded assertions.

These are two different failure modes. `/notchecked` closes the second one, which the hook cannot catch. Together they form a more complete accountability layer. See [[discipline-loop]] and [[enforcement-model]].

## Config fields read

None. This command reads no `workflow.config.yaml` fields.

## Scripts invoked

None. This is a pure prose instruction to Claude — no shell scripts are called.

## Preconditions

There must be recent responses in the session with non-trivial claims to audit. In a fresh session with only tool results and no assertions, the output is empty.

## See also

- [[discipline-loop]] — the discipline framework: confidence labels, DNV sections, advisory commands, mechanical hooks
- [[enforcement-model]] — why this command is advisory and [[check_verify_loop]] is mechanical
- [[check_verify_loop]] — the Stop hook that enforces explicit DNV tagging automatically
- [[assumptions]] — surface hidden assumptions before or during a task
- [[verify]] — re-derive a specific claim from sources only (the verification step after `/notchecked` identifies the gap)
- [[disconfirm]] — argue against Claude's most recent recommendation
