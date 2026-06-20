# Chapter 04 -- Driving the Agent

**TL;DR.** Mastery of Claude Code has less to do with prompts than with driving. You are choosing, moment to moment, how much oversight the agent gets, and recovering cleanly when it wanders off the road. This chapter covers the controls that turn a chat into a steerable agentic loop: **plan mode** (explore -> plan -> approve -> execute), the **permission modes** that govern how often the agent stops to ask, the **slash-command** surface (built-in, bundled, and your own custom commands/skills), **checkpoints & rewind** for undoing edits, the autonomous **`/goal`** loop, and **headless** (`claude -p`) for scripting and CI. One safety fact sits above the rest: **checkpoints revert file *edits*, not Bash side effects.** A `git push`, an `rm`, a DB migration, or a `pip install` is permanent, and rewind cannot take it back. Once that asymmetry lives in your hands, you configure permissions correctly without thinking about it.

> **A note on versions.** Claude Code ships near-daily, so every version-pinned number below (e.g. "auto mode requires v2.1.83") is a "this exists, roughly here" marker, not a fixed fact. Re-verify anything you teach or script against the changelog (`https://code.claude.com/docs/en/changelog`) or `/release-notes`, and treat prices and model availability the same way -- read them from the live pages rather than memorizing. [^1]

---

## 1. The mental model: oversight is a dial, not a switch

Claude Code runs an **agentic loop**: gather context, take action, verify results, and repeat, with you inside that loop the whole time. The phases blend together, and you stay part of it. Press `Esc` to stop the running tool immediately, or type a correction and `Enter` to steer without stopping the current action. [^2]

What this chapter teaches is how to modulate oversight over the life of a single task. One session might travel through several postures before it's done:

1. **Read-only exploration** while you and the agent build a shared model of the problem (plan mode).
2. **Reviewed execution** where you approve each edit (default mode).
3. **Trusted execution** where the agent edits freely and you review the diff afterward (`acceptEdits`).
4. **Unattended execution** where a classifier or a goal evaluator stands in for you (`auto` mode, `/goal`).

Every control below exists so you can move along that dial on purpose. You raise oversight for the parts that cannot be undone, the deploys and migrations and force-pushes, and you lower it for the cheap reversible parts, the source you're about to read in a diff anyway. The edge a senior engineer carries into this is knowing which parts are which, and the checkpoints section turns that instinct into a rule.

---

## 2. Plan mode: explore -> plan -> approve -> execute

### 2.1 What plan mode is

Plan mode is **enforced read-only exploration.** Claude reads files, runs shell commands to explore, and writes a plan, but it does **not edit your source** until you approve. Permission prompts still apply for anything that isn't read-only, exactly as in default mode. [^3]

This is the highest-leverage habit the tool has. Anthropic's own guidance is to separate research from coding for complex problems. Work the problem in two phases, explore and then implement against an approved plan, and the results come back materially better than they would have if you'd dived straight into edits. The reason is plain: the agent commits its understanding to something you can read before it lays a finger on the code. [^4]

### 2.2 Entering and leaving plan mode

Four ways in:

- **`Shift+Tab`** cycles permission modes; plan is in the default cycle (`default` -> `acceptEdits` -> `plan`). [^5]
- **`/plan`** enters plan mode for the next prompt. You can hand it the task in one shot: `/plan fix the auth bug`. [^6]
- **`claude --permission-mode plan`** starts the whole session in plan mode. [^7]
- **`defaultMode: "plan"`** in `.claude/settings.json` makes plan the project default. [^8]

Press `Shift+Tab` again to leave plan mode *without* approving a plan.

### 2.3 Reviewing and approving the plan

When the plan is ready, Claude presents it and asks how to proceed. This is where the handoff from exploring to executing happens, and there's a detail that's easy to miss: approving a plan is also how you choose your execution posture. Approving exits plan mode and switches the session to whichever mode the option describes. [^9]

- **Approve and start in auto mode** -- unattended execution with background safety checks.
- **Approve and accept edits** -- Claude edits freely; you review the diff after.
- **Approve and review each edit manually** -- back to default-mode prompting.
- **Keep planning with feedback** -- iterate on the plan in conversation.
- **Refine with Ultraplan** -- the dialog's "No, refine with Ultraplan on Claude Code on the web" option hands the draft to a browser-based plan-mode session where you comment on individual sections, then execute on the web or teleport back to your terminal. Requires v2.1.91+; unavailable on Bedrock/Vertex/Foundry. [^10]

Three details worth knowing: [^9]

- **`Ctrl+G`** opens the proposed plan in your default text editor so you can edit it directly before Claude proceeds. Editing the plan by hand is often faster than re-prompting -- fix the one wrong assumption, save, approve.
- Accepting a plan **auto-names the session** from the plan's content (unless you already set a name with `--name`/`/rename`), useful when you run several plan-then-build sessions in parallel and need to find them later in `/resume`.
- With `showClearContextOnPlanAccept` enabled, each approve option also offers to **clear the planning context first**, so execution starts on a clean window instead of dragging the whole exploration transcript forward. Worth turning on for large explorations.

### 2.4 The Plan subagent

Plan mode pairs naturally with the read-only **Plan** and **Explore** subagents. Both skip loading `CLAUDE.md` and git status so their context stays small and fast. When you want exploration *with* context isolation, run the research in a forked Explore/Plan agent so only the synthesized findings come back to your main window instead of the full file dump. Subagents get the full treatment in Ch. 08; the thing to carry from here is that plan mode and read-only agents are the same instinct working at two scales. [^11]

### 2.5 Senior workflow

```
Shift+Tab Shift+Tab            # into plan mode
> Read src/auth/ and understand session handling.
  Then plan adding OAuth. Don't write code yet.
[Claude explores, proposes a plan]
Ctrl+G                          # tweak the one wrong assumption
[approve -> review each edit manually]   # for risky work
   ...or [approve -> accept edits]       # for low-stakes work you'll diff after
```

The discipline is one line: never let the agent's first edit be its first commitment. Force the commitment into a plan you can read.

---

## 3. Permission modes: how often the agent stops to ask

Permission modes set the **baseline** for what runs without a prompt. On top of that baseline you layer per-tool `allow`/`ask`/`deny` rules (`/permissions`). Deny rules and explicit ask rules apply in *every* mode, including `bypassPermissions`; allow rules have no effect there because everything else is already approved. [^12]

### 3.1 The six modes

