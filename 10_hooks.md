# Chapter 10 -- Hooks

**TL;DR.** Hooks are user-defined handlers that fire deterministically at points in Claude Code's lifecycle. They are the one mechanism in the system that does not wait on the model choosing to do the right thing. The harness runs them whether the model cooperates or not. Reach for a hook when a rule has to hold every time: block `rm -rf`, auto-format every edited file, refuse writes to `.env`, inject project state at session start, append an immutable audit record of every tool call. The mental model is three layers, event then matcher then handler, plus a control protocol your handler uses to tell the harness what happens next. Exit codes cover the simple case. JSON covers `allow`/`deny`/`ask`, input rewriting, and context injection. When a decision needs judgment instead of a fixed rule, a hook can call a model through the `prompt` and `agent` types, but the safety-critical guardrails belong in deterministic command hooks. Claude Code ships near-daily and the event roster grows often, so confirm the exact event list and any version-pinned field against the live [Hooks reference](https://code.claude.com/docs/en/hooks) and [changelog](https://code.claude.com/docs/en/changelog) before you teach or deploy.

---

## 1. Why hooks exist: deterministic control over a probabilistic agent

Everything else in Claude Code is advice. The `CLAUDE.md` file, skills, subagents, the prompt itself. You hand the model context and instructions and it decides what to do with them, and that is the right default, because the model's flexibility is the entire reason you brought it in. Some rules, though, cannot ride on a probability. Never force-push to `main`. Always run the formatter. Log every shell command for the auditor. Those have to hold on the model's worst day, not its average one.

That gap is what hooks fill. Anthropic's own framing is direct: *"Unlike CLAUDE.md instructions which are advisory, hooks are deterministic and guarantee the action happens."* [^1] And from the guide: *"Hooks are user-defined shell commands that execute at specific points in Claude Code's lifecycle. They provide deterministic control over Claude Code's behavior, ensuring certain actions always happen rather than relying on the LLM to choose to run them."* [^2]

The senior-engineer heuristic falls right out of that. Use a hook for actions that must happen every time with zero exceptions. [^3] When you catch yourself writing "IMPORTANT: ALWAYS..." in `CLAUDE.md` and the model still skips it sometimes, that instruction wants to be a hook.

It is also the cleanest decision boundary in the whole extension surface.

| You want... | Reach for |
|-----------|-----------|
| A rule that must always hold (safety, compliance, formatting) | **Hook** (deterministic command) |
| A judgment call applied consistently but not absolutely | **Prompt / agent hook** (model-evaluated) |
| Reusable knowledge or a procedure the model applies itself | **Skill** (see Ch. 07) |
| Access to a live external system | **MCP server** (see Ch. 09) |
| Static project conventions | **CLAUDE.md / `.claude/rules/`** (see Ch. 03) |

There is a corollary worth holding onto, because the asymmetry is the whole security story in miniature. Hooks can tighten restrictions but they cannot loosen them. A `PreToolUse` hook that returns `deny` blocks a tool even in `bypassPermissions` mode or under `--dangerously-skip-permissions`, which means you can enforce policy a user cannot escape by flipping a permission mode. [^4] The reverse fails. A hook that returns `allow` does not override a deny rule from settings. The reference states it plainly: *"Hooks can tighten restrictions but not loosen them past what permission rules allow."* Deny rules from any scope, managed and enterprise settings included, always beat a hook's approval. [^4]

---

## 2. The lifecycle: what events fire, and when

A hook is bound to a lifecycle event. When that event fires, all matching hooks run in parallel, identical handlers are deduplicated automatically, and the harness merges their outputs. [^5] The reference groups events into three cadences: once per session (`SessionStart`, `SessionEnd`), once per turn (`UserPromptSubmit`, `Stop`, `StopFailure`), and on every tool call inside the agentic loop (`PreToolUse`, `PostToolUse`). [^6] The full roster runs to roughly thirty distinct events. You will lean on a handful constantly and meet the rest rarely, so the tables below group them to let you find the right one fast.

