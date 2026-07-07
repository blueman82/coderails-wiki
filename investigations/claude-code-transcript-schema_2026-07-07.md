---
title: "Claude Code transcript JSONL schema variants — 2026-07-07"
type: investigation
created: 2026-07-07
last_updated: 2026-07-07
sources: [sources/pr_27-39_workflow-audit-skill.md]
tags: [investigation, transcripts, jsonl, schema, privacy, workflow-audit]
---

# Claude Code transcript JSONL schema variants — 2026-07-07

> Filed: 2026-07-07. Point-in-time snapshot, based on record types observed and handled by [[workflow-audit]]'s `scan_transcripts.sh` and its test fixtures on `origin/main`. First wiki coverage of transcript file structure — the vault previously had zero pages on this topic.

## Question

What record types appear in a Claude Code session transcript (`~/.claude/projects/<slug>/<session-id>.jsonl`), which of them carry actual conversation content, and what schema gotchas does code that reads transcripts need to defend against? [[workflow-audit]]'s scan stage is the first code in this repo to systematically parse transcript files across a whole corpus, and its review process surfaced several non-obvious structural details worth recording independently of that skill.

## Record types observed

One JSON object per line. `.type` discriminates the record kind. Types confirmed from `scan_transcripts.sh`'s handling and the skill's test fixtures (`fixture-small.jsonl`, `fixture-edge.jsonl`, `fixture-sentinel.jsonl`, `e2e.test.sh`'s synthetic sessions):

| `.type` | Carries conversation? | Notes |
|---|---|---|
| `user` | yes | `.message.content` — see "bare string" gotcha below |
| `assistant` | yes | `.message.content[]` is where tool calls live |
| `system` | yes | system-level conversation turns |
| `last-prompt` | no | resume cache — see gotcha below |
| `mode` | no | session mode bookkeeping |
| `permission-mode` | no | permission-mode bookkeeping |
| `bridge-session` | no | cross-session bridging metadata |
| `attachment` | no | attached file/content metadata |
| `ai-title` | no | model-generated session title |
| `custom-title` | no | user-set session title |
| `agent-name` | no | agent naming metadata |
| `queue-operation` | no | queue bookkeeping (task/dispatch system) |
| `worktree-state` | no | worktree bookkeeping |

**Only `user`, `assistant`, and `system` carry conversation content.** Every other type is session bookkeeping/metadata and should be excluded from any content-scanning pass. `scan_transcripts.sh` enforces this by scoping its jq filter to `select(.type == "assistant")` before doing anything else — every other record type is excluded by construction, not by a separate filter step. `(verified against scan_transcripts.sh's JQ_FILTER)`

## Tool-use extraction locus

Tool invocations appear only inside `assistant` records:

```
.type == "assistant"
  → .message.content[]
    → select(.type == "tool_use")
      → {name, input}
```

`scan_transcripts.sh`'s exact filter: `select(.type == "assistant") | ($rec.message.content // []) | (if type == "array" then . else [] end)[]? | select(.type == "tool_use")`. `(verified)`

## Gotchas

- **`.message.content` is sometimes a bare string, not a list of parts.** A plain user turn with no tool use or attachments can have `content` as a raw string rather than an array of content-block objects. Code that assumes array shape without a type guard will crash or silently misparse. `scan_transcripts.sh`'s defence: `(if type == "array" then . else [] end)` — coerces a non-array to an empty list rather than erroring. The e2e test fixture explicitly includes `{"type":"user","message":{"role":"user","content":"a bare string content field"}}` to pin this. `(verified)`
- **`last-prompt` is a resume cache, not a conversation record — and it bumps file `mtime` without carrying its own `timestamp` field.** This means **file `mtime` must never be used as a recency signal** for "when did this session last have activity" — a session can look freshly touched purely because it was resumed, with no new conversation. `scan_transcripts.sh`'s `--last-sessions` and `--days` filtering both rank by the **latest in-file message `timestamp`** (scoped to `user`/`assistant` records only), explicitly never by mtime. The e2e fixture pins this with a `last-prompt` record containing the comment "resume cache text that must never be read." `(verified)`
- **MCP tool names are long and dotted**, e.g. `mcp__claude-in-chrome__navigate`, `mcp__claude_ai_Gmail__create_draft`. Any code keying behaviour off tool name prefixes/patterns should account for this naming shape rather than assuming short bare names like `Bash` or `Read`. `(inferred from tool names visible in this session's own available-tools list; not directly asserted by workflow-audit's code, but consistent with the `tool: .name` field it extracts verbatim)`
- **Non-string input values must be type-guarded before string operations.** A critical finding from workflow-audit's own WU1 code review: `.input.command`, `.input.skill`, and `.input.subagent_type` are not guaranteed to be strings (a malformed or unusual tool call could carry a non-string value in that field). `scan_transcripts.sh`'s head-extraction filter guards every one of these with `if type == "string" then . else "" end` before attempting token-splitting or concatenation — an ungated version would throw a jq type error and, per the file's own error-isolation contract, corrupt the whole line's output rather than degrading gracefully. `(verified against scan_transcripts.sh's JQ_FILTER — the three `if .name == "Bash"/"Skill"/"Agent"` branches each wrap their input access in this exact guard)`
- **Malformed lines should be isolated per-line, not per-file.** A single corrupt JSON line in a transcript should not discard every valid record in that session. `scan_transcripts.sh` parses each line independently via `jq -R -n '[ inputs | fromjson? // empty ]'`, so one bad line becomes `empty` and is dropped while the rest of the file's lines still parse; a corrupt-line count is detected by comparing parsed-line count against non-blank raw-line count, and surfaced as `jq_parse_error:<file>` on stderr rather than silently swallowed. `cluster_ngrams.sh` applies the identical per-line isolation pattern to its own stdin stream (`jq_parse_error:<line-no>`). `(verified)`
- **Session self-exclusion is done by filename, not content.** A scan run from within a live session must not read its own transcript file mid-write. `scan_transcripts.sh` compares `$CLAUDE_CODE_SESSION_ID` against each candidate file's basename and skips a match, emitting `skipped_own_session:<file>` on stderr. `(verified)`

## Privacy implication

Because transcript content can include arbitrary tool arguments (API keys, file contents, prose), any code that reads across a whole transcript corpus needs an explicit, structural allowlist of what may leave the transcript — not a denylist/redaction pass over free text. [[workflow-audit]]'s three-field whitelist (Bash first-two-tokens, Skill name, Agent `subagent_type`) is the concrete precedent in this repo; its e2e test's sentinel negative control (a fake secret planted in a Bash command's arguments, proven present in the raw fixture, then asserted absent from all pipeline output except the whitelisted head) is the verification pattern worth reusing for any future transcript-reading code.

## See also

- [[workflow-audit]] — the skill whose scan stage this schema catalogue documents
- [[pr_27-39_workflow-audit-skill]] — the ingest source page
- [[skill-testing-state_2026-06-26]] — prior investigation using the same "verify wiring + schema, not just existence" discipline