| Mode | Runs without asking | Best for |
|------|---------------------|----------|
| `default` | Reads only | Getting started; sensitive work; per-action review |
| `acceptEdits` | Reads, file edits, common filesystem Bash (`mkdir`, `touch`, `rm`, `rmdir`, `mv`, `cp`, `sed`) -- in-scope paths only | Iterating on code you'll review via `git diff` |
| `plan` | Reads only | Exploring before changing (see section 2) |
| `auto` | Everything, **with a background classifier** vetting each action | Long tasks; reducing prompt fatigue (research preview) |
| `dontAsk` | Only pre-approved (`permissions.allow`) tools + read-only Bash; everything else **auto-denied** | Locked-down CI / scripts |
| `bypassPermissions` | Everything; no safety checks | Isolated containers / VMs **only** |

[^12]

### 3.2 Setting the mode

- **Mid-session:** `Shift+Tab` cycles `default` -> `acceptEdits` -> `plan`. Optional modes slot in after `plan` once enabled, with `bypassPermissions` first and `auto` last. `dontAsk` *never* appears in the cycle -- set it only with the flag. [^5]
- **At startup:** `claude --permission-mode <mode>`.
- **As a default:** `permissions.defaultMode` in settings.

> **Security guardrail in the config system itself.** `defaultMode: "auto"` and `defaultMode: "bypassPermissions"` are **ignored when set in project/local settings** (`.claude/settings.json`, `.claude/settings.local.json`) -- they must live in `~/.claude/settings.json`. The intent is deliberate: a checked-in repo cannot grant *itself* unattended execution on your machine. Claude Code on the web likewise ignores `defaultMode: "bypassPermissions"` / `"dontAsk"` from settings files. (The project-settings ignore for `auto` dates to v2.1.142.) [^13]

### 3.3 `acceptEdits` -- the daily-driver loop

`acceptEdits` is where most of your reviewed-velocity work actually lives. Claude writes files and runs common filesystem commands without prompting, though **only inside your working directory or `additionalDirectories`**. Those commands stay auto-approved even when wrapped in safe environment variables (`LANG=C`, `NO_COLOR=1`) or process wrappers (`timeout`, `nice`, `nohup`); with the PowerShell tool enabled, the PowerShell equivalents (`Set-Content`, `Add-Content`, `Clear-Content`, `Remove-Item`, and aliases) are covered too. Paths outside scope, writes to protected paths, and all other Bash commands still prompt. The status bar shows `>> accept edits on`. The move that makes this mode pay off is pairing it with `git diff` review: let the agent run, then read the diff as one reviewable unit instead of approving keystroke by keystroke. [^14]

### 3.4 `auto` mode -- the classifier stands in for you

`auto` mode (research preview; **requires v2.1.83+**) lets Claude execute without routine permission prompts. A **separate classifier model**, independent of your `/model` selection, reviews each action *before* it runs, blocking anything that escalates beyond your request, targets unrecognized infrastructure, or appears driven by hostile content Claude read. Explicit `ask` rules still force a prompt. [^15]

