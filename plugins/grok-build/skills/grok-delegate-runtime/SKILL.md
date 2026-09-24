---
name: grok-delegate-runtime
description: Internal helper contract for calling the grok-bridge runtime from Claude Code
user-invocable: false
---

# Grok Build Delegate Runtime

Use this skill only inside the `grok-build:grok-delegate` subagent.

Primary helper:
- `node "${CLAUDE_PLUGIN_ROOT}/scripts/grok-bridge.mjs" run "<raw arguments>"`

Execution rules:
- The delegate subagent is a forwarder, not an orchestrator. Its only job is to invoke `run` once and return that stdout unchanged.
- Prefer the helper over hand-rolled `git`, direct Grok CLI strings, or any other Bash activity.
- Do not call `check`, `review`, `critique`, `runs`, `show`, or `stop` from `grok-build:grok-delegate`.
- Use `run` for every delegate request, including diagnosis, planning, research, and explicit fix requests.
- Leave `--effort` unset unless the user explicitly requests a specific effort.
- Leave model unset by default. Add `--model` only when the user explicitly asks for one.
- Default to a write-capable Grok run by adding `--write` unless the user explicitly asks for read-only behavior or only wants review, diagnosis, or research without edits.

Command selection:
- Use exactly one `run` invocation per delegate handoff.
- If the forwarded request includes `--background` or `--wait`, treat that as Claude-side execution control only for short enqueue semantics. Prefer bridge `--background` for long work so the run records `bridgePid` and `agentPid`. When forwarding long work yourself, pass `--background` to `run` when the user chose background mode. Strip Claude-only framing that is not a bridge flag, and do not treat those tokens as part of the natural-language task text.
- If the forwarded request includes `--model`, pass it through to `run`.
- If the forwarded request includes `--effort`, pass it through to `run`.
- If the forwarded request includes `--resume`, strip that token from the task text and add `--resume-last`.
- If the forwarded request includes `--fresh`, strip that token from the task text and do not add `--resume-last`.
- `--resume`: always use `run --resume-last`, even if the request text is ambiguous.
- `--fresh`: always use a fresh `run`, even if the request sounds like a follow-up.
- `--effort`: accepted values are `low`, `medium`, `high`, `xhigh`.
- Use `run --resume-last` only when the request explicitly asks to continue, extend, or keep working on Grok's own prior task in this thread (using one of: "continue", "keep going", "resume", "apply the top fix", "dig deeper", or an equivalent unambiguous continuation instruction) or when the request itself contains an explicit `--resume` token. A request to review, re-review, or assess an updated/new diff or piece of content is NEVER a continuation request on its own, even if it mentions or contrasts with a prior review — default to a fresh run (no `run --resume-last`) for those. When genuinely ambiguous, prefer a fresh run: a fresh run wastes some redundant context; an incorrectly resumed run silently reuses stale reasoning and produces a plausible-looking but wrong answer.

Safety rules:
- Default to write-capable Grok work in `grok-build:grok-delegate` unless the user explicitly asks for read-only behavior.
- Preserve the user's task text as-is apart from stripping routing flags.
- Do not inspect the repository, read files, grep, monitor progress, poll status, fetch results, stop runs, summarize output, or do any follow-up work of your own.
- Return the stdout of the `run` command exactly as-is.
- If the Bash call fails or Grok cannot be invoked, return nothing.
