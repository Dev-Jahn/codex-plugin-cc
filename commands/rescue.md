---
description: Delegate investigation, an explicit fix request, or follow-up rescue work to the Codex rescue subagent
argument-hint: "[--background|--wait] [--resume|--fresh] [--model <model|spark>] [--effort <none|minimal|low|medium|high|xhigh>] [what Codex should investigate, solve, or continue]"
allowed-tools: Bash(node *codex-companion.mjs* task-resume-candidate*), AskUserQuestion, Agent
---

<hard_rules importance="critical">
The user invoked `/codex:rescue`. That IS the explicit decision to delegate the work to Codex. Do NOT second-guess that decision.

FORBIDDEN behaviors (no exceptions, no matter how trivial the request looks):
- Saying "이 요청은 Codex 작업이 아니니 직접 처리" / "I'll just do this directly" / "이건 너무 단순해서" / "this is simpler than a Codex task" / "단순한 shell 명령이므로" / any equivalent reasoning.
- Substituting your own work for the codex-rescue subagent: do NOT execute the user's request via Bash, Read, Edit, Grep, Glob, etc. yourself. The only Bash you may issue here is the `task-resume-candidate --json` helper.
- Skipping the `Agent` call because the request looks short, trivial, one-line, or "obvious" (e.g. `echo "hello world!"`, a single-file read, a tiny calculation).
- Falling back to your own judgment if Codex's output is short or looks "wrong" — return Codex's stdout verbatim and let the user judge.

The user may explicitly be testing the Codex pipeline (sandbox-bypass sanity checks, end-to-end verification, long-running delegation). Short-circuiting defeats the entire purpose of invoking this command.

REQUIRED PATH (the only allowed flow):
1. (Optional) `task-resume-candidate --json` helper, only if `--resume`/`--fresh` are NOT in the user's arguments.
2. (Optional) `AskUserQuestion` for continue-vs-new-thread, only if the helper reports `available: true`.
3. `Agent(subagent_type: "codex:codex-rescue", prompt: <user request verbatim, with `--resume`/`--fresh` routing flag attached if applicable>)`.
4. Return the Agent's stdout verbatim.
</hard_rules>

Invoke the `codex:codex-rescue` subagent via the `Agent` tool (`subagent_type: "codex:codex-rescue"`), forwarding the raw user request as the prompt.
`codex:codex-rescue` is a subagent, not a skill — do not call `Skill(codex:codex-rescue)` (no such skill) or `Skill(codex:rescue)` (that re-enters this command and hangs the session). The command runs inline so the `Agent` tool stays in scope; forked general-purpose subagents do not expose it.
The final user-visible response must be Codex's output verbatim.

Raw user request:
$ARGUMENTS

Execution mode:

- If the request includes `--background`, run the `codex:codex-rescue` subagent in the background.
- If the request includes `--wait`, run the `codex:codex-rescue` subagent in the foreground.
- If neither flag is present, default to foreground.
- `--background` and `--wait` are execution flags for Claude Code. Do not forward them to `task`, and do not treat them as part of the natural-language task text.
- `--model` and `--effort` are runtime-selection flags. Preserve them for the forwarded `task` call, but do not treat them as part of the natural-language task text.
- If the request includes `--resume`, do not ask whether to continue. The user already chose.
- If the request includes `--fresh`, do not ask whether to continue. The user already chose.
- Otherwise, before starting Codex, check for a resumable rescue thread from this Claude session by running:

```bash
node "${CLAUDE_PLUGIN_ROOT}/scripts/codex-companion.mjs" task-resume-candidate --json
```

- If that helper reports `available: true`, use `AskUserQuestion` exactly once to ask whether to continue the current Codex thread or start a new one.
- The two choices must be:
  - `Continue current Codex thread`
  - `Start a new Codex thread`
- If the user is clearly giving a follow-up instruction such as "continue", "keep going", "resume", "apply the top fix", or "dig deeper", put `Continue current Codex thread (Recommended)` first.
- Otherwise put `Start a new Codex thread (Recommended)` first.
- If the user chooses continue, add `--resume` before routing to the subagent.
- If the user chooses a new thread, add `--fresh` before routing to the subagent.
- If the helper reports `available: false`, do not ask. Route normally.

Operating rules:

- The subagent is a thin forwarder only. It should use one `Bash` call to invoke `node "${CLAUDE_PLUGIN_ROOT}/scripts/codex-companion.mjs" task ...` and return that command's stdout as-is.
- Return the Codex companion stdout verbatim to the user.
- Do not paraphrase, summarize, rewrite, or add commentary before or after it.
- Do not ask the subagent to inspect files, monitor progress, poll `/codex:status`, fetch `/codex:result`, call `/codex:cancel`, summarize output, or do follow-up work of its own.
- Leave `--effort` unset unless the user explicitly asks for a specific reasoning effort.
- Leave the model unset unless the user explicitly asks for one. If they ask for `spark`, map it to `gpt-5.3-codex-spark`.
- Leave `--resume` and `--fresh` in the forwarded request. The subagent handles that routing when it builds the `task` command.
- If the helper reports that Codex is missing or unauthenticated, stop and tell the user to run `/codex:setup`.
- If the user did not supply a request, ask what Codex should investigate or fix.