> **Version sensitivity, read this.** New events and fields land frequently. To take confirmed examples from the changelog: `terminalSequence` arrived in v2.1.141 (2026-05-13), and the `MessageDisplay` event arrived in v2.1.152 (2026-05-27). [^7] Treat any single event below as "may not exist on an older binary" and confirm against the [reference](https://code.claude.com/docs/en/hooks) and changelog before relying on it.

### 2.1 The events you'll actually use

| Event | Fires | Can block? | Canonical use |
|-------|-------|:---------:|---------------|
| `PreToolUse` | Before a tool call executes | **Yes** | Guardrails: deny destructive commands, protect files, rewrite args |
| `PostToolUse` | After a tool call **succeeds** | No (tool already ran) | Auto-format, audit log, run linters/tests on the changed file |
| `UserPromptSubmit` | When you submit a prompt, before Claude sees it | **Yes** (rejects prompt) | Inject context, validate/redact input, block secret-leaking prompts |
| `SessionStart` | Session begins or resumes | No | Inject project state / ticket / recent commits into context |
| `Stop` | Claude finishes responding | **Yes** (forces more work) | Gate the turn on a passing test/build before Claude is allowed to stop |
| `Notification` | Claude sends a notification (waiting for you) | No | Desktop / Slack alert when Claude needs input |
| `SessionEnd` | Session terminates | No | Cleanup scratch files, flush logs |

[^8]

### 2.2 The fuller roster (grouped by concern)

The reference is the live source for this list. The grouping here is to give you the shape.

**Per-turn and prompt**

- `UserPromptSubmit`: prompt submitted, before processing. Blockable; exit 2 erases the prompt.
- `UserPromptExpansion`: a typed command expands into a prompt before it reaches Claude. Can block the expansion. Matches on the command name.
- `Stop`: Claude finishes responding. Blockable. Fires on every turn end, not just task completion, and does not fire on user interrupts. [^9]
- `StopFailure`: the turn ends due to an API error. Output and exit code are ignored. The matcher filters by error type (`rate_limit`, `overloaded`, `authentication_failed`, `billing_error`, `server_error`, `max_output_tokens`, and others).
- `MessageDisplay`: while assistant text displays. `hookSpecificOutput.displayContent` replaces what the user sees without changing the transcript.

**Agentic loop and tools**

- `PreToolUse`: before a tool call. Blockable, and can rewrite input.
- `PostToolUse`: after a tool succeeds.
- `PostToolUseFailure`: after a tool fails.
- `PostToolBatch`: after a full batch of parallel tool calls resolves, before the next model call. No matcher, fires exactly once with the full batch. The right place to inject context that depends on the set of tools that ran. [^10]
- `PermissionRequest`: a permission dialog is about to appear. Auto-approve or deny. Does not fire in non-interactive `-p` mode, so use `PreToolUse` for automated permission decisions there. [^9]
- `PermissionDenied`: a call was denied by the auto-mode classifier. Return `{retry: true}` to tell the model it may retry.
- `PreCompact` / `PostCompact`: before and after context compaction. Matcher is `manual` or `auto`.

**Session and setup**

- `SessionStart` (matcher `startup` | `resume` | `clear` | `compact`), `SessionEnd` (matcher `clear` | `resume` | `logout` | `prompt_input_exit` | `bypass_permissions_disabled` | `other`), `Setup` (`--init-only`, or `--init` / `--maintenance` in `-p` mode; matcher `init` | `maintenance`).

**Subagents, teams, tasks**

- `SubagentStart` / `SubagentStop` (matcher is the agent type, e.g. `Explore`, `Plan`, `general-purpose`, or a custom name), `TeammateIdle`, `TaskCreated`, `TaskCompleted`.

**Environment, config, files**

- `CwdChanged`: working directory changed, e.g. after a `cd`. The canonical fix for reloading `direnv`-style env into Claude's Bash tool.
- `FileChanged`: a watched file changed on disk. The `matcher` is a `|`-separated list of literal filenames to watch, split into filenames rather than evaluated as a regex.
- `ConfigChange`: a settings or skills file changed during the session (matcher `user_settings` | `project_settings` | `local_settings` | `policy_settings` | `skills`). Blockable except for `policy_settings`, and a natural compliance tripwire.
- `InstructionsLoaded`: a `CLAUDE.md` or `.claude/rules/*.md` file is loaded (matcher `session_start` | `nested_traversal` | `path_glob_match` | `include` | `compact`).
- `WorktreeCreate` / `WorktreeRemove`: replace default git worktree behavior.

**MCP elicitation**

- `Elicitation` / `ElicitationResult`: an MCP server requests user input mid-tool-call. Matcher is the MCP server name.

[^11] [^12]

> **Teaching note.** Do not memorize all of these. Internalize the seven in 2.1 and the shape of the rest. When a student asks "can I hook X," the answer is almost always "yes, check the roster," and the roster is the live doc, not this chapter.

---

## 3. Configuration: event -> matcher -> handler

Hooks live in a `hooks` block inside a settings file. The structure is three levels of nesting. The event name maps to an array of matcher groups, each of which has a `matcher` and an array of `hooks` (the handlers). [^13]

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/block-rm-rf.sh",
            "timeout": 5
          }
        ]
      }
    ]
  }
}
```

Read it inside-out: on `PreToolUse`, for tool calls matching `Bash`, run this command handler.

### 3.1 Where settings live, and what wins

| Location | Scope | Shareable |
|----------|-------|-----------|
| `~/.claude/settings.json` | All your projects | No (your machine) |
| `.claude/settings.json` | Single project | **Yes** -- commit it |
| `.claude/settings.local.json` | Single project | No (gitignored) |
| Managed policy settings | Org-wide | Yes (admin-controlled) |
| Plugin `hooks/hooks.json` | When plugin enabled | Yes (bundled) |
| Skill/agent frontmatter | While that component is active | Yes (in the component) |

[^14]

Two operational facts trip seniors up. The first is that all applicable scopes contribute. Hooks from managed, user, project, and local settings all fire. They do not override each other the way a single key would. Every matching hook runs. The second is that edits are hot-reloaded. Editing a settings file mid-session is normally picked up by the file watcher within a few seconds, so when `/hooks` shows nothing after you edit, the JSON is probably invalid (no trailing commas, no comments) or the watcher missed it, and restarting the session forces a reload. [^15]

`/hooks` opens a read-only browser of every configured hook, grouped by event, showing the matcher, type, source file, and command. It is the first thing to run when a hook "is not firing." To change a hook you edit the JSON, or ask Claude to. *"Write a hook that runs eslint after every file edit"* works well. [^1]

The kill switch is `"disableAllHooks": true` in a settings file. Managed hooks keep running unless `disableAllHooks` is also set at the managed level. [^14] At the enterprise tier there is one more lever: administrators can set `allowManagedHooksOnly` to block user, project, and plugin hooks entirely, with force-enabled plugin hooks exempted so a vetted set can still ship through an org marketplace. [^16]

### 3.2 Matchers, the filter

Without a matcher, a hook fires on every occurrence of its event. The `matcher` narrows that. What it matches against depends on the event: tool name for tool events, session-start reason for `SessionStart`, notification type for `Notification`, and so on through 2.2.

The matcher's syntax decides how it gets evaluated, and this is the rule to commit to memory. [^17]

| Matcher value | Evaluated as | Example |
|---------------|--------------|---------|
| `"*"`, `""`, or omitted | Match all | fires on every occurrence |
| Only letters, digits, `_`, `\|` | Exact string, `\|` = alternation | `Bash`, `Edit\|Write` |
| Anything else | **JavaScript regex** | `^Notebook`, `mcp__.*__write.*` |

The classic gotcha lives in that middle row. `mcp__memory` contains only letters and underscores, so it gets treated as an exact tool name and matches nothing, because no tool is literally named that. To match a server's tools you need a regex: `mcp__memory__.*`. MCP tools follow `mcp__<server>__<tool>`, so `mcp__.*__write.*` matches any write-shaped tool across all servers. Matchers are case-sensitive. [^18]

### 3.3 The `if` field, filter by argument, not just tool name

`matcher` filters at the group level by tool name. The `if` field filters at the handler level using permission-rule syntax, so you can scope to specific arguments and the hook process only spawns when the call actually matches. It requires Claude Code v2.1.85 or later; earlier versions ignore it and run the hook on every matched call. [^19]

```json
{
  "matcher": "Bash",
  "hooks": [
    {
      "type": "command",
      "if": "Bash(git *)",
      "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/check-git-policy.sh"
    }
  ]
}
```

`if` accepts the same patterns as permission rules (`Bash(git *)`, `Edit(*.ts)`, and so on) and works only on tool events (`PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `PermissionRequest`, `PermissionDenied`). Adding it elsewhere prevents the hook from running.

For Bash the matcher is parser-aware, which sounds reassuring until you read the last sentence. It strips leading assignments, checks each subcommand after `&&` and `;`, and inspects commands inside `$()` and backticks. So `Bash(git *)` fires on `FOO=bar git push`, on `npm test && git push`, and on `echo $(git log)`. It fails open, though. When a command cannot be parsed, the hook runs anyway. So treat `if` as best-effort filtering, never a security boundary. The reference is explicit: *"Because the filter is best-effort, use the permission system rather than a hook to enforce a hard allow or deny."* For a hard allow or deny, use the permission system or have the handler itself re-validate. [^19]

### 3.4 Handler types

Most hooks are `command`, but five types exist. [^20]

- **`command`** runs a shell command. The default, covered throughout this chapter.
- **`http`** POSTs the event JSON to a URL, and the response body carries the decision in the same JSON format command hooks use. HTTP status codes alone cannot block. Good for a shared team audit service. [^21]
- **`mcp_tool`** calls a tool on an already-connected MCP server, with `input` templated from event fields like `${tool_input.file_path}`. Available on every event once servers connect, though `SessionStart` and `Setup` fire before servers finish connecting and should expect a "not connected" error on first run. [^22]
- **`prompt`** is a single-turn model call, Haiku by default, that returns a yes/no decision. The judgment-call escape hatch (section 6).
- **`agent`** is a multi-turn subagent with tool access that can read files and run commands before deciding. Experimental, so prefer command hooks for production. [^23]