**What it blocks by default** (treating only your working directory and your repo's configured remotes as trusted): downloading and executing code like `curl | bash`, sending sensitive data to external endpoints, production deploys/migrations, mass cloud-storage deletion, IAM/permission grants, modifying shared infrastructure, irreversibly destroying files that existed before the session, **force push, and pushing directly to `main`.**

**What it allows by default:** local file ops in your working dir, installing dependencies declared in your lockfiles/manifests, reading `.env` and sending credentials to the *matching* API, read-only HTTP, and pushing to the branch you started on (or one Claude created). Run `claude auto-mode defaults` to print the full rule lists as JSON. [^16]

Four properties a senior should know cold:

- **Conversational boundaries are honored.** If you say "don't push" or "wait until I review before deploying," the classifier blocks matching actions even when the default rules would allow them. The boundary holds until *you* lift it in a later message -- and *Claude's own judgment that the condition was met does not lift it.* But boundaries are not stored as rules; the classifier **re-reads them from the transcript on each check**, so compaction can drop the message that stated one. For a hard guarantee, use a `deny` rule, not a sentence. [^17]
- **It circuit-breaks on repeated blocks.** 3 blocks in a row, or 20 total, pauses auto mode and resumes prompting (not configurable; any allowed action resets the consecutive counter, the total persists for the session). In headless `-p` mode, repeated blocks **abort the session** since there's no one to ask. [^18]
- **Broad code-execution allow rules are dropped on entry.** Blanket `Bash(*)`/`PowerShell(*)`, wildcarded interpreters (`Bash(python*)`), package-manager run commands, and `Agent` allow rules are stripped while in auto mode (restored on exit); narrow rules like `Bash(npm test)` carry over. The classifier sees user messages, tool calls, and your `CLAUDE.md` -- **but not tool results** (those are stripped so file/web content can't manipulate it directly), with a separate server-side probe scanning incoming results before Claude reads them. [^19]
- **Subagents are checked three times.** The classifier evaluates the delegated task description before a subagent starts (v2.1.178+), each of its actions while it runs (its frontmatter `permissionMode` is ignored), and its full action history when it finishes, prepending a security warning to the results if the return check flags a concern. Auto mode is not a hole you can route around by spawning a subagent. [^20]

Availability is gated by plan/admin/model/provider. On Team and Enterprise an admin must enable it, and admins can lock it off with `permissions.disableAutoMode: "disable"` in managed settings. Auto mode requires a recent model (Opus 4.6+/Sonnet 4.6 on the Anthropic API; only Opus 4.7/4.8 on Bedrock/Vertex/Foundry, where it's gated behind `CLAUDE_CODE_ENABLE_AUTO_MODE=1`). The model/provider matrix is version-sensitive, so confirm it against the live page before you depend on it. [^15]

### 3.5 `dontAsk` and `bypassPermissions` -- the two CI/container modes

- **`dontAsk`** auto-*denies* anything not in your `permissions.allow` rules or the read-only command set (explicit `ask` rules are denied rather than prompting). Fully non-interactive -- the right baseline for CI where you've pre-declared exactly what Claude may do. [^21]
- **`bypassPermissions`** (a.k.a. `--dangerously-skip-permissions`) disables prompts *and* safety checks, including writes to protected paths (as of v2.1.126; earlier versions still prompted for those). Use **only** in isolated containers/VMs without network access. You cannot enter it from a session that wasn't started with an enabling flag. It refuses to start as root/sudo on Linux/macOS (skipped inside a recognized sandbox -- see the dev-container config). Even here, two protections remain: explicit `ask` rules still prompt, and `rm -rf /` / `rm -rf ~` still prompt as a circuit breaker against model error. Admins can block this mode with `permissions.disableBypassPermissionsMode: "disable"`. [^22]

> Anthropic's own framing: for "far fewer prompts" with a safety net, reach for **auto mode**, not `bypassPermissions`. `bypassPermissions` "offers no protection against prompt injection or unintended actions." [^23]

### 3.6 Protected paths -- the floor under every mode

In every mode *except* `bypassPermissions`, writes to a fixed set of **protected paths are never auto-approved**, and `permissions.allow` rules in settings files cannot pre-approve them -- the safety check runs *before* allow rules are evaluated. (In prompting modes they prompt; in `auto` they route to the classifier; in `dontAsk` they're denied.) The set guards repo state and Claude's own config: `.git`, `.config/git`, `.gitconfig`/`.gitmodules`, shell rc files (`.bashrc`, `.zshrc`, `.profile`, `.envrc`, and the rest), package configs (`.npmrc`, `.yarnrc`, `bunfig.toml`, pnp files), build/toolchain configs (`.cargo`, `.yarn`, `.mvn`, bazel files, gradle/maven wrapper properties), CI/hook configs (`.husky`, `.pre-commit-config.yaml`, lefthook), `.vscode`/`.idea`, `.devcontainer`, `.ripgreprc`, `pyrightconfig.json`, and `.claude` / `.mcp.json` / `.claude.json` (except `.claude/worktrees`). In prompting modes, a `.claude/` write offers a "Yes, and allow Claude to edit its own settings for this session" option. **Teaching point:** this is why `Edit(.claude/**)` in your settings doesn't silently let the agent rewrite your hooks. The protected-path check sits above the allow layer. [^24]

### 3.7 Choosing a mode

| Situation | Mode |
|-----------|------|
| New codebase, building shared understanding | `plan` |
| Sensitive change, want to approve each step | `default` |
| Iterating on source you'll review via `git diff` | `acceptEdits` |
| Long task, trust the direction, want a safety net | `auto` (state irreversible boundaries explicitly *and* back them with deny rules) |
| CI with a pre-declared allowlist | `dontAsk` |
| Throwaway container, no network, full autonomy | `bypassPermissions` |

---

## 4. Slash commands

Type `/` to see everything available; `/` followed by letters filters. A command is recognized **only at the start of a message**; text after the name is passed as arguments. [^25]

### 4.1 The built-in surface (the ones that earn their keybindings)

Availability varies by platform, plan, and environment. What follows is the load-bearing subset, not the full catalog. [^26]

**Setup & memory:** `/init` (bootstrap `CLAUDE.md`; `CLAUDE_CODE_NEW_INIT=1` for the interactive flow through skills/hooks/personal memory) - `/memory` (edit memory files, toggle auto-memory) - `/agents` (manage subagents) - `/mcp` (manage MCP servers/OAuth) - `/permissions` (allow/ask/deny rules; review auto-mode denials) - `/config` (settings; **`/config key=value` sets a setting directly, even in `-p`**, from v2.1.181) - `/hooks` (view hook configs).

**During a task:** `/plan` - `/model` (switch + save default; `s` on a row = session-only) - `/effort` (`low`/`medium`/`high`/`xhigh`/`max`/`ultracode`; `max` and `ultracode` are session-only) - `/advisor` (consult a second model for guidance at key moments; `opus`/`sonnet`/`fable` or a full id; v2.1.98+) - `/context` (visualize window usage as a grid + optimization hints) - `/usage` (cost + limits; per-skill/subagent/plugin/MCP breakdown on paid plans; aliases `/cost`, `/stats`) - `/compact [focus]` - `/clear` (fresh context; aliases `/reset`, `/new`) - `/btw` (side question that doesn't bloat history) - `/diff` (interactive diff viewer, per-turn).

**Recovery & sessions:** `/rewind` (aliases `/checkpoint`, `/undo` -- section 6) - `/resume` (alias `/continue`) - `/branch` - `/rename` - `/export` - `/cd` (move session's working dir, v2.1.169+) - `/add-dir`.

**Autonomy & parallelism:** `/goal` (section 5) - `/background` (alias `/bg`) - `/tasks` (alias `/bashes`) - `/fork` (forked subagent, v2.1.161+) - `/loop` (bundled skill, alias `/proactive` -- repeat a prompt on an interval or self-paced) - `/batch` (bundled skill -- decompose a large change into 5-30 units, one worktree each).

**Review & verify:** `/review [PR]` (local) - `/code-review [level] [--fix] [--comment] [target]` (bundled skill; `/code-review ultra` for the cloud variant) - `/simplify` (bundled skill, cleanup-only, v2.1.154+) - `/security-review` - `/run`, `/verify`, `/run-skill-generator` (bundled skills, v2.1.145+) - `/debug` (bundled skill) - `/claude-api` (bundled skill).

**Diagnostics:** `/doctor` (diagnose install/config; `f` to auto-fix) - `/status` - `/release-notes` - `/insights`, `/recap`. (To isolate a bad customization, the controlling switch is the startup flag `--safe-mode` (v2.1.169+), not a slash command -- it starts a session with `CLAUDE.md`, skills, plugins, hooks, MCP servers, custom commands/agents, and auto-memory all disabled.) [^27]

> Some legacy commands have changed: `/vim` was **removed in v2.1.92** (use `/config` -> Editor mode); `/pr-comments` was **removed in v2.1.91** (just ask Claude to view PR comments). Confirm anything you script against. [^28]

### 4.2 Command vs. bundled skill vs. workflow

In the official reference, entries are tagged by *kind*, and the distinction earns its keep the moment you start reasoning about behavior. [^29]

- **Built-in commands** (e.g. `/clear`, `/model`, `/compact`) execute **fixed logic** directly.
- **Bundled skills** (e.g. `/code-review`, `/debug`, `/loop`, `/batch`, `/run`, `/verify`, `/claude-api`) are **prompt-based** -- they hand Claude detailed instructions and let it orchestrate with its tools. They can be disabled with `disableBundledSkills`, and you can *override* one by defining a project skill of the same name.
- **Bundled workflows** (e.g. `/deep-research`) fan work out across many subagents and run in the background.

So the practical line is this: a bundled skill behaves like very good prompting you didn't have to write yourself, which is exactly why it's overridable and tunable. A built-in command does not negotiate.

---

## 5. Custom commands & skills

**Custom commands have been merged into skills.** A file at `.claude/commands/deploy.md` and a skill at `.claude/skills/deploy/SKILL.md` both create `/deploy` and work the same way. Existing `.claude/commands/` files keep working; skills are the recommended form because they add a directory for supporting files, frontmatter to control invocation, and automatic (model-driven) loading. If a skill and a command share a name, the **skill wins.** [^30]

This chapter looks at custom commands from the driving angle, which is the slash command you type. The full skill model, with progressive disclosure and supporting files and lifecycle, is its own chapter. Here is the minimum you need to build commands that drive the agent.

### 5.1 Where they live and how the name is derived

| Location | Path | Scope |
|----------|------|-------|
| Personal | `~/.claude/skills/<name>/SKILL.md` | All your projects |
| Project | `.claude/skills/<name>/SKILL.md` (commit it) | This repo |
| Legacy command | `.claude/commands/<name>.md` | Same `/name` |
| Plugin | `<plugin>/skills/<name>/SKILL.md` | Where the plugin is enabled |

The command you type comes from the **directory name** (`.claude/skills/deploy-staging/` -> `/deploy-staging`), not the frontmatter `name` (which is just the display label). Nested skills in a monorepo namespace by path when names collide: `apps/web/.claude/skills/deploy/` -> `/apps/web:deploy`. Plugin skills namespace as `plugin:name`. Precedence on name clash: enterprise overrides personal, personal overrides project, and any of these overrides a bundled skill of the same name. [^31]

`/reload-skills` (v2.1.152+) re-scans the directories so a skill you just wrote becomes available without restarting; live change detection already covers edits to existing `SKILL.md` files. [^32]

### 5.2 The frontmatter that matters for driving

```yaml
---
name: deploy
description: Deploy the application to production
disable-model-invocation: true          # only YOU can trigger it
allowed-tools: Bash(git add *) Bash(git commit *) Bash(git status *)
argument-hint: [environment]
model: sonnet                            # override model while active
effort: high                            # override effort while active
---
```

The fields you reach for when building a command: [^33]

- **`disable-model-invocation: true`** -- the agent can't auto-trigger it; only `/name` does. Use it for anything with side effects (`/deploy`, `/commit`, `/send-slack-message`). You do not want Claude deciding to deploy because the code "looks ready." It also keeps the description *out of context* until you invoke it (a token saving), and prevents the skill from being preloaded into subagents.
- **`user-invocable: false`** -- the inverse: hidden from the `/` menu, Claude-only. For background knowledge that isn't a meaningful user action.
- **`allowed-tools`** -- pre-approve specific tools while the command is active (no per-use prompt). For project skills this takes effect only after you accept the workspace trust dialog -- review committed skills before trusting, since a skill can grant itself broad access.
- **`disallowed-tools`** -- remove tools from the pool while active (e.g. strip `AskUserQuestion` from a background loop so it can't stall). Clears on your next message.
- **`model` / `effort`** -- override per command, scoped to the rest of that turn; the session model/effort resumes on your next prompt.
- **`paths`** -- glob patterns that limit when the skill auto-loads (only when Claude works with matching files), the same format as path-specific memory rules.
- **`argument-hint`** -- autocomplete hint.

### 5.3 Argument substitution and dynamic context

**Arguments:** `$ARGUMENTS` (everything after the command), `$ARGUMENTS[N]` / `$N` (0-indexed positional; shell-style quoting, so wrap multi-word values), and `$name` (named positional args declared in `arguments:`). If you pass args but the body has no `$ARGUMENTS`, Claude Code appends `ARGUMENTS: <value>` so Claude still sees them. Escape a literal with a backslash (`\$1.00`). [^34]

Other substitutions: `${CLAUDE_SESSION_ID}`, `${CLAUDE_EFFORT}`, `${CLAUDE_SKILL_DIR}` (the skill's own dir, which you use to reference bundled scripts regardless of cwd).

**Dynamic context injection -- the power feature.** The `` !`<command>` `` syntax runs a shell command **before the skill content is sent to Claude** and replaces the placeholder with the output. This is preprocessing, not something Claude executes. Claude only ever sees the final rendered text, with real data already inlined:

```markdown
---
name: pr-summary
description: Summarize changes in a pull request
allowed-tools: Bash(gh *)
---

## Pull request context
- PR diff:      !`gh pr diff`
- PR comments:  !`gh pr view --comments`
- Changed files: !`gh pr diff --name-only`

## Your task
Summarize this pull request and flag anything risky.
```

The `!` form is recognized only at the start of a line or after whitespace (so `KEY=!`cmd`` is left literal). For multi-line, open a fenced block with ` ```! `. Output is inserted as plain text and **not re-scanned** for further placeholders, so a command cannot emit a placeholder for a later pass to expand. This is the cleanest way to ground a command in live state instead of hoping the agent re-derives it. Org-wide, `disableSkillShellExecution: true` neuters it, replacing each command with `[shell command execution disabled by policy]` -- a managed-settings control worth knowing for regulated environments. [^35]

### 5.4 Running a command in isolation

`context: fork` runs the command in a forked subagent. Its content becomes the subagent's prompt, it doesn't see your conversation history, and only the summary returns. Pair with `agent: Explore` (or `Plan`, `general-purpose`, or a custom agent) to pick the execution environment; if omitted, it uses `general-purpose`. Only use `context: fork` for skills with an *actual task*. A pure-guidelines skill forked into an agent has nothing to do and returns without meaningful output. [^36]

### 5.5 Lifecycle gotcha (carries over from skills)

An invoked command enters the conversation as a single message and **stays for the rest of the session. It is not re-read each turn.** Write standing instructions, not "do this now." After compaction, the most recent invocation of each skill is re-attached after the summary (first 5,000 tokens each, sharing a combined 25,000-token budget filled newest-first), so older ones can drop entirely if you invoked many. When a command "stops working," the content is usually still sitting right there and the model has simply chosen a different approach. Strengthen the wording, re-invoke after compaction, or enforce the rule with a **hook** for a deterministic guarantee. [^37]

---

## 6. Checkpoints & rewind -- and the edit-only limit

### 6.1 How checkpoints work

Claude Code **automatically captures the state of your code before each edit.** Properties to know: [^38]

- Every **user prompt** creates a new checkpoint.
- Checkpoints **persist across sessions** (accessible in resumed conversations).
- They're cleaned up with sessions after **30 days** (configurable via `cleanupPeriodDays`).
- They are **local to your session and separate from git** -- "local undo," with git as "permanent history." Each message, tool use, and result is written to a plaintext JSONL file under `~/.claude/projects/`, and Claude snapshots affected files before editing. [^39]

### 6.2 Rewinding

Run **`/rewind`** (aliases `/checkpoint`, `/undo`) or press **`Esc` twice when the input is empty** to open the rewind menu. (If the input has text, double-`Esc` clears it instead, and the cleared text is saved to history; press `Up` to recall.) The menu lists each prompt you sent; pick a point, then an action: [^40]

- **Restore code and conversation** -- revert both to that point.
- **Restore conversation** -- rewind the conversation, keep current code.
- **Restore code** -- revert files, keep the conversation.
- **Summarize from here** -- compress the selected message and everything after into a summary (discard a side discussion, keep early context in full).
- **Summarize up to here** -- compress everything *before* the selected message, keep later messages intact.
- **Never mind** -- back out.

After restoring the conversation (or "Summarize from here"), the original prompt is restored into the input field so you can edit and re-send. Summarize options *compress* without changing files on disk; the original messages stay in the transcript so Claude can still reference details. **Rewind vs. fork:** rewind/summarize keep you in the same session; if you want to try a different approach while preserving the original session, **fork** instead (`/branch` or `claude --continue --fork-session`). Forked sessions are grouped under their root in the `/resume` picker. [^41]

### 6.3 The critical limitation -- Bash side effects are permanent

This is the one fact from the chapter to over-learn. Checkpointing **only tracks files changed through Claude's file-editing tools.** It does **not** track files modified by Bash commands. The docs are explicit: [^42]

```bash
rm file.txt        # gone -- rewind cannot restore it
mv old.txt new.txt # not reversible via rewind
cp source.txt dest.txt
```

Anything with effects beyond a local file edit lives **outside the undo boundary entirely**, and those are exactly the operations that leave a mark:

- `git push` / `git push --force` (your mistake is now on the remote)
- database writes, migrations, `DROP TABLE`
- API/network calls (emails sent, webhooks fired, resources provisioned)
- package installs (`npm install`, `pip install`)
- file deletions/moves via shell rather than the edit tool

Anthropic ties this directly to permissions: *"Actions that affect remote systems (databases, APIs, deployments) can't be checkpointed, which is why Claude asks before running commands with external side effects."* [^43]

Two more boundaries: checkpoints don't capture **external/manual edits** (your own changes outside Claude Code, or another concurrent session) unless they touch the same files; and checkpoints are **not a replacement for version control.** [^44]

### 6.4 The operating discipline this implies

The edit-only limit is the reason every earlier section fits together. Because reversible edits cost you nothing to undo, you can run `acceptEdits` or auto mode freely on source changes. Because Bash side effects are permanent, you keep oversight high on the irreversible boundary, and there are layered tools built for exactly that:

1. **Commit early and often** so git, not just checkpoints, holds a restore point -- your real safety net for anything Bash touches.
2. **Deny rules** (`/permissions`) for the truly destructive: `Bash(git push --force *)`, `Bash(rm -rf *)`, `DROP TABLE`. These hold in every mode (even `bypassPermissions`), unlike a conversational "don't push."
3. **`PreToolUse` hooks** for deterministic guardrails the model cannot talk its way around.
4. **Auto mode's defaults already block** force-push, push-to-`main`, deploys, migrations, and mass deletion -- its design encodes this exact asymmetry.

The shorthand to keep in your head: rewind is for "the agent wrote bad code"; git plus deny rules plus hooks are for "the agent did something to the world." The two are not interchangeable.

---

## 7. `/goal` -- autonomous run-to-condition

### 7.1 What it does

`/goal <condition>` sets a completion condition and Claude **keeps working across turns** without you prompting each one. After each turn, a small fast model checks whether the condition holds; if not, Claude starts another turn instead of returning control. The goal clears automatically once met. **Requires v2.1.139+.** [^45]

```text
/goal all tests in test/auth pass and the lint step is clean
```

Setting a goal **starts a turn immediately** with the condition as the directive, so no separate prompt is needed. While active, a `(o) /goal active` indicator shows elapsed time, and the evaluator's most recent reason appears in the status view and transcript so you can see what the agent is working toward next. One goal per session; a new `/goal` replaces the old. The condition can be up to **4,000 characters.** [^46]

Good fits: migrating a module until every call site compiles and tests pass; implementing a design doc until all acceptance criteria hold; splitting a large file until each piece is under a size budget; working a labeled issue backlog until the queue is empty. [^47]

### 7.2 How evaluation works -- and its hard limit

`/goal` is a wrapper around a session-scoped **prompt-based Stop hook.** After each turn, the condition plus the conversation-so-far go to your configured small fast model (**defaults to Haiku**), which returns yes/no plus a short reason. A "no" tells Claude to keep working and feeds the reason in as guidance; a "yes" clears the goal and records an achieved entry. Evaluation tokens bill on the small fast model and are typically negligible against main-turn spend. [^48]

> **The limit that makes or breaks a goal:** the evaluator **does not call tools**, "so it can only judge what Claude has already surfaced in the conversation." It cannot run commands or read files independently. So the condition must be something Claude's own output can *demonstrate.* "All tests in `test/auth` pass" works because Claude runs the tests and the result lands in the transcript for the evaluator to read. A condition the evaluator would have to verify itself (by running something) cannot be judged correctly. [^48]

This separation is the whole point. The agent that does the work is not the agent that decides the work is done. Completion gets judged by a fresh model, not by the one that did the labor and would dearly love to call it finished.

### 7.3 Writing a condition that holds across many turns

A durable condition usually has three parts: [^49]

- **One measurable end state** -- a test result, a build exit code, a file count, an empty queue.
- **A stated check** -- *how* Claude should prove it: "`npm test` exits 0," "`git status` is clean."
- **Constraints that matter** -- what must *not* change on the way: "no other test file is modified."

Bound the runtime by putting a clause in the condition itself, for example `... or stop after 20 turns`. Claude reports progress against that clause each turn and the evaluator judges it from the conversation.

### 7.4 Status, clearing, resuming, headless

- **`/goal`** (no args) shows the active (or last-achieved) goal: condition, elapsed time, turns evaluated, token spend, latest reason.
- **`/goal clear`** (aliases `stop`, `off`, `reset`, `none`, `cancel`) removes an active goal early; `/clear` also removes it.
- **Resume:** a goal still active at session end is **restored on `--resume`/`--continue`** (condition carries over; turn count, timer, and token baseline reset). Achieved/cleared goals are not restored.
- **Headless:** `claude -p "/goal CHANGELOG.md has an entry for every PR merged this week"` runs the loop to completion in one invocation; `Ctrl+C` interrupts.

[^50]

### 7.5 `/goal` in context -- pick the right looping primitive

| Approach | Next turn starts when | Stops when |
|----------|----------------------|------------|
| `/goal` | The previous turn finishes | A model confirms the condition is met |
| `/loop` | A time interval elapses | You stop it, or Claude decides it's done |
| Stop hook | The previous turn finishes | Your own script/prompt decides |

`/goal` and auto mode are complementary, and it helps to see why they don't overlap: auto mode removes the per-tool prompts inside a turn, while `/goal` removes the per-turn prompts across turns. The configuration that runs unattended overnight is **`/goal` + auto mode**, where each goal turn runs without tool prompts and the evaluator decides when to stop. Pair that with deny rules and hooks (section 6.4) so "unattended" never quietly becomes "unguarded." [^51]

### 7.6 Requirements & gotchas

`/goal` runs only in **trusted workspaces** (the evaluator is part of the hooks system) and is unavailable when `disableAllHooks` is set at any level, or `allowManagedHooksOnly` is set in managed settings. In each case it tells you why rather than silently no-op'ing. [^52]

---

## 8. Headless basics -- `claude -p`

Headless, non-interactive mode is the backbone of scripted and CI use. Add **`-p`** (or `--print`) to any `claude` command to run it once, non-interactively. All CLI flags work with `-p`. [^53]

```bash
claude -p "What does the auth module do?"
claude -p "Find and fix the bug in auth.py" --allowedTools "Read,Edit,Bash"
```

### 8.1 The flags that matter

- **`--output-format`** -- `text` (default), `json` (structured: `result`, `session_id`, usage/cost metadata; `total_cost_usd` plus a per-model breakdown), or `stream-json` (newline-delimited events). [^54]
- **`--json-schema '<JSON Schema>'`** (with `--output-format json`) -- constrained structured output; the conforming object lands in `structured_output`.
- **`--output-format stream-json --verbose --include-partial-messages`** -- token-by-token streaming; each line is a JSON event (`system/init` first, `system/api_retry` on retryable failures, `system/plugin_install` when `CLAUDE_CODE_SYNC_PLUGIN_INSTALL` is set).
- **`--allowedTools` / `--disallowedTools`** -- auto-approve/remove tools using permission-rule syntax: `--allowedTools "Bash(git diff *),Read,Edit"`. The trailing ` *` (with the space) enables prefix matching; without the space, `Bash(git diff*)` would also match `git diff-index`.
- **`--permission-mode`** -- set a baseline instead of listing tools; `dontAsk` for locked-down CI, `acceptEdits` to let it write files. Other shell/network commands still need an `--allowedTools` entry or a `permissions.allow` rule, otherwise the run aborts.
- **`--continue` / `--resume <session-id>`** -- chain non-interactive calls. `-p` sessions don't appear in the `/resume` picker but resume by ID (scoped to the project dir + its worktrees).
- **`--append-system-prompt` / `--system-prompt`** -- append to, or fully replace, the system prompt for the run (append preserves default tool/safety guidance; replace drops all of it).
- **`--max-turns`** -- cap the agentic loop (print mode only; exits with an error at the limit).
- **`--max-budget-usd`** -- stop once API spend crosses a dollar threshold (print mode only).
- **`--no-session-persistence`** -- don't write a transcript (print mode only).
- **`--worktree`/`-w`** -- run in an isolated git worktree at `<repo>/.claude/worktrees/<name>`, the clean way to parallelize.

[^55]

### 8.2 `--bare` -- reproducibility for CI

`--bare` skips auto-discovery of hooks, skills, plugins, MCP servers, auto memory, and `CLAUDE.md`, so a teammate's `~/.claude` hook or the repo's `.mcp.json` **won't silently run.** Only flags you pass explicitly take effect, giving the same result on every machine. In bare mode Claude still has Bash, file read, and file edit tools; load anything else explicitly (`--append-system-prompt`, `--settings`, `--mcp-config`, `--agents`, `--plugin-dir`). Auth must come from `ANTHROPIC_API_KEY` or an `apiKeyHelper` (bare skips OAuth/keychain; Bedrock/Vertex/Foundry use their usual provider credentials). **Anthropic recommends `--bare` for all scripted/SDK calls, and "it will become the default for `-p` in a future release,"** so write CI as if it already is. [^56]

### 8.3 Piping and composition

`-p` reads stdin, so Claude composes like any Unix tool:

```bash
# Explain a build failure, write to a file
cat build-error.txt | claude -p 'concisely explain the root cause' > output.txt

# Claude as a typo linter in package.json (piping the diff avoids needing Bash perms)
"lint:claude": "git diff main | claude -p \"you are a typo linter. for each typo in this diff, report filename:line then the issue on the next line. return nothing else.\""

# Capture and resume a specific session
session_id=$(claude -p "Start a review" --output-format json | jq -r '.session_id')
claude -p "Continue that review" --resume "$session_id"

# Extract structured data with jq
claude -p "Extract function names from auth.py" --output-format json \
  --json-schema '{"type":"object","properties":{"functions":{"type":"array","items":{"type":"string"}}},"required":["functions"]}' \
  | jq '.structured_output'
```

Piped stdin is **capped at 10MB** (v2.1.128+); for larger inputs, write to a file and reference the path. Background Bash tasks started during a `-p` run are terminated ~5s after the final result and stdin close (v2.1.163+). Before that, a never-exiting process could hold the invocation open indefinitely. [^57]

### 8.4 Commands and goals work in `-p`

User-invoked **skills and custom commands work in `-p`**. Put `/skill-name` in the prompt string and Claude Code expands it before running. Built-in commands that open an interactive dialog (e.g. `/login`) are not available headless. From v2.1.181, `/config key=value` works in `-p` (e.g. `/config thinking=false`). And as covered in section 7, `claude -p "/goal ..."` runs an autonomous loop to completion. [^58]

> `claude -p` is the **CLI surface of the Agent SDK** -- the same tools, agent loop, and context management that power Claude Code. When you outgrow shell composition -- tool-approval callbacks, native message objects, programmatic subagents -- graduate to the Python or TypeScript SDK; the loop and tools are identical. The SDK itself is Ch. 11's territory. [^59]

---

## 9. Putting it together -- driving postures

The controls in this chapter compose into a handful of repeatable postures. A senior's fluency shows up as moving between them on purpose.

| Posture | Configuration | When |
|---------|--------------|------|
| **Explore & align** | `plan` mode -> review plan -> `Ctrl+G` to edit | New problem; before any large change |
| **Reviewed velocity** | `acceptEdits` + `git diff` after | Source changes you'll review as one unit |
| **Guarded autonomy** | `auto` mode + explicit boundaries + **deny rules / hooks** for the irreversible | Long tasks you trust the direction of |
| **Run-to-done** | `/goal <measurable condition>` (+ auto mode for hands-off) | Verifiable end state; you want a fresh model to judge "done" |
| **Scripted / CI** | `claude -p --bare --permission-mode dontAsk --output-format json` | Reproducible pipelines, linters, gates |
| **Recovery** | `/rewind` for edits - git + deny rules for everything Bash touched | After a wrong turn |

One thread runs through all six: reversible work can be cheap, and irreversible work has to stay guarded. Plan mode forces commitments into a reviewable artifact. Permission modes set the default friction. Checkpoints make edit-mistakes free to undo. Deny rules and hooks fence the operations checkpoints can't undo. `/goal` lets a separate evaluator decide completion, and headless lets you encode the whole posture in a script. Drive with that asymmetry in your hands and the tool stops feeling risky.

---

## Sources

- https://code.claude.com/docs/en/how-claude-code-works
- https://code.claude.com/docs/en/permission-modes
- https://code.claude.com/docs/en/commands
- https://code.claude.com/docs/en/cli-reference
- https://code.claude.com/docs/en/skills
- https://code.claude.com/docs/en/ultraplan
- https://code.claude.com/docs/en/checkpointing
- https://code.claude.com/docs/en/goal
- https://code.claude.com/docs/en/headless
- https://code.claude.com/docs/en/changelog
- https://www.anthropic.com/engineering/claude-code-auto-mode

[^1]: web | 2026-06-18 | [code.claude.com/docs/en/commands](https://code.claude.com/docs/en/commands) -- /release-notes
[^2]: web | 2026-06-18 | [code.claude.com/docs/en/how-claude-code-works](https://code.claude.com/docs/en/how-claude-code-works) -- "The agentic loop", "Interrupt and steer"
[^3]: web | 2026-06-18 | [code.claude.com/docs/en/permission-modes](https://code.claude.com/docs/en/permission-modes) -- "Analyze before you edit with plan mode"
[^4]: web | 2026-06-18 | [code.claude.com/docs/en/how-claude-code-works](https://code.claude.com/docs/en/how-claude-code-works) -- "Explore before implementing": "This two-phase approach produces better results than jumping straight to code."
[^5]: web | 2026-06-18 | [code.claude.com/docs/en/permission-modes](https://code.claude.com/docs/en/permission-modes) -- "Switch permission modes"
[^6]: web | 2026-06-18 | [code.claude.com/docs/en/commands](https://code.claude.com/docs/en/commands) -- /plan
[^7]: web | 2026-06-18 | [code.claude.com/docs/en/cli-reference](https://code.claude.com/docs/en/cli-reference) -- --permission-mode
[^8]: web | 2026-06-18 | [code.claude.com/docs/en/permission-modes](https://code.claude.com/docs/en/permission-modes) -- "Set plan mode as the default"
[^9]: web | 2026-06-18 | [code.claude.com/docs/en/permission-modes](https://code.claude.com/docs/en/permission-modes) -- "Review and approve a plan"
[^10]: web | 2026-06-18 | [code.claude.com/docs/en/ultraplan](https://code.claude.com/docs/en/ultraplan) -- "Launch ultraplan from the CLI", "Choose where to execute"
[^11]: web | 2026-06-18 | [code.claude.com/docs/en/skills](https://code.claude.com/docs/en/skills) -- "Run skills in a subagent": built-in Explore and Plan agents "skip CLAUDE.md and git status to keep their context small"
[^12]: web | 2026-06-18 | [code.claude.com/docs/en/permission-modes](https://code.claude.com/docs/en/permission-modes) -- "Available modes"
[^13]: web | 2026-06-18 | [code.claude.com/docs/en/permission-modes](https://code.claude.com/docs/en/permission-modes) -- "Eliminate prompts with auto mode" and "Skip all checks with bypassPermissions mode"
[^14]: web | 2026-06-18 | [code.claude.com/docs/en/permission-modes](https://code.claude.com/docs/en/permission-modes) -- "Auto-approve file edits with acceptEdits mode"
[^15]: web | 2026-06-18 | [code.claude.com/docs/en/permission-modes](https://code.claude.com/docs/en/permission-modes) -- "Eliminate prompts with auto mode"
[^16]: web | 2026-06-18 | [code.claude.com/docs/en/permission-modes](https://code.claude.com/docs/en/permission-modes) -- "What the classifier blocks by default"; code.claude.com/docs/en/cli-reference -- claude auto-mode defaults
[^17]: web | 2026-06-18 | [code.claude.com/docs/en/permission-modes](https://code.claude.com/docs/en/permission-modes) -- "Boundaries you state in conversation"
[^18]: web | 2026-06-18 | [code.claude.com/docs/en/permission-modes](https://code.claude.com/docs/en/permission-modes) -- "When auto mode falls back"
[^19]: web | 2026-06-18 | [code.claude.com/docs/en/permission-modes](https://code.claude.com/docs/en/permission-modes) -- "How the classifier evaluates actions"; deep dive at anthropic.com/engineering/claude-code-auto-mode
[^20]: web | 2026-06-18 | [code.claude.com/docs/en/permission-modes](https://code.claude.com/docs/en/permission-modes) -- "How auto mode handles subagents"
[^21]: web | 2026-06-18 | [code.claude.com/docs/en/permission-modes](https://code.claude.com/docs/en/permission-modes) -- "Allow only pre-approved tools with dontAsk mode"
[^22]: web | 2026-06-18 | [code.claude.com/docs/en/permission-modes](https://code.claude.com/docs/en/permission-modes) -- "Skip all checks with bypassPermissions mode"
[^23]: web | 2026-06-18 | [code.claude.com/docs/en/permission-modes](https://code.claude.com/docs/en/permission-modes) -- bypassPermissions warning callout
[^24]: web | 2026-06-18 | [code.claude.com/docs/en/permission-modes](https://code.claude.com/docs/en/permission-modes) -- "Protected paths"
[^25]: web | 2026-06-18 | [code.claude.com/docs/en/commands](https://code.claude.com/docs/en/commands) -- intro
[^26]: web | 2026-06-18 | [code.claude.com/docs/en/commands](https://code.claude.com/docs/en/commands) -- "All commands"
[^27]: web | 2026-06-18 | [code.claude.com/docs/en/cli-reference](https://code.claude.com/docs/en/cli-reference) -- --safe-mode
[^28]: web | 2026-06-18 | [code.claude.com/docs/en/commands](https://code.claude.com/docs/en/commands) -- /vim "Removed in v2.1.92", /pr-comments "Removed in v2.1.91"
[^29]: web | 2026-06-18 | [code.claude.com/docs/en/commands](https://code.claude.com/docs/en/commands) -- "All commands" legend; code.claude.com/docs/en/skills -- "Bundled skills"
[^30]: web | 2026-06-18 | [code.claude.com/docs/en/skills](https://code.claude.com/docs/en/skills) -- "Custom commands have been merged into skills"
[^31]: web | 2026-06-18 | [code.claude.com/docs/en/skills](https://code.claude.com/docs/en/skills) -- "Where skills live", "How a skill gets its command name"
[^32]: web | 2026-06-18 | [code.claude.com/docs/en/commands](https://code.claude.com/docs/en/commands) -- /reload-skills "Added in v2.1.152"
[^33]: web | 2026-06-18 | [code.claude.com/docs/en/skills](https://code.claude.com/docs/en/skills) -- "Frontmatter reference"
[^34]: web | 2026-06-18 | [code.claude.com/docs/en/skills](https://code.claude.com/docs/en/skills) -- "Available string substitutions", "Pass arguments to skills"
[^35]: web | 2026-06-18 | [code.claude.com/docs/en/skills](https://code.claude.com/docs/en/skills) -- "Inject dynamic context"
[^36]: web | 2026-06-18 | [code.claude.com/docs/en/skills](https://code.claude.com/docs/en/skills) -- "Run skills in a subagent"
[^37]: web | 2026-06-18 | [code.claude.com/docs/en/skills](https://code.claude.com/docs/en/skills) -- "Skill content lifecycle"
[^38]: web | 2026-06-18 | [code.claude.com/docs/en/checkpointing](https://code.claude.com/docs/en/checkpointing) -- "How checkpoints work"
[^39]: web | 2026-06-18 | [code.claude.com/docs/en/how-claude-code-works](https://code.claude.com/docs/en/how-claude-code-works) -- "Work with sessions", "Undo changes with checkpoints"
[^40]: web | 2026-06-18 | [code.claude.com/docs/en/checkpointing](https://code.claude.com/docs/en/checkpointing) -- "Rewind and summarize"; code.claude.com/docs/en/commands -- /rewind aliases
[^41]: web | 2026-06-18 | [code.claude.com/docs/en/checkpointing](https://code.claude.com/docs/en/checkpointing) -- "Restore vs. summarize" and fork callout
[^42]: web | 2026-06-18 | [code.claude.com/docs/en/checkpointing](https://code.claude.com/docs/en/checkpointing) -- "Bash command changes not tracked"
[^43]: web | 2026-06-18 | [code.claude.com/docs/en/how-claude-code-works](https://code.claude.com/docs/en/how-claude-code-works) -- "Undo changes with checkpoints"
[^44]: web | 2026-06-18 | [code.claude.com/docs/en/checkpointing](https://code.claude.com/docs/en/checkpointing) -- "External changes not tracked", "Not a replacement for version control"
[^45]: web | 2026-06-18 | [code.claude.com/docs/en/goal](https://code.claude.com/docs/en/goal) -- intro
[^46]: web | 2026-06-18 | [code.claude.com/docs/en/goal](https://code.claude.com/docs/en/goal) -- "Set a goal", "Write an effective condition"
[^47]: web | 2026-06-18 | [code.claude.com/docs/en/goal](https://code.claude.com/docs/en/goal) -- intro list
[^48]: web | 2026-06-18 | [code.claude.com/docs/en/goal](https://code.claude.com/docs/en/goal) -- "How evaluation works"
[^49]: web | 2026-06-18 | [code.claude.com/docs/en/goal](https://code.claude.com/docs/en/goal) -- "Write an effective condition"
[^50]: web | 2026-06-18 | [code.claude.com/docs/en/goal](https://code.claude.com/docs/en/goal) -- "Check status", "Clear a goal", "Resume with an active goal", "Run non-interactively"
[^51]: web | 2026-06-18 | [code.claude.com/docs/en/goal](https://code.claude.com/docs/en/goal) -- "Compare ways to keep a session running"
[^52]: web | 2026-06-18 | [code.claude.com/docs/en/goal](https://code.claude.com/docs/en/goal) -- "Requirements"
[^53]: web | 2026-06-18 | [code.claude.com/docs/en/headless](https://code.claude.com/docs/en/headless) -- "Basic usage"
[^54]: web | 2026-06-18 | [code.claude.com/docs/en/headless](https://code.claude.com/docs/en/headless) -- "Get structured output"
[^55]: web | 2026-06-18 | [code.claude.com/docs/en/cli-reference](https://code.claude.com/docs/en/cli-reference) -- "CLI flags"; code.claude.com/docs/en/headless -- "Stream responses"
[^56]: web | 2026-06-18 | [code.claude.com/docs/en/headless](https://code.claude.com/docs/en/headless) -- "Start faster with bare mode"
[^57]: web | 2026-06-18 | [code.claude.com/docs/en/headless](https://code.claude.com/docs/en/headless) -- "Pipe data through Claude", "Background tasks at exit"
[^58]: web | 2026-06-18 | [code.claude.com/docs/en/headless](https://code.claude.com/docs/en/headless) -- "Create a commit" note
[^59]: web | 2026-06-18 | [code.claude.com/docs/en/headless](https://code.claude.com/docs/en/headless) -- "Run Claude Code programmatically" intro
