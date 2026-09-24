---
description: Delegate investigation, an explicit fix request, or follow-up work to the Grok Build delegate subagent
argument-hint: "[--background|--wait] [--resume|--fresh] [--model <model>] [--effort <low|medium|high|xhigh>] [what Grok should investigate, solve, or continue]"
allowed-tools: Bash(node:*), AskUserQuestion, Agent
---

Invoke the `grok-build:grok-delegate` subagent via the `Agent` tool (`subagent_type: "grok-build:grok-delegate"`), forwarding the raw user request as the prompt.
`grok-build:grok-delegate` is a subagent, not a skill — do not call `Skill(grok-build:grok-delegate)` (no such skill) or `Skill(grok-build:delegate)` (that re-enters this command and hangs the session). The command runs inline so the `Agent` tool stays in scope; forked general-purpose subagents do not expose it.
The final user-visible response must be Grok's output verbatim.

Raw user request:
$ARGUMENTS

Execution mode:

- If the request includes `--background`, run the `grok-build:grok-delegate` subagent in the background.
- If the request includes `--wait`, run the `grok-build:grok-delegate` subagent in the foreground.
- If neither flag is present, default to foreground.
- Prefer bridge `--background` for long or open-ended work so the run records both `bridgePid` (Node worker) and `agentPid` (grok child).
- `--background` and `--wait` are execution flags for Claude Code. Do not forward them to `run`, and do not treat them as part of the natural-language task text.
- `--model` and `--effort` are runtime-selection flags. Preserve them for the forwarded `run` call, but do not treat them as part of the natural-language task text.
- If the request includes `--resume`, do not ask whether to continue. The user already chose.
- If the request includes `--fresh`, do not ask whether to continue. The user already chose.
- Otherwise, before starting Grok, check for a resumable delegate thread from this Claude session by running:

```bash
node "${CLAUDE_PLUGIN_ROOT}/scripts/grok-bridge.mjs" run-resume-candidate --json
```

- If that helper reports `available: true`, use `AskUserQuestion` exactly once to ask whether to continue the current Grok thread or start a new one.
- The two choices must be:
  - `Continue current Grok thread`
  - `Start a new Grok thread`
- Put `Continue current Grok thread (Recommended)` first only when the request explicitly asks to continue, extend, or keep working on Grok's own prior task in this thread (using one of: "continue", "keep going", "resume", "apply the top fix", "dig deeper", or an equivalent unambiguous continuation instruction). An explicit `--resume` token already means the user chose continue and skips this prompt. A request to review, re-review, or assess an updated/new diff or piece of content is NEVER a continuation request on its own, even if it mentions or contrasts with a prior review — put `Start a new Grok thread (Recommended)` first for those. When genuinely ambiguous, put `Start a new Grok thread (Recommended)` first: a fresh thread wastes some redundant context; an incorrectly continued thread silently reuses stale reasoning and produces a plausible-looking but wrong answer.
- Otherwise put `Start a new Grok thread (Recommended)` first.
- If the user chooses continue, add `--resume` before routing to the subagent.
- If the user chooses a new thread, add `--fresh` before routing to the subagent.
- If the helper reports `available: false`, do not ask. Route normally.

Operating rules:

- The subagent is a thin forwarder only. It should use one `Bash` call to invoke `node "${CLAUDE_PLUGIN_ROOT}/scripts/grok-bridge.mjs" run ...` and return that command's stdout as-is.
- Return the Grok bridge stdout verbatim to the user.
- Do not paraphrase, summarize, rewrite, or add commentary before or after it.
- Do not ask the subagent to inspect files, monitor progress, poll `/grok-build:runs`, fetch `/grok-build:show`, call `/grok-build:stop`, summarize output, or do follow-up work of its own.
- Leave `--effort` unset unless the user explicitly asks for a specific reasoning effort.
- Leave the model unset unless the user explicitly asks for one.
- Leave `--resume` and `--fresh` in the forwarded request. The subagent handles that routing when it builds the `run` command.
- If the helper reports that Grok is missing or unauthenticated, stop and tell the user to run `/grok-build:check`.
- If the user did not supply a request, ask what Grok should investigate or fix.