Common handler fields: `type` (required); `timeout` in seconds, defaulting to 600 for `command`/`http`/`mcp_tool`, 30 for `prompt`, 60 for `agent`, with `UserPromptSubmit` lowering them to 30 and `MessageDisplay` to 10; `if` (tool events only); `statusMessage` (custom spinner text); `once` (run once per session then remove, honored only in skill frontmatter and ignored in settings and agent frontmatter); `async` / `asyncRewake` (background execution, section 5.7). For command hooks specifically, adding an `args` array switches to exec form, spawned directly with no shell and no quoting headaches. [^24]

### 3.5 Path placeholders and environment

Never hard-code paths. Use these so a hook works regardless of cwd. [^25]

- `${CLAUDE_PROJECT_DIR}` is the project root.
- `${CLAUDE_PLUGIN_ROOT}` / `${CLAUDE_PLUGIN_DATA}` are the plugin install dir and persistent data dir.

Hooks inherit Claude Code's environment and also see `CLAUDE_PROJECT_DIR`, `CLAUDE_EFFORT` (the active effort level), `CLAUDE_ENV_FILE` (a path your hook can write `export KEY=value` lines to, which Claude Code runs as a Bash preamble before each command, central to the `direnv` pattern, available on `SessionStart`, `Setup`, `CwdChanged`, and `FileChanged`), and `CLAUDE_CODE_REMOTE` (`"true"` in the web environment, unset locally). HTTP-hook headers interpolate `$VAR` and `${VAR}`, but only variables you list in `allowedEnvVars` resolve. Everything else stays empty, so a leaked header template cannot exfiltrate arbitrary env. [^26]

---

## 4. The control protocol: how a handler tells the harness what to do

A handler reads the event on stdin, does its work, and answers with stdout, stderr, and an exit code. There are two tiers. The exit-code protocol is simple, block or stay silent. The JSON protocol is structured: allow, deny, ask, rewrite input, inject context. Do not mix them. The reference is blunt about it: *"Don't mix them: Claude Code ignores JSON when you exit 2."* [^27]

### 4.1 What the handler receives (stdin)

Every event delivers JSON on stdin, or as the POST body for HTTP hooks. Shared fields include `session_id`, `transcript_path`, `cwd`, `hook_event_name`, `permission_mode`, and `effort` (an object with a `level`). Inside a subagent the input also carries `agent_id` and `agent_type`. Tool events add `tool_name`, `tool_input`, and, on post-events, the tool response. `UserPromptSubmit` adds `prompt`. `SessionStart` adds `source`. `Stop` and `SubagentStop` add `stop_hook_active` (section 6.3). [^28]

A `PreToolUse` payload for a Bash call looks like this.

```json
{
  "session_id": "abc123",
  "cwd": "/Users/sarah/myproject",
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": { "command": "npm test" }
}
```

Parse with `jq` in shell (`jq -r '.tool_input.command'`), or read it in Python or Node if you would rather not depend on `jq`. [^29]

### 4.2 The exit-code protocol

The simplest output a handler can give is an exit code. [^30]

- **Exit `0`, success and no objection.** The action proceeds. For `PreToolUse` this does not approve the call, the normal permission flow still applies. For `UserPromptSubmit`, `UserPromptExpansion`, and `SessionStart`, anything written to stdout is added to Claude's context, which is the simplest possible context-injection mechanism.
- **Exit `2`, blocking error.** The action is blocked, and stderr is fed to Claude as feedback so it can adjust. Not every event is blockable. For `SessionStart`, `Setup`, `Notification`, and others, exit 2 just shows stderr to the user and execution continues. Check the per-event behavior in the reference.
- **Any other exit code, non-blocking error.** The action proceeds. The transcript shows a `<hook> hook error` notice plus the first stderr line, and the full stderr goes to the debug log.

> **The single most common hook bug** is using `exit 1` to block. Unix convention says non-zero means failure, but the harness only treats `exit 2` as a block. The reference warns: *"Claude Code treats exit code 1 as a non-blocking error and proceeds with the action, even though 1 is the conventional Unix failure code. If your hook is meant to enforce a policy, use exit 2."* [^31]

A minimal deny-by-exit-code, lifted from the guide.

```bash
#!/bin/bash
INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command')
if echo "$COMMAND" | grep -q "drop table"; then
  echo "Blocked: dropping tables is not allowed" >&2   # stderr -> Claude's feedback
  exit 2                                                # block
fi
exit 0                                                  # no decision; permission flow continues
```

[^30]

### 4.3 The JSON protocol, structured control

Exit `0` and print a JSON object to stdout for richer control. There are universal top-level fields and event-specific `hookSpecificOutput` fields.

The universal fields work on every event. [^32]

| Field | Default | Meaning |
|-------|---------|---------|
| `continue` | `true` | If `false`, Claude **stops entirely** after this hook |
| `stopReason` | -- | Message shown when `continue: false` |
| `suppressOutput` | `false` | Hide the hook's stdout from the transcript |
| `systemMessage` | -- | Warning or message shown to the **user**, not the model |
| `terminalSequence` | -- | A terminal escape sequence (restricted to OSC `0`/`1`/`2`/`9`/`99`/`777` and BEL) for notifications, window titles, and bells (v2.1.141+) |

The `PreToolUse` decision is the workhorse. Put the decision in `hookSpecificOutput`. [^27]

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "deny",
    "permissionDecisionReason": "Use rg instead of grep for better performance"
  }
}
```

The `permissionDecision` values:

- `"allow"` skips the interactive permission prompt. Deny and ask rules still apply, including enterprise managed deny lists, so a managed deny cannot be allowed away.
- `"deny"` cancels the call, and `permissionDecisionReason` goes back to Claude so it can adjust.
- `"ask"` shows the normal permission prompt.
- `"defer"` is non-interactive (`-p`) only. It exits the process with the call preserved so an Agent SDK wrapper can collect input and resume, and it only works when Claude makes a single tool call in the turn. [^33]

Other events use other shapes. `PostToolUse`, `Stop`, `SubagentStop`, `UserPromptSubmit`, `UserPromptExpansion`, `PostToolBatch`, `ConfigChange`, and `PreCompact` use a top-level `decision: "block"` plus `reason`. `PermissionRequest` uses `hookSpecificOutput.decision.behavior` (`allow` or `deny`). `MessageDisplay` uses `displayContent`. `SessionStart`, `Setup`, and `SubagentStart` are context-only (`additionalContext`, plus startup-only fields like `sessionTitle` and `reloadSkills`). Consult the reference's decision-control table per event. [^34]

### 4.4 `additionalContext`, inject text into the model's context

For `UserPromptSubmit` and the post-tool and session-start style events, instead of a `decision` you return `additionalContext` to add information the model reads as a system reminder. This is the structured equivalent of echoing to stdout on `SessionStart`.

```json
{
  "hookSpecificOutput": {
    "hookEventName": "UserPromptSubmit",
    "additionalContext": "Current sprint: auth refactor. Ticket JIRA-481 is in review."
  }
}
```

Two limits bound this. Injected text is plain text, so hooks cannot trigger slash commands or tool calls. They only feed text and decisions back. [^9] And output strings (`additionalContext`, `systemMessage`, plain stdout) are capped at 10,000 characters, with overflow saved to a file and a preview plus path shown in its place. [^35] One more thing the reference notes: for mid-session events, resuming with `--continue` or `--resume` replays the saved text rather than re-running the hook, so values like timestamps and commit SHAs go stale on resume. `SessionStart` hooks run again on resume, so they can refresh.

### 4.5 `updatedInput`, rewrite a tool call before it runs

A `PreToolUse` hook can modify the tool's arguments instead of just allowing or denying. It can sanitize a command, redirect a path, normalize an arg. Put `updatedInput` inside `hookSpecificOutput` and set `permissionDecision: "allow"` to auto-approve the rewrite or `"ask"` to show it. With `"defer"`, `updatedInput` is ignored. [^36]

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow",
    "updatedInput": { "command": "npm run lint" }
  }
}
```

One caveat. If multiple `PreToolUse` hooks return `updatedInput` for the same tool, the last to finish wins, and since hooks run in parallel that order is non-deterministic. Do not have two hooks rewrite the same tool's input. [^9]

### 4.6 How multiple hooks combine

When several hooks match one event, every hook's command runs to completion, then the harness merges results. Two rules carry the weight here. [^37]

1. **Most restrictive wins** for permission decisions, in the order `deny` > `defer` > `ask` > `allow`. One hook's `deny` blocks the call no matter what siblings return.
2. **A `deny` does not stop sibling hooks from executing.** If hook A denies and hook B has a side effect, B's side effect still happens. So a guardrail hook that denies `rm -rf` does not suppress a logging hook that already recorded the command. The guide spells out this exact pairing: the log entry is still written because the logging hook already ran. That is what you want for audit, and a footgun if you assumed `deny` short-circuits.

So write each hook to act on its own. Never lean on another hook having run, or not run, first.

---

## 5. Canonical patterns

These are the patterns worth knowing cold. Each one is production-shaped, not a toy.

### 5.1 Deterministic guardrail (`PreToolUse` -> deny)

The flagship use. Block dangerous Bash before it executes. Keep the script fast, well under a second, because it sits in the critical path of every matched call, and use exit 2, not exit 1.

```bash
#!/bin/bash
# .claude/hooks/block-dangerous-bash.sh
INPUT=$(cat)
CMD=$(echo "$INPUT" | jq -r '.tool_input.command // empty')

# Deny-list of patterns that must never run
if echo "$CMD" | grep -Eq 'rm[[:space:]]+-rf[[:space:]]+/|git[[:space:]]+push[[:space:]].*--force|DROP[[:space:]]+TABLE'; then
  echo "Blocked: '$CMD' matches a prohibited pattern." >&2
  exit 2
fi
exit 0
```

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [{ "type": "command", "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/block-dangerous-bash.sh", "timeout": 5 }]
      }
    ]
  }
}
```

Anthropic ships a complete reference implementation, [`bash_command_validator_example.py`](https://github.com/anthropics/claude-code/blob/main/examples/hooks/bash_command_validator_example.py), and it is worth reading for the parsing edge cases. [^38] Because `PreToolUse` `deny` overrides even `bypassPermissions`, this is the layer where org policy actually lives. [^4]

Protect-files is the same shape with a path check, and the guide ships this one nearly as-is.

```bash
#!/bin/bash
# .claude/hooks/protect-files.sh
INPUT=$(cat)
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')
PROTECTED=(".env" "package-lock.json" ".git/")
for p in "${PROTECTED[@]}"; do
  if [[ "$FILE_PATH" == *"$p"* ]]; then
    echo "Blocked: $FILE_PATH matches protected pattern '$p'" >&2
    exit 2
  fi
done
exit 0
```

Registered on `PreToolUse` with matcher `Edit|Write`. [^39]

> **Coverage gap to teach explicitly.** `Edit` and `Write` matchers do not see files created or modified through the **`Bash` tool** (`echo > file`, `sed -i`, `cp`). The guide flags this directly: if your hook must see every file change, add a `Stop` hook that scans the working tree once per turn, or also match `Bash` and have the script list modified and untracked files with `git status --porcelain`. [^40]

### 5.2 Auto-format (`PostToolUse` -> format the changed file)

Formatting should never be the model's job. Run the formatter after every edit. This is Anthropic's own example.

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [{ "type": "command", "command": "jq -r '.tool_input.file_path' | xargs npx prettier --write" }]
      }
    ]
  }
}
```

[^41]

`PostToolUse` cannot undo the edit, since the tool already ran, so it is for fix-up and feedback rather than blocking. [^9] For a formatter whose result you do not need back in context, make it async (section 5.7) so it does not add latency. Swap `prettier` for `ruff format`, `gofmt`, `rustfmt`, and so on, gated by extension via `if` (`Edit(*.py)`) or a check inside the script.

### 5.3 Context injection (`SessionStart` -> pipe project state in)

Load live state at the start of every session and re-load it after compaction, which can quietly drop the details you cared about. The simplest form is that anything echoed to stdout on `SessionStart` becomes context. The guide ships exactly this with a `compact` matcher.

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "compact",
        "hooks": [{ "type": "command", "command": "echo 'Reminder: use Bun, not npm. Run bun test before committing. Current sprint: auth refactor.'" }]
      }
    ]
  }
}
```

Replace the `echo` with anything dynamic: `git log --oneline -5`, `cat STATUS.md`, a `gh issue view` of the active ticket, or the output of a project-state script. [^42]

> **Choose the right tool.** For context needed on every session, `CLAUDE.md` is simpler and survives compaction by being re-read from disk (see Ch. 03). Reach for a `SessionStart` hook when the context is dynamic (current branch, open ticket, last deploy) or when you specifically want to re-inject after compaction with the `compact` matcher. The guide makes the same recommendation. [^42]

A close cousin is input validation or context on `UserPromptSubmit`: reject prompts that would leak secrets (exit 2 plus stderr), or attach ticket context via `additionalContext` (section 4.4).

### 5.4 Compliance audit trail (`PostToolUse` -> append immutable log)

For regulated environments this is the highest-value pattern in the chapter. Log every tool call to an append-only record.

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Bash",
        "hooks": [{ "type": "command", "command": "jq -c '{ts: now|todate, session: .session_id, cwd: .cwd, cmd: .tool_input.command}' >> ~/.claude/bash-audit.log" }]
      }
    ]
  }
}
```

A few things turn this from a log into an auditor-grade trail. Audit on `PreToolUse` too, not just `PostToolUse`, because a denied call never reaches `PostToolUse` and attempted-but-blocked actions only show up if you also log at `Pre`. The "log plus guardrail" pair from section 4.6 is the canonical shape, and the guide builds it explicitly: the logging hook records the command, the guardrail hook denies it, and both run, so the audit captures the attempt and the block. [^37] Track config tampering with `ConfigChange`, appending who, what, and when, or returning exit 2 or `{"decision":"block"}` to refuse unauthorized changes mid-session. [^43] For a team-wide, tamper-resistant sink, use an `http` hook posting to a central audit service rather than a local file an engineer can edit. [^21] And remember the Bash-tool coverage gap from section 5.1: file changes made through the shell will not appear under `Edit|Write` audit, so add a `Stop`-hook working-tree scan for completeness.

What goes in the log is a controls question, not an Anthropic-spec one. Practitioner Amit Kothari's SOC 2 guide proposes a minimum set for evidence packages: *"user authentication events, data access by repository or codebase, code modifications or suggestions accepted, and security policy violations or approval overrides."* [^44] That framing is one practitioner's, not an Anthropic specification, so confirm against your own auditor's controls.

### 5.5 Stop-gate (`Stop` -> don't let Claude quit until the check passes)

Turn "looks done" into "is done" by gating the end of a turn on a real check. Anthropic frames this as the deterministic rung of the verification ladder: *"a Stop hook runs your check as a script and blocks the turn from ending until it passes. Claude Code overrides the hook and ends the turn after 8 consecutive blocks."* [^45] The deeper treatment of verification, including TDD with agents and adversarial review, lives in Ch. 12.

```bash
#!/bin/bash
# .claude/hooks/require-tests-green.sh
INPUT=$(cat)
# Avoid an infinite loop: if we already forced a continuation, let Claude stop
if [ "$(echo "$INPUT" | jq -r '.stop_hook_active')" = "true" ]; then exit 0; fi
if ! npm test --silent >/dev/null 2>&1; then
  echo "Tests are failing. Fix them before finishing." >&2
  exit 2   # block the stop; Claude keeps working
fi
exit 0
```

The `stop_hook_active` guard is mandatory. Skip it and you have built a loop. The harness keeps its own backstop: it overrides a `Stop` hook after 8 consecutive blocks without progress, then ends the turn with a warning. Raise that ceiling only when you have a genuinely convergent loop, via `CLAUDE_CODE_STOP_HOOK_BLOCK_CAP`. [^46]

### 5.6 Notifications, env reload, auto-approve (quick hits)

- **Notify when Claude needs you.** A `Notification` hook running `osascript`, `notify-send`, or PowerShell, with matcher `permission_prompt` or `idle_prompt` to narrow. [^47]
- **Reload `direnv` on `cd`.** Pair `SessionStart` and `CwdChanged`, each running `direnv export bash > "$CLAUDE_ENV_FILE"`, so Claude's Bash tool picks up per-directory env. The same pattern works with `devbox shellenv` for devbox or nix. [^48]
- **Auto-approve a specific prompt.** A `PermissionRequest` hook with a narrow matcher (e.g. `ExitPlanMode`) returning `{"hookSpecificOutput":{"hookEventName":"PermissionRequest","decision":{"behavior":"allow"}}}`. Keep the matcher tight. The guide warns that an empty or `.*` matcher auto-approves everything, including file writes and shell commands. [^49]

### 5.7 Async hooks, side effects off the critical path

For side-effect-only hooks (logging, webhooks, metrics) that do not need to influence behavior, run them in the background so they do not add latency. [^50]

- **`"async": true`** runs in the background, and the agent proceeds without waiting.
- **`"asyncRewake": true`** runs in the background but wakes Claude on exit code 2, surfacing stderr (or stdout if stderr is empty) as a system reminder.

Async hooks cannot block, deny, or inject context, because by the time they finish the agent has already moved on. Use them only for fire-and-forget work. The auto-format hook in section 5.2 is a good async candidate. The guardrail in section 5.1 has to stay synchronous.

---

## 6. Deterministic vs. prompt/agent hooks

The default and the safe choice is a deterministic command hook: a script with a fixed rule, fast and predictable. Some conditions, though, want judgment. Is this prompt actually trying to leak a secret? Are all the requested tasks really done? For those, a hook can call a model. Anthropic's own guidance: *"For decisions that require judgment rather than deterministic rules, you can also use prompt-based hooks or agent-based hooks that use a Claude model to evaluate conditions."* [^2]

### 6.1 Prompt hooks (`type: "prompt"`)

A single-turn model call, Haiku by default, with a `model` override available. The model returns only a yes/no decision as JSON: `{"ok": true}` to proceed, `{"ok": false, "reason": "..."}` otherwise. What `ok: false` does depends on the event. On `Stop` and `SubagentStop` the `reason` is fed back so Claude keeps working. On `PreToolUse` the call is denied with the reason as the tool error. On `PostToolUse`, `PostToolBatch`, `UserPromptSubmit`, and `UserPromptExpansion` the turn ends and the reason appears in the chat as a warning. [^51]

```json
{
  "hooks": {
    "Stop": [
      { "hooks": [{ "type": "prompt", "prompt": "Check if all tasks are complete. If not, respond with {\"ok\": false, \"reason\": \"what remains to be done\"}." }] }
    ]
  }
}
```

### 6.2 Agent hooks (`type: "agent"`), experimental

When the decision needs to inspect actual state (read files, run the suite), an agent hook spawns a subagent with tool access, with a longer default timeout of 60 seconds and up to 50 tool-use turns. Same `ok` and `reason` format. [^52]

```json
{
  "hooks": {
    "Stop": [
      { "hooks": [{ "type": "agent", "prompt": "Verify that all unit tests pass. Run the test suite and check the results. $ARGUMENTS", "timeout": 120 }] }
    ]
  }
}
```

> **Agent hooks are experimental.** The reference warns that behavior and configuration may change, and to prefer command hooks for production workflows. [^23]

### 6.3 Choosing between them

The guide gives the rule of thumb directly: use prompt hooks when the hook input data alone is enough to make a decision, and agent hooks when you need to verify against the actual state of the codebase. [^52]

| Use a... | When |
|--------|------|
| **Command hook** | The rule is deterministic and safety-critical. *Always the default for guardrails, formatting, audit.* |
| **Prompt hook** | The hook's input alone is enough to judge (e.g. "does this prompt look like a secret leak"). |
| **Agent hook** | You must verify against real codebase or runtime state, and you accept experimental status plus cost and latency. |

The principle for a senior cohort is the line that should make you uncomfortable if you ever break it. Safety-critical rules are deterministic, judgment calls are model-evaluated, and you never let a model be the only thing standing between the agent and `rm -rf /`. A model hook can supplement a deterministic guardrail. It should never replace one.

---

## 7. Hooks in the Agent SDK

When you embed Claude programmatically through the Agent SDK (Python and TypeScript), hooks become callback functions instead of shell scripts. Same lifecycle, same matcher rules, same JSON output shape, different registration. [^53]

Register through the `hooks` option, keyed by event, with `HookMatcher` entries in Python and matcher objects in TypeScript.

```python
from claude_agent_sdk import ClaudeAgentOptions, ClaudeSDKClient, HookMatcher

async def protect_env_files(input_data, tool_use_id, context):
    file_path = input_data["tool_input"].get("file_path", "")
    if file_path.split("/")[-1] == ".env":
        return {
            "hookSpecificOutput": {
                "hookEventName": input_data["hook_event_name"],
                "permissionDecision": "deny",
                "permissionDecisionReason": "Cannot modify .env files",
            }
        }
    return {}   # {} = allow unchanged

options = ClaudeAgentOptions(
    hooks={"PreToolUse": [HookMatcher(matcher="Write|Edit", hooks=[protect_env_files])]}
)
```

```typescript
import { query, HookCallback, PreToolUseHookInput } from "@anthropic-ai/claude-agent-sdk";

const protectEnvFiles: HookCallback = async (input, toolUseID, { signal }) => {
  const pre = input as PreToolUseHookInput;
  const ti = pre.tool_input as Record<string, unknown>;
  if ((ti.file_path as string)?.split("/").pop() === ".env") {
    return { hookSpecificOutput: { hookEventName: pre.hook_event_name,
      permissionDecision: "deny", permissionDecisionReason: "Cannot modify .env files" } };
  }
  return {};
};
// options: { hooks: { PreToolUse: [{ matcher: "Write|Edit", hooks: [protectEnvFiles] }] } }
```

[^54]

Specifics worth knowing, all from the SDK hooks doc:

- A callback receives `(input_data, tool_use_id, context)`. `tool_use_id` correlates `PreToolUse` and `PostToolUse` for the same call. In TypeScript, `context.signal` is an `AbortSignal` for cancellation. In Python the third argument is reserved for future use.
- Return `{}` to allow unchanged. The same merge rule applies: deny over defer over ask over allow, and a single `deny` blocks the call regardless of the others.
- The same `updatedInput` and `updatedToolOutput` fields are available. `updatedToolOutput` replaces a tool's output before Claude sees it and works for any tool in both SDKs. The older `updatedMCPToolOutput` field is deprecated. On `PostToolUse`, `additionalContext` appends to the tool result.
- Async outputs return `{"async_": True, "asyncTimeout": 30000}` in Python and `{ async: true, asyncTimeout: 30000 }` in TypeScript for fire-and-forget side effects.
- Not every event is available in both SDKs. `SessionStart` and `SessionEnd` register as callback hooks in TypeScript, but in Python they are only available as shell command hooks loaded from settings, which the SDK reads when you include the right `setting_sources` or `settingSources` entry (for example `["project"]`). The SDK reads `.claude/settings.json` command hooks by default under standard `query()` options, so settings-defined and callback hooks coexist.

[^55]

For gating individual tool calls, the SDK also exposes `canUseTool`, a permission callback distinct from hooks. Reach for it alongside `PreToolUse` hooks and deny rules to bound a programmatic agent. [^56]

There is a practical SDK footgun lurking in all this. A `UserPromptSubmit` hook that spawns subagents can recurse forever if those subagents re-trigger the same hook. The SDK doc names the fix: check for a subagent indicator in the input before spawning, track whether you are already inside a subagent with session state, or scope the hook to top-level sessions only. [^57]

---

## 8. Security: a hook is arbitrary code execution

This earns its own section, because the blast radius is real. The reference puts it plainly: *"Command hooks run with your system user's full permissions."* The boxed warning spells out the blast radius: *"Command hooks execute shell commands with your full user permissions. They can modify, delete, or access any files your user account can access. Review and test all hook commands before adding them to your configuration."* [^58]

Here is the part that gets seniors specifically. `.claude/settings.json` is committed to the repo, and so are plugin `hooks/hooks.json` and skill or agent frontmatter (see section 3.1). A pull request can therefore add a hook that runs on your machine the next time you open the project. The mitigation the reference points at is the same one you already apply to any executable change: review and test before adding. So review hook changes in a PR the way you review any code that runs with your privileges, and arguably more carefully, because they run silently and automatically.

Anthropic's hardening checklist for hook authors runs five points deep. [^59]

1. **Validate and sanitize inputs.** Parse the JSON carefully and validate field values before passing any of them into a shell command.
2. **Quote shell variables.** Wrap variables in double quotes to stop word-splitting and glob expansion. Write `"$cwd"`, not `$cwd`.
3. **Block path traversal.** Check for `../` sequences and reject paths that escape your project root.
4. **Use absolute paths.** Prefer absolute paths over relative ones to avoid working-directory surprises. In exec form `${CLAUDE_PROJECT_DIR}` needs no quoting; in shell form, wrap it in double quotes.
5. **Skip sensitive files.** Never read or log files holding secrets: `.env`, `.aws/credentials`, SSH keys, and the like.

For enterprise and regulated cohorts the architecture writes itself. Put the non-bypassable guardrails and audit hooks in managed policy settings, which are org-controlled and cannot be disabled without managed-level `disableAllHooks`, lock the rest down with `allowManagedHooksOnly` if you want only vetted hooks running, and treat project-committed hooks as reviewable-but-untrusted. [^60]

---

## 9. Debugging hooks

The failure modes are stereotyped. Learn the checklist and you will fix most issues in a minute. [^61]

- **Hook never fires.** Run `/hooks` and confirm it is listed under the right event. Check the matcher is case-correct and matches the tool name exactly. Confirm you are using the right event (`PreToolUse` vs `PostToolUse`). In `-p` mode, `PermissionRequest` hooks do not fire, so use `PreToolUse`.
- **`<hook> hook error` in the transcript.** The script exited non-zero unexpectedly. Test it standalone: `echo '{"tool_name":"Bash","tool_input":{"command":"ls"}}' | ./my-hook.sh; echo $?`. "command not found" means use absolute paths or `${CLAUDE_PROJECT_DIR}`, or add `"args": []` to switch to exec form (spawned directly, no shell, no quoting headaches). "jq: command not found" means install `jq` or parse in Python or Node. Not running at all means `chmod +x` the script.
- **`/hooks` shows nothing after you edited settings.** Invalid JSON (no trailing commas, no comments) or the watcher missed it. Restart the session.
- **JSON validation failed despite valid output.** Your shell profile is printing to stdout in non-interactive shells and corrupting the hook's JSON. Wrap profile `echo`s in an interactivity check: `if [[ $- == *i* ]]; then echo "..."; fi`.
- **`Stop` hook hits the block cap.** Claude keeps working then warns it blocked too many times. You forgot the `stop_hook_active` guard (section 5.5), and the harness overrides after 8 consecutive blocks. Add the guard, and raise `CLAUDE_CODE_STOP_HOOK_BLOCK_CAP` only for genuinely convergent loops.
- **Tool blocked unexpectedly.** Some `PreToolUse` hook returned `deny`. Add logging of `permissionDecisionReason`, and check that matchers are not too broad (an empty matcher means all tools).

For deep debugging, toggle the transcript with `Ctrl+O` (one line per fired hook, and success is silent), and for full detail run with `claude --debug-file /tmp/claude.log` then `tail -f /tmp/claude.log`, or `/debug` mid-session. Set `CLAUDE_CODE_DEBUG_LOG_LEVEL=verbose` for matcher-level detail. [^62]

---

## 10. Decision guidance and a hook starter set

When to add a hook, versus `CLAUDE.md` or a skill:

- The rule must hold every time with no exceptions -> hook (deterministic command).
- The model already does it correctly without prompting -> add nothing.
- You keep writing "ALWAYS..." in `CLAUDE.md` and it is still ignored sometimes -> convert it to a hook. Anthropic's own pruning advice in the failure-patterns section: *"If Claude already does something correctly without the instruction, delete it or convert it to a hook."* [^63]
- It needs judgment, not a fixed rule -> prompt or agent hook, but keep a deterministic backstop for anything safety-critical.

A sane starter set for a real project, with the project-scoped ones committed in `.claude/settings.json`:

1. `PreToolUse` / `Bash` -> block destructive commands (section 5.1). Managed settings if you can.
2. `PreToolUse` / `Edit|Write` -> protect `.env`, lockfiles, `.git/` (section 5.1).
3. `PostToolUse` / `Edit|Write` -> auto-format, async (section 5.2).
4. `SessionStart` with the `compact` matcher -> inject current branch, project state, active ticket (section 5.3).
5. `PostToolUse` plus `PreToolUse` -> audit log, if you are in a regulated context (section 5.4).
6. `Notification` -> desktop or Slack ping when Claude needs you (section 5.6).
7. `Stop` -> gate on tests or build for unattended runs (section 5.5).

One final caution on version sensitivity. Claude Code ships near-daily. The event roster, the `if` field (v2.1.85+), field names like `terminalSequence` (v2.1.141+) and `updatedToolOutput`, and the per-event decision shapes all keep moving. Before teaching from this chapter or shipping a hook to a team, re-check the live [Hooks reference](https://code.claude.com/docs/en/hooks) and [changelog](https://code.claude.com/docs/en/changelog). Anything here beyond the core seven events and the exit-code and JSON protocols should be verified against the binary you are actually running.

---

## Sources

Primary (official Anthropic), all fetched 2026-06-18:

- Hooks reference -- https://code.claude.com/docs/en/hooks
- Automate actions with hooks (guide) -- https://code.claude.com/docs/en/hooks-guide
- Best practices for Claude Code -- https://code.claude.com/docs/en/best-practices
- Agent SDK -- Intercept and control agent behavior with hooks -- https://code.claude.com/docs/en/agent-sdk/hooks
- Agent SDK -- Permissions (`canUseTool`) -- https://code.claude.com/docs/en/agent-sdk/permissions
- Changelog (terminalSequence v2.1.141, MessageDisplay v2.1.152) -- https://code.claude.com/docs/en/changelog
- Bash command validator example -- https://github.com/anthropics/claude-code/blob/main/examples/hooks/bash_command_validator_example.py

Practitioner (one author's controls framing, not an Anthropic specification):

- SOC 2 audit-log minimum set (Amit Kothari) -- https://amitkoth.com/claude-code-soc2-compliance-auditor-guide

[^1]: web | 2026-06-18 | [code.claude.com/docs/en/best-practices](https://code.claude.com/docs/en/best-practices), "Set up hooks"
[^2]: web | 2026-06-18 | [code.claude.com/docs/en/hooks-guide](https://code.claude.com/docs/en/hooks-guide), opening
[^3]: web | 2026-06-18 | [code.claude.com/docs/en/best-practices](https://code.claude.com/docs/en/best-practices), "Set up hooks" tip
[^4]: web | 2026-06-18 | [code.claude.com/docs/en/hooks-guide](https://code.claude.com/docs/en/hooks-guide), "Hooks and permission modes"
[^5]: web | 2026-06-18 | [code.claude.com/docs/en/hooks-guide](https://code.claude.com/docs/en/hooks-guide), "How hooks work"
[^6]: web | 2026-06-18 | [code.claude.com/docs/en/hooks](https://code.claude.com/docs/en/hooks), "Hook lifecycle"
[^7]: web | 2026-06-18 | [code.claude.com/docs/en/changelog](https://code.claude.com/docs/en/changelog), v2.1.141 and v2.1.152
[^8]: web | 2026-06-18 | [code.claude.com/docs/en/hooks-guide](https://code.claude.com/docs/en/hooks-guide), "How hooks work" event table
[^9]: web | 2026-06-18 | [code.claude.com/docs/en/hooks-guide](https://code.claude.com/docs/en/hooks-guide), "Limitations"
[^10]: web | 2026-06-18 | [code.claude.com/docs/en/hooks](https://code.claude.com/docs/en/hooks), "PostToolBatch"
[^11]: web | 2026-06-18 | [code.claude.com/docs/en/hooks](https://code.claude.com/docs/en/hooks), event reference
[^12]: web | 2026-06-18 | [code.claude.com/docs/en/hooks-guide](https://code.claude.com/docs/en/hooks-guide), "Filter hooks with matchers" table
[^13]: web | 2026-06-18 | [code.claude.com/docs/en/hooks](https://code.claude.com/docs/en/hooks), "Configuration"
[^14]: web | 2026-06-18 | [code.claude.com/docs/en/hooks-guide](https://code.claude.com/docs/en/hooks-guide), "Configure hook location"
[^15]: web | 2026-06-18 | [code.claude.com/docs/en/hooks-guide](https://code.claude.com/docs/en/hooks-guide), "Configure hook location" and "/hooks shows no hooks configured"
[^16]: web | 2026-06-18 | [code.claude.com/docs/en/hooks](https://code.claude.com/docs/en/hooks), "Hook locations"
[^17]: web | 2026-06-18 | [code.claude.com/docs/en/hooks](https://code.claude.com/docs/en/hooks), "Matcher patterns"
[^18]: web | 2026-06-18 | [code.claude.com/docs/en/hooks](https://code.claude.com/docs/en/hooks), "Match MCP tools"; code.claude.com/docs/en/hooks-guide, "Hook not firing"
[^19]: web | 2026-06-18 | [code.claude.com/docs/en/hooks-guide](https://code.claude.com/docs/en/hooks-guide), "Filter by tool name and arguments with the if field"
[^20]: web | 2026-06-18 | [code.claude.com/docs/en/hooks-guide](https://code.claude.com/docs/en/hooks-guide), "How hooks work" type list
[^21]: web | 2026-06-18 | [code.claude.com/docs/en/hooks-guide](https://code.claude.com/docs/en/hooks-guide), "HTTP hooks"
[^22]: web | 2026-06-18 | [code.claude.com/docs/en/hooks](https://code.claude.com/docs/en/hooks), "MCP tool hook fields"
[^23]: web | 2026-06-18 | [code.claude.com/docs/en/hooks-guide](https://code.claude.com/docs/en/hooks-guide), "Agent-based hooks" warning
[^24]: web | 2026-06-18 | [code.claude.com/docs/en/hooks](https://code.claude.com/docs/en/hooks), "Command hook fields"; code.claude.com/docs/en/hooks-guide, "Limitations"
[^25]: web | 2026-06-18 | [code.claude.com/docs/en/hooks](https://code.claude.com/docs/en/hooks), "Path variables"
[^26]: web | 2026-06-18 | [code.claude.com/docs/en/hooks](https://code.claude.com/docs/en/hooks), "Persist environment variables" and HTTP hook fields
[^27]: web | 2026-06-18 | [code.claude.com/docs/en/hooks-guide](https://code.claude.com/docs/en/hooks-guide), "Structured JSON output"
[^28]: web | 2026-06-18 | [code.claude.com/docs/en/hooks](https://code.claude.com/docs/en/hooks), "Common input fields"
[^29]: web | 2026-06-18 | [code.claude.com/docs/en/hooks-guide](https://code.claude.com/docs/en/hooks-guide), "Hook input"
[^30]: web | 2026-06-18 | [code.claude.com/docs/en/hooks-guide](https://code.claude.com/docs/en/hooks-guide), "Hook output"
[^31]: web | 2026-06-18 | [code.claude.com/docs/en/hooks](https://code.claude.com/docs/en/hooks), exit-code warning
[^32]: web | 2026-06-18 | [code.claude.com/docs/en/hooks](https://code.claude.com/docs/en/hooks), "JSON output"
[^33]: web | 2026-06-18 | [code.claude.com/docs/en/hooks](https://code.claude.com/docs/en/hooks), "Defer a tool call for later"
[^34]: web | 2026-06-18 | [code.claude.com/docs/en/hooks](https://code.claude.com/docs/en/hooks), "Decision control"
[^35]: web | 2026-06-18 | [code.claude.com/docs/en/hooks](https://code.claude.com/docs/en/hooks), "Add context for Claude"
[^36]: web | 2026-06-18 | [code.claude.com/docs/en/hooks](https://code.claude.com/docs/en/hooks), "PreToolUse"
[^37]: web | 2026-06-18 | [code.claude.com/docs/en/hooks-guide](https://code.claude.com/docs/en/hooks-guide), "Combine results from multiple hooks"
[^38]: web | 2026-06-18 | [code.claude.com/docs/en/hooks-guide](https://code.claude.com/docs/en/hooks-guide), "Learn more"
[^39]: web | 2026-06-18 | [code.claude.com/docs/en/hooks-guide](https://code.claude.com/docs/en/hooks-guide), "Block edits to protected files"
[^40]: web | 2026-06-18 | [code.claude.com/docs/en/hooks-guide](https://code.claude.com/docs/en/hooks-guide), "Filter hooks with matchers" note
[^41]: web | 2026-06-18 | [code.claude.com/docs/en/hooks-guide](https://code.claude.com/docs/en/hooks-guide), "Auto-format code after edits"
[^42]: web | 2026-06-18 | [code.claude.com/docs/en/hooks-guide](https://code.claude.com/docs/en/hooks-guide), "Re-inject context after compaction"
[^43]: web | 2026-06-18 | [code.claude.com/docs/en/hooks-guide](https://code.claude.com/docs/en/hooks-guide), "Audit configuration changes"
[^44]: web | 2026-06-18 | [amitkoth.com/claude-code-soc2-compliance-auditor-guide](https://amitkoth.com/claude-code-soc2-compliance-auditor-guide) (Amit Kothari), "minimum requirements"
[^45]: web | 2026-06-18 | [code.claude.com/docs/en/best-practices](https://code.claude.com/docs/en/best-practices), "Give Claude a way to verify its work"
[^46]: web | 2026-06-18 | [code.claude.com/docs/en/hooks-guide](https://code.claude.com/docs/en/hooks-guide), "Stop hook hits the block cap"
[^47]: web | 2026-06-18 | [code.claude.com/docs/en/hooks-guide](https://code.claude.com/docs/en/hooks-guide), "Get notified when Claude needs input"
[^48]: web | 2026-06-18 | [code.claude.com/docs/en/hooks-guide](https://code.claude.com/docs/en/hooks-guide), "Reload environment when directory or files change"
[^49]: web | 2026-06-18 | [code.claude.com/docs/en/hooks-guide](https://code.claude.com/docs/en/hooks-guide), "Auto-approve specific permission prompts"
[^50]: web | 2026-06-18 | [code.claude.com/docs/en/hooks](https://code.claude.com/docs/en/hooks), async hook fields
[^51]: web | 2026-06-18 | [code.claude.com/docs/en/hooks-guide](https://code.claude.com/docs/en/hooks-guide), "Prompt-based hooks"
[^52]: web | 2026-06-18 | [code.claude.com/docs/en/hooks-guide](https://code.claude.com/docs/en/hooks-guide), "Agent-based hooks"
[^53]: web | 2026-06-18 | [code.claude.com/docs/en/agent-sdk/hooks](https://code.claude.com/docs/en/agent-sdk/hooks), "How hooks work"
[^54]: web | 2026-06-18 | [code.claude.com/docs/en/agent-sdk/hooks](https://code.claude.com/docs/en/agent-sdk/hooks), "How hooks work" example
[^55]: web | 2026-06-18 | [code.claude.com/docs/en/agent-sdk/hooks](https://code.claude.com/docs/en/agent-sdk/hooks), "Available hooks", "Outputs", "Asynchronous output", "Session hooks not available in Python"
[^56]: web | 2026-06-18 | [code.claude.com/docs/en/agent-sdk/permissions](https://code.claude.com/docs/en/agent-sdk/permissions)
[^57]: web | 2026-06-18 | [code.claude.com/docs/en/agent-sdk/hooks](https://code.claude.com/docs/en/agent-sdk/hooks), "Recursive hook loops with subagents"
[^58]: web | 2026-06-18 | [code.claude.com/docs/en/hooks](https://code.claude.com/docs/en/hooks), "Security considerations" disclaimer and warning
[^59]: web | 2026-06-18 | [code.claude.com/docs/en/hooks](https://code.claude.com/docs/en/hooks), "Security best practices"
[^60]: web | 2026-06-18 | [code.claude.com/docs/en/hooks](https://code.claude.com/docs/en/hooks), "Hook locations"; code.claude.com/docs/en/hooks-guide, "Configure hook location"
[^61]: web | 2026-06-18 | [code.claude.com/docs/en/hooks-guide](https://code.claude.com/docs/en/hooks-guide), "Limitations and troubleshooting"
[^62]: web | 2026-06-18 | [code.claude.com/docs/en/hooks-guide](https://code.claude.com/docs/en/hooks-guide), "Debug techniques"; code.claude.com/docs/en/hooks, "Debug hooks"
[^63]: web | 2026-06-18 | [code.claude.com/docs/en/best-practices](https://code.claude.com/docs/en/best-practices), "Avoid common failure patterns"
