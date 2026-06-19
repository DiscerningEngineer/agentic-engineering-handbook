# Chapter 07 -- Subagents and Agent Types

**TL;DR.** A subagent is a worker that runs in its own context window with its own system prompt, tool set, model, and permissions, and hands back only a summary. The whole game is context isolation. You keep the verbose middle of a task out of your main conversation, the log scans and test runs and file dumps and parallel research, so only the conclusion ever lands in front of you. Claude Code ships three you will reach for constantly: Explore (Haiku, read-only, fast search), Plan (read-only research during plan mode), and general-purpose (all tools). On top of those you write your own as Markdown files in `.claude/agents/`. The moves that separate someone who delegates from someone who orchestrates are these: send cheap, bounded work to small models and keep your strongest model at the helm; fan out work that is genuinely independent and synthesize what comes back; understand the five-level nesting cap and what a level-five worker gives up; reach for forks when a worker needs the whole conversation and you want cheaper prompt-cache reuse; and know the handful of tools a subagent never gets, which turns out to be the real reason an agent cannot ask the user. Everything pinned to a version here came from a product that ships almost every day, so check the changelog before you teach it. [^1]

---

## 7.1 What a subagent actually is

A subagent runs "in its own context window with a custom system prompt, specific tool access, and independent permissions. When Claude encounters a task that matches a subagent's description, it delegates to that subagent, which works independently and returns results." [^2]

Here is the mental model that earns its keep. The main conversation is a budget you pay for on every single turn, and a subagent is how you do expensive work off-budget. When some side task is about to flood your window with search results, logs, or file contents you will never look at again, the subagent does that work in its own window and hands you back the distilled answer. The docs put it plainly: use one "when a side task would flood your main conversation with search results, logs, or file contents you won't reference again." Anthropic frames the five things this buys you like so. [^3]

- **Preserve context** -- keep exploration and implementation out of the main conversation.
- **Enforce constraints** -- limit which tools a subagent can use.
- **Reuse configurations** -- user-level subagents follow you across projects.
- **Specialize behavior** -- focused system prompts for specific domains.
- **Control costs** -- route tasks to faster, cheaper models like Haiku.

### The isolation boundary (read this twice)

A non-fork subagent starts with a fresh, isolated context window. It does not see your conversation history, the skills you have already invoked, or the files Claude has already read. Claude composes a delegation message that summarizes the task, and the subagent works from there and nothing else. [^4]

This cuts both ways, and it is the single most common source of "why did my subagent ignore my rule?" A non-fork subagent's initial context contains exactly: [^4]

1. **System prompt** -- the agent's own prompt plus environment details Claude Code appends (working directory, etc.), *not* the full Claude Code system prompt.
2. **Task message** -- the delegation prompt Claude wrote at hand-off.
3. **CLAUDE.md and memory** -- every level of the memory hierarchy the main conversation loads (`~/.claude/CLAUDE.md`, project rules, `CLAUDE.local.md`, managed policy). The hierarchy itself lives in Ch. 03. **Exception: the built-in Explore and Plan agents skip this** to stay fast and cheap.
4. **Git status** -- a snapshot taken at the start of the *parent* session. Absent when not a git repo or when `includeGitInstructions` is false. Explore and Plan skip it too.
5. **Preloaded skills** -- full content of any skill named in the agent's `skills` field. Built-in agents preload nothing.

What this means for teaching: if a rule has to reach an Explore or Plan subagent, something like "ignore the `vendor/` directory," you have to restate it in the prompt you hand Claude at the moment you delegate. It will not arrive through CLAUDE.md. The main conversation does read Explore and Plan results back with full CLAUDE.md context, so most of your rules never need to travel into the subagent at all. [^4]

> **Distinguish three nearby things.** Subagents live *within a single session*. Running many *independent sessions* in parallel and watching them from one place is **background agents** (see `/en/agent-view`). Sessions that *talk to each other as peers* are **agent teams** (the canonical home is Ch. 10, Orchestration). Keep them apart in your head. [^5]

---

## 7.2 The built-in subagents

Claude Code "includes built-in subagents that Claude automatically uses when appropriate. Each inherits the parent conversation's permissions with additional tool restrictions." [^6]

| Built-in | Model | Tools | Loads CLAUDE.md + git status? | Purpose |
|---|---|---|---|---|
| **Explore** | Haiku (fast, low-latency) | Read-only (denied Write/Edit) | **No** (skips for speed) | File discovery, code search, codebase exploration |
| **Plan** | Inherits main conversation | Read-only (denied Write/Edit) | **No** (skips for speed) | Codebase research during plan mode |
| **general-purpose** | Inherits main conversation | **All tools** | Yes | Complex multi-step work needing exploration *and* modification |
| **statusline-setup** | Sonnet | (helper) | Yes | Fires when you run `/statusline` |
| **claude-code-guide** | Haiku | (helper) | Yes | Fires when you ask about Claude Code features |

[^7]

A few details are worth carrying around with you.

Explore takes a thoroughness level. When Claude invokes it, it specifies quick (targeted lookups), medium (balanced exploration), or very thorough (comprehensive analysis). That is your lever for choosing between a cheap grep-and-report and a deep survey. [^8]

Plan is the thing that lets plan mode stay read-only without going blind. When Claude needs to actually understand your codebase, it hands the research to the Plan subagent "so that exploration output stays in a separate context window while the main conversation remains read-only." That is the architectural reason plan mode can do heavy research without ballooning your main window. [^9]

Explore and Plan are both one-shot and return no agent ID, which means they cannot be resumed. The moment you need to continue a worker's thread, reach for `general-purpose` or a custom subagent, because those return IDs. [^10]

### Disabling built-ins

Built-ins are always registered in interactive sessions. To block one type, add it to `permissions.deny`: [^11]

```json
{ "permissions": { "deny": ["Agent(Explore)", "Agent(my-custom-agent)"] } }
```

To stop *all* delegation, deny the `Agent` tool itself. In non-interactive mode and the Agent SDK, `CLAUDE_AGENT_SDK_DISABLE_BUILTIN_AGENTS=1` removes every built-in type so you can supply only your own. [^12]

---

## 7.3 Defining your own subagent

A subagent is a Markdown file with YAML frontmatter, and the body is the system prompt. Here is the minimal version. [^13]

```markdown
---
name: code-reviewer
description: Reviews code for quality and best practices
tools: Read, Glob, Grep
model: sonnet
---

You are a code reviewer. When invoked, analyze the code and provide
specific, actionable feedback on quality, security, and best practices.
```

Only `name` and `description` are required. The body becomes the system prompt, and the subagent receives only this plus basic env details, never the full Claude Code system prompt. [^13]

### Scope and precedence

Where the file lives decides who can use it and who wins when names collide. Higher priority overrides lower. [^14]

| Location | Scope | Priority |
|---|---|---|
| Managed settings (`.claude/agents/` in the managed dir) | Organization-wide | 1 (highest) |
| `--agents` CLI flag (JSON) | Current session only | 2 |
| `.claude/agents/` | Current project | 3 |
| `~/.claude/agents/` | All your projects | 4 |
| Plugin's `agents/` directory | Where plugin is enabled | 5 (lowest) |

[^15]

Project subagents in `.claude/agents/` belong in version control so the team uses them and sharpens them together. Claude discovers them by walking *up* from the working directory, scanning every `.claude/agents/` between cwd and repo root. As of v2.1.178, when nested directories define the same `name`, the definition closest to the working directory wins. [^16]

Both `.claude/agents/` and `~/.claude/agents/` get scanned recursively, so you are free to organize into subfolders like `agents/review/` and `agents/research/`. Identity comes only from the `name` field. The subfolder path has no effect on how a subagent is identified or invoked. Keep `name` values unique across the tree, because two files in one scope sharing a name means Claude Code "keeps one and discards the other without warning." [^14]

Plugin subagents are the exception to that path rule. There a subfolder *does* become part of the scoped identifier, so `agents/review/security.md` in plugin `my-plugin` registers as `my-plugin:review:security`. One more constraint for regulated shops: plugin subagents ignore the `hooks`, `mcpServers`, and `permissionMode` fields entirely for security reasons; if you need them, copy the agent file into `.claude/agents/` or `~/.claude/agents/`. [^17]

CLI-defined subagents passed through `--agents '{...}'` exist only for that session and never touch disk, which makes them handy for testing and automation scripts. The JSON accepts the same fields as the files, using `prompt` for the system prompt. [^18]

> **Editing gotcha.** Subagents are loaded at session start. If you edit a subagent file *directly on disk*, **restart the session** to pick it up. Subagents created or edited through the `/agents` interface take effect immediately. [^19]

### The `/agents` command

`/agents` opens a tabbed manager. The **Running** tab lists live and recently finished subagents, where you can open or stop them. The **Library** tab lets you view all available subagents (built-in, user, project, plugin), create new ones (guided setup or generate-with-Claude), edit config and tool access, delete custom ones, and see which wins when duplicates exist. This is the recommended path for creating and managing subagents. [^20]

---

## 7.4 Full frontmatter reference (the config knobs)

Every field below can also be passed via `--agents` JSON. Only `name` and `description` are required. [^21]

| Field | What it does |
|---|---|
| `name` | Unique id, lowercase + hyphens. Hooks receive it as `agent_type`. Filename need not match. |
| `description` | When Claude should delegate here. This is the **discoverability surface** -- write it for matching. |
| `tools` | Allowlist of tools. **Inherits all if omitted.** To preload skills use `skills`, not `Skill` here. |
| `disallowedTools` | Denylist, removed from the inherited or specified set. |
| `model` | `sonnet`, `opus`, `haiku`, `fable`, a full ID (e.g. `claude-opus-4-8`), or `inherit`. **Defaults to `inherit`.** |
| `permissionMode` | `default`, `acceptEdits`, `auto`, `dontAsk`, `bypassPermissions`, or `plan`. |
| `maxTurns` | Max agentic turns before the subagent stops. A hard budget cap. |
| `skills` | Skills to **preload** (full content injected at startup). Subagent can still invoke unlisted skills via the Skill tool. |
| `mcpServers` | MCP servers for this subagent only -- by name (shares parent connection) or inline (connected at start, disconnected at finish). |
| `hooks` | Lifecycle hooks scoped to this subagent. |
| `memory` | `user`, `project`, or `local` -- persistent cross-session learning directory. |
| `background` | `true` = always run as a background task. Default `false`. |
| `effort` | `low`/`medium`/`high`/`xhigh`/`max` (available levels depend on model). Overrides session effort. |
| `isolation` | `worktree` = run in a temporary git worktree, branched by default from your **default branch** (not the parent's `HEAD`). Auto-cleaned if no changes. |
| `color` | Display color in the task list and transcript (`red`, `blue`, `green`, `yellow`, `purple`, `orange`, `pink`, `cyan`). |
| `initialPrompt` | Auto-submitted as the first user turn when the agent runs as the *main* session (via `--agent`/`agent` setting). |

[^22]

### Tool restriction mechanics (precise rules)

By default a subagent inherits the internal tools and MCP tools of the main conversation. You have two levers, and they interact in a defined way. [^23]

- **`tools`** is an allowlist. **`disallowedTools`** is a denylist.
- **If both are set, `disallowedTools` is applied first, then `tools` is resolved against the remaining pool.** A tool in both is removed.
- Both accept MCP **server-level patterns**: `mcp__<server>` or `mcp__<server>__*` grant or remove every tool from that server; in `disallowedTools`, `mcp__*` removes *all* MCP tools from any server.

```yaml
# Inherit everything except file writes -- keeps Bash, MCP, etc.
---
name: no-writes
description: Inherits every tool except file writes
disallowedTools: Write, Edit
---
```

```yaml
# Strict allowlist -- read/search/run only, no edits, no MCP
---
name: safe-researcher
description: Research agent with restricted capabilities
tools: Read, Grep, Glob, Bash
---
```

### Restricting which *subagents* an agent can spawn

When an agent runs as the main thread via `claude --agent`, it can spawn subagents using the Agent tool. Use `Agent(agent_type)` in `tools` to allowlist spawnable types: [^24]

```yaml
tools: Agent(worker, researcher), Read, Bash   # only worker & researcher spawnable
tools: Agent, Read, Bash                        # any subagent spawnable
# (Agent omitted entirely -> cannot spawn any subagents)
```

> **Sharp edge -- a real pitfall.** The `Agent(agent_type)` allowlist syntax "applies only to an agent running as the main thread with `claude --agent`." Inside an ordinary **subagent definition**, listing `Agent` lets it spawn nested subagents, but "any type list inside the parentheses is ignored." You think you have restricted spawning, and you have not. To block specific agents universally, use `permissions.deny` instead. [^25]

> **Naming note.** In **v2.1.63** the `Task` tool was renamed to **`Agent`**. Existing `Task(...)` references in settings and agent definitions still work as aliases, but write new config against `Agent`. [^26]

### Scoping MCP servers to one subagent

`mcpServers` does double duty. It grants a subagent access to servers the main conversation does not have, and it also keeps a server's tool descriptions out of the main window. "To keep an MCP server out of the main conversation entirely and avoid its tool descriptions consuming context there, define it inline here rather than in `.mcp.json`. The subagent gets the tools; the parent conversation does not." [^27]

```yaml
---
name: browser-tester
description: Tests features in a real browser using Playwright
mcpServers:
  - playwright:                 # inline: scoped to this subagent only
      type: stdio
      command: npx
      args: ["-y", "@playwright/mcp@latest"]
  - github                      # by name: reuses an already-configured server
---
Use the Playwright tools to navigate, screenshot, and interact with pages.
```

As of v2.1.153, the MCP restrictions that govern the main session (`--strict-mcp-config`, `--bare`, enterprise managed MCP config, `allowedMcpServers`/`deniedMcpServers`) also cover servers declared in subagent frontmatter. One exception: `--strict-mcp-config` does *not* filter servers passed inline via `--agents` or the SDK `agents` option, since those are explicit caller input. [^28]

### Permission modes inside subagents

A subagent inherits the parent's permission context and may override the mode, except where the parent takes precedence: [^29]

- If the parent uses `bypassPermissions` or `acceptEdits`, **that wins** and cannot be overridden by the subagent.
- If the parent is in **auto mode**, the subagent inherits auto mode and its `permissionMode` frontmatter is **ignored** -- the classifier evaluates its tool calls with the parent's block/allow rules.
- As of **v2.1.178**, auto mode evaluates **subagent spawns themselves** with the classifier *before launch*, closing a gap where a subagent could request a blocked action without review. [^30]

> **`bypassPermissions` warning** (worth quoting verbatim to a cohort): it skips prompts and allows writes to `.git`, `.config/git`, `.claude`, `.vscode`, `.idea`, `.husky`, `.cargo`, `.devcontainer`, `.yarn`, `.mvn`. Explicit `ask` rules and root/home removals like `rm -rf /` still prompt. Use only in containers or VMs. [^31]

### Preloading skills, persistent memory, and conditional hooks

`skills` injects the full content of named skills at startup, so the subagent walks in with domain knowledge already in hand rather than discovering it at runtime. It governs what is preloaded, not what is accessible, and the subagent can still invoke other project, user, or plugin skills through the Skill tool. To forbid skills entirely, drop `Skill` from `tools` or add it to `disallowedTools`. You cannot preload a skill marked `disable-model-invocation: true`. [^32]

`memory` hands the subagent a persistent directory that survives across conversations, accumulating codebase patterns, debugging insights, and architectural decisions as it goes. The scopes are `user` -> `~/.claude/agent-memory/<name>/`, `project` -> `.claude/agent-memory/<name>/` (shareable via VCS, the recommended default), and `local` -> `.claude/agent-memory-local/<name>/` (gitignored). When it is enabled, the subagent's system prompt gets memory read/write instructions plus the first 200 lines or 25KB of its `MEMORY.md` (whichever comes first), and Read/Write/Edit are auto-enabled. This is how you grow a code-reviewer that gets sharper at *your* codebase over weeks. [^33]

Conditional hooks are the right tool when the `tools` allowlist is too blunt an instrument. Picture a `Bash`-enabled subagent that must run read-only SQL. A `PreToolUse: Bash` hook validates each command and `exit 2` blocks writes (`INSERT|UPDATE|DELETE|DROP|...`). That is finer-grained than tool gating, and it is the canonical pattern for allowing some operations of a tool while blocking others. [^34]

```yaml
---
name: db-reader
description: Execute read-only database queries. Use when analyzing data or generating reports.
tools: Bash
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/validate-readonly-query.sh"
---
```

In subagent frontmatter, a `Stop` hook is automatically converted to a `SubagentStop` event at runtime. You can also wire `SubagentStart`/`SubagentStop` hooks in `settings.json` to react in the *main* session when any (or a named) subagent begins or completes, opening a DB connection on start and tearing it down on stop, for instance. [^35]

---

## 7.5 Delegation: how Claude decides, and how you force its hand

Claude delegates "based on the task description in your request, the `description` field in subagent configurations, and current context." You have two levers. [^36]

The first is the `description` itself, which is the field Claude actually reads to decide. Write it for matching. Adding "use proactively" or "use immediately after writing code" nudges delegation without you having to ask. [^36]

The second lever is escalating explicitness when auto-delegation falls short, and it comes in three rungs: [^37]

- **Natural language** -- name the subagent in your prompt (`Use the test-runner subagent to fix failing tests`). Claude *decides* whether to delegate.
- **@-mention** -- `@"code-reviewer (agent)" look at the auth changes` (or type `@agent-<name>`). **Guarantees** that subagent runs for one task. Your full message still goes to Claude, which *writes the subagent's task prompt*; the @-mention controls *which* subagent, not the prompt.
- **Session-wide** -- `claude --agent code-reviewer` makes the *main thread itself* take on that subagent's system prompt, tool restrictions, and model. The subagent's prompt **replaces** the default Claude Code system prompt entirely (CLAUDE.md and project memory still load via the normal message flow). Persist it per-project with `{ "agent": "code-reviewer" }` in `.claude/settings.json`; the CLI flag overrides the setting.

### Resuming subagents

Each invocation creates a new instance with fresh context. To continue prior work, ask Claude to resume, and a resumed subagent retains its full history of tool calls, results, and reasoning, picking up exactly where it stopped. Mechanically, when a subagent completes Claude receives its agent ID and resumes via the `SendMessage` tool (with the ID as `to`). The caveats matter here. Explore and Plan are one-shot, return no ID, and cannot be resumed, so use `general-purpose` or a custom subagent for resumable work. `SendMessage` is only available when agent teams are enabled (`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`). Transcripts live at `~/.claude/projects/{project}/{sessionId}/subagents/agent-{agentId}.jsonl` and persist independently of main-conversation compaction (compaction internals are Ch. 02), cleaned up per `cleanupPeriodDays`, default 30. [^10]

A long-running subagent compacts its own window on the same triggers and logic the main conversation uses, and `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` reaches into subagents too. So a deep worker does not just blow past its budget and die; it folds its own context the way the main session does (the mechanics of that fold are Ch. 02). [^38]

### `/btw` -- the sub-question that isn't a subagent

For a quick question about something already sitting in your conversation, reach for `/btw` instead of spawning a worker. It sees your full context, has no tool access, and its answer is discarded rather than added to history. Cheaper and cleaner than standing up a subagent just to ask "wait, what does this function return again?" [^39]

---

## 7.6 Cost control via model routing

This is where senior judgment shows up most plainly. Every subagent opens its own window, so tokens multiply fast. Anthropic's own multi-agent research reports that "agents typically use about 4x more tokens than chat interactions, and multi-agent systems use about 15x more tokens than chats." The full set of orchestration figures is Ch. 10's territory; what you carry into a subagent design session is the order of magnitude: a swarm is not a rounding error on your bill. [^40]

The discipline is straightforward to state and harder to hold. Route cheap, bounded, parallelizable work to small models, and keep the expensive model for the orchestrator and the genuinely hard reasoning. Search, format and lint checks, log triage, first-pass review, and doc fetches all go to Haiku. Synthesis, architecture, and the actual fix stay with your strongest model at the helm. This is exactly the "control costs by routing tasks to faster, cheaper models like Haiku" framing the docs lead with, applied with the discernment to know which work is which. [^3]

### Model resolution order (memorize this)

When Claude invokes a subagent, the model is resolved in this order: [^41]

1. **`CLAUDE_CODE_SUBAGENT_MODEL`** env var (if set) -- **overrides everything below**, including per-invocation and frontmatter.
2. The **per-invocation `model`** parameter Claude passes.
3. The subagent definition's **`model` frontmatter**.
4. The **main conversation's model** (the `inherit` default).

> **The `CLAUDE_CODE_SUBAGENT_MODEL` trap.** The env var is "the model to use for all subagents and agent teams" and "overrides the per-invocation `model` parameter and the subagent definition's `model` frontmatter." Read that literally before you set it to `haiku` for "maximum savings": it flattens *every* subagent and every agent-team teammate to one model, including the planning and research workers whose output the rest of the run depends on. A starved planner compounds into downstream mistakes. The safer move is to leave it unset (or set it to `inherit` to force normal resolution) and choose the model per-agent in frontmatter, reaching for Haiku where the work is mechanical. [^42]

### Per-agent model choices that pay off

These are senior judgment calls, not Anthropic prescriptions. The pattern beneath them is the docs' own routing principle: cheap models for bounded mechanical work, capable models where reasoning lives.

| Subagent role | Model | Why |
|---|---|---|
| Codebase search / file discovery | **Haiku** (this is what Explore uses) | Bounded, mechanical, high-throughput |
| Lint / format / "did the build pass" checks | **Haiku** | Pass/fail, no deep reasoning |
| First-pass / mechanical code review | **Haiku, escalating to Sonnet** | Cheap triage, escalate the findings |
| Research / planning subagents | **Sonnet** | Planner quality compounds -- don't starve it |
| The orchestrator + hard debugging / architecture | **Opus / Fable 5** | Where judgment lives |

The `effort` knob stacks on top of model choice. A Haiku searcher at `low` effort is about the cheapest unit of useful work you can buy, while a debugging subagent at `xhigh` buys you depth right where it matters. Effort resolution is its own precedence chain: the `CLAUDE_CODE_EFFORT_LEVEL` env var "takes precedence over all other methods, then your configured level, then the model default," and frontmatter `effort` "applies when that skill or subagent is active, overriding the session level but not the environment variable." [^43]

For the regulated cohorts, one enterprise note. `availableModels` (managed/policy settings) is enforced on subagent models too: the allowlist applies to "the `model` field in subagent frontmatter, the Agent tool's `model` parameter, the model picker in `/agents`, and `CLAUDE_CODE_SUBAGENT_MODEL`," and "a blocked subagent or advisor override falls back to the inherited or default model rather than failing the request." [^44]

---

## 7.7 Fan-out, nesting, and forks

These are the three structural patterns. Learn when each one applies and the rest follows.

### Fan-out (parallel subagents, then synthesize)

For investigations that do not depend on each other, spawn several subagents at once and let Claude pull their answers together: [^45]

```text
Research the authentication, database, and API modules in parallel using separate subagents
```

Anthropic's multi-agent research system, an Opus lead coordinating parallel Sonnet subagents, posted a large uplift over single-agent Opus on their internal research eval, and most of the variance traced to token usage rather than model choice (the figures and their interpretation are Ch. 10). The practical reading you bring to fan-out is the part that holds across domains: multi-agent systems excel at "valuable tasks that involve heavy parallelization, information that exceeds single context windows, and interfacing with numerous complex tools," and they struggle in domains "that require all agents to share the same context or involve many dependencies between agents," with coding named as the example. Most coding tasks are the second kind, so reach for fan-out at the research-and-exploration edges, not the tightly-coupled middle. [^40]

Their decomposition rule of thumb is to give each subagent "an objective, an output format, guidance on the tools and sources to use, and clear task boundaries." Skip that and the agents "duplicate work, leave gaps, or fail to find necessary information." Their scaling heuristic: "simple fact-finding requires just 1 agent with 3-10 tool calls," while "complex research might use more than 10 subagents with clearly divided responsibilities." Embed those rules right in the orchestrator prompt. [^40]

Here is the fan-out caveat that bites people. When subagents finish, their results come home to your main conversation, and "running many subagents that each return detailed results can consume significant context," which defeats the very isolation you spawned them for. Two ways out. Instruct each one to return a tight summary ("report only the failing tests with their error messages"), or step up to agent teams when you need sustained parallelism with every worker holding its own independent window. [^46]

Fan-out helps when the tasks are truly independent, when wall-clock pressure is real, when parallel exploration adds signal, and when your window is already saturated. It hurts when there are sequential dependencies, same-file edits, scope so tiny the coordination costs more than the work, or boundaries so fuzzy the agents end up redoing each other's effort.

### Nesting (subagents spawning subagents) -- the five-level cap

As of v2.1.172 (June 10, 2026), "sub-agents can now spawn their own sub-agents (up to 5 levels deep)." Before that, subagents could not spawn other subagents at all, so any older guidance you find is stale. [^47]

The mechanics: [^48]

- **Depth is the number of subagent levels below the main conversation**, regardless of whether each level runs foreground or background.
- **A subagent at depth five does not receive the Agent tool and cannot spawn further.** The limit is **fixed and not configurable.**
- A nested subagent is configured exactly like a top-level one and resolves from the same scopes. Only the **top-level** subagent's summary returns to you; intermediate output never reaches your main conversation. That is the whole appeal: a reviewer subagent that dispatches a verifier per finding keeps all the verifier chatter off your budget.
- The subagent panel shows the full tree (each row a `(+N)` descendant count, with a path back to `main`); the `/agents` Running tab shows a flat list.
- To stop a specific subagent from spawning children, omit `Agent` from its `tools` or add it to `disallowedTools`.

> A **bug fix worth knowing** for older installs: **v2.1.181 (June 17, 2026)** fixed *foreground* subagents spawning unbounded nested chains; "they now respect the same 5-level depth limit as background subagents." If you are on an earlier build, foreground nesting may not be bounded. [^49]

Nesting buys you clean intermediate isolation, but each layer is a fresh context window with its own prompt prefix and its own token cost, and deep trees compound that fast. The discipline is to keep your leaves on small models. An Opus instance sitting at a leaf reading a log file is paying top-tier rates for Haiku work.

### Forks (`/fork`) -- a worker that inherits the whole conversation

A fork is a subagent that "inherits the entire conversation so far instead of starting fresh." It deliberately drops the input isolation the others give you. It sees the same system prompt, tools, model, and message history as the main session, so you can hand it a side task without re-explaining a single thing. Its own tool calls still stay out of your conversation and only its final result comes back, which keeps the main window clean. [^50]

Reach for a fork when a named subagent would need too much background to be worth the setup, or when you want to try several approaches in parallel from the same starting point.

```text
/fork draft unit tests for the parser changes so far
```

The key behaviors and version gating: [^51]

- Forked subagents require **v2.1.117+**. From **v2.1.161** the `/fork` command is **enabled by default**; on earlier versions it needs `CLAUDE_CODE_FORK_SUBAGENT=1`. Letting Claude *itself* spawn forks is experimental and staged-rollout.
- Setting `CLAUDE_CODE_FORK_SUBAGENT=1` does two things: Claude can spawn a `fork` subagent type explicitly, **and every subagent spawn runs in the background** (set `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1` to keep spawns synchronous).
- **Prompt-cache reuse is the cost win:** because a fork's system prompt and tool definitions are identical to the parent, its first request "reuses the parent's prompt cache," making forking cheaper than spawning a fresh subagent for tasks that need the same context.
- **A fork cannot spawn another fork.** It *can* spawn other (named) subagent types, and those count toward the five-level depth limit.
- Running forks appear in a panel below the prompt (one row per fork): Up/Down to move, Enter to open a fork's transcript and send follow-ups, `x` to dismiss or stop, Esc back to the prompt.

**Fork vs named subagent -- the decision table** [^52]

| | Fork | Named subagent |
|---|---|---|
| Context | Full conversation history | Fresh context with the prompt you pass |
| System prompt + tools | Same as main session | From the subagent's definition file |
| Model | Same as main session | From the subagent's `model` field |
| Permissions | Prompts surface in your terminal | Auto-denied when running in background |
| Prompt cache | **Shared with main session** | Separate cache |

---

## 7.8 Foreground vs background, and worktree isolation

Foreground subagents block the main conversation until they finish, and permission prompts pass straight through to you live. [^53]

Background subagents run concurrently while you keep working. They operate on permissions already granted and auto-deny any tool call that would otherwise prompt. If a background subagent wants to ask a clarifying question, that call fails and the subagent carries on anyway. If one dies on a missing permission, re-run the same task as a foreground subagent to retry it interactively. Claude picks foreground or background based on the task, or you ask for it ("run this in the background"), or you press Ctrl+B to background a running task. Disable background tasks entirely with `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1`. [^53]

`isolation: worktree` gives a subagent its own copy of the repo in a temporary git worktree, branched by default from your default branch rather than the parent's `HEAD`, and auto-cleaned if it makes no changes. This is the safe way to let parallel agents make edits without colliding on files. Worth remembering: inside any subagent, `cd` does not persist between Bash calls and does not affect the main working directory, so when you genuinely need a separate checkout, worktree isolation is the proper tool. (The deeper warning that Bash side effects are permanent lives in Ch. 04.) [^54]

---

## 7.9 Tools subagents never get (why an agent "can't ask the user")

A subagent inherits internal and MCP tools by default, but a handful of tools "depend on the main conversation's UI or session state and are not available to subagents, even when listed in the `tools` field": [^23]

- **`AskUserQuestion`** -- this is the literal reason a subagent "can't ask the user." A background subagent that hits a fork in the road cannot prompt you; it must decide or fail. Design subagents to be *self-contained* and return uncertainty in their summary rather than expecting to interrupt.
- **`EnterPlanMode`**
- **`ExitPlanMode`** -- *unless* the subagent's `permissionMode` is `plan`.
- **`ScheduleWakeup`**
- **`WaitForMcpServers`**

This trips people up often enough that it is worth saying plainly to a cohort. Delegation is a one-way hand-off. The subagent gets a task, works alone, and returns a result. There is no interactive channel back to the human in the middle of the work. When a workflow genuinely needs a person to weigh in partway through, keep that decision point in the main conversation, or in a foreground subagent whose permission prompts pass through to you, and not buried inside a background worker where nobody can hear it.

---

## 7.10 Subagent vs the alternatives -- when to reach for what

Use the main conversation when the task needs frequent back-and-forth, when several phases share significant context (plan, then implement, then test), when you are making a quick targeted change, or when latency matters, since subagents start fresh and need time to gather their bearings. Use a subagent when the task produces verbose output you do not need, when you want to enforce tool restrictions, or when the work is self-contained and can come back as a summary. Consider a Skill instead when you want reusable prompts or workflows that run in the main context rather than in isolation. Use `/btw` for a throwaway question about existing context. [^55]

| You need... | Reach for |
|---|---|
| Isolated, verbose, report-back work | **Subagent** |
| Same isolation but the worker needs full conversation context (cheaper via prompt cache) | **Fork** |
| Reusable knowledge/procedure run in the main context | **Skill** |
| A throwaway question about what's already in context | **`/btw`** |
| Sustained parallelism, each worker its own independent window, peer messaging | **Agent teams** (Ch. 10) |
| Many independent sessions watched from one dashboard | **Background agents** (`/en/agent-view`) |
| A rule that must *always* hold regardless of model judgment | **Hook** (deterministic) |

### Three canonical subagent patterns

1. **Isolate high-volume operations** -- running tests, fetching docs, processing logs. The verbose output stays in the subagent; only the summary returns. `Use a subagent to run the test suite and report only the failing tests with their error messages`. [^56]
2. **Parallel research** -- independent investigations across modules; Claude synthesizes. Best when paths don't depend on each other. [^45]
3. **Chain subagents** -- sequential workflows where each returns to Claude, which passes relevant context to the next. `Use the code-reviewer subagent to find performance issues, then use the optimizer subagent to fix them`. [^57]

---

## 7.11 Worked examples (lift-and-adapt)

**Read-only code reviewer** -- focused, no Edit/Write, detailed checklist: [^58]

```markdown
---
name: code-reviewer
description: Expert code review specialist. Proactively reviews code for quality, security, and maintainability. Use immediately after writing or modifying code.
tools: Read, Grep, Glob, Bash
model: inherit
---
You are a senior code reviewer ensuring high standards of code quality and security.
When invoked: run git diff, focus on modified files, begin review immediately.
Report Critical / Warnings / Suggestions, each with a concrete fix.
```

**Debugger** -- includes Edit because fixing requires modifying code: [^59]

```markdown
---
name: debugger
description: Debugging specialist for errors, test failures, and unexpected behavior. Use proactively when encountering any issues.
tools: Read, Edit, Bash, Grep, Glob
---
You are an expert debugger specializing in root cause analysis.
Capture the error -> reproduce -> isolate -> minimal fix -> verify.
For each issue report: root cause, supporting evidence, the fix, testing approach, prevention.
```

**Self-improving reviewer with memory** -- accumulates codebase patterns across sessions: [^60]

```markdown
---
name: code-reviewer
description: Reviews code for quality and best practices
memory: project
---
You are a code reviewer. Before starting, check your memory for patterns seen before.
As you review, update your agent memory with conventions and recurring issues.
```

---

## Sources

Official Anthropic:
- Create custom subagents -- https://code.claude.com/docs/en/sub-agents
- Model configuration (effort, model resolution, `availableModels`) -- https://code.claude.com/docs/en/model-config
- Claude Code changelog -- https://code.claude.com/docs/en/changelog
- Multi-agent research system -- https://www.anthropic.com/engineering/multi-agent-research-system

[^1]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents)
[^2]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), "Create custom subagents" intro
[^3]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), "Subagents help you" list
[^4]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), "What loads at startup"
[^5]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), Note block: "Subagents work within a single session"
[^6]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), "Built-in subagents"
[^7]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), "Built-in subagents" tabs and "Other" helper table
[^8]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), Explore tab
[^9]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), Plan tab
[^10]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), "Resume subagents"
[^11]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), "Disable specific subagents"
[^12]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), "Built-in subagents" final paragraph
[^13]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), "Write subagent files"
[^14]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), "Choose the subagent scope"
[^15]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), "Choose the subagent scope" table
[^16]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), "Choose the subagent scope", min-version note 2.1.178; corroborated by changelog v2.1.178, 2026-06-15
[^17]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), "Choose the subagent scope" plugin paragraph + plugin Note block
[^18]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), "CLI-defined subagents"
[^19]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), "Write subagent files" Note block
[^20]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), "Use the /agents command"
[^21]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), "Supported frontmatter fields"
[^22]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), "Supported frontmatter fields" table
[^23]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), "Available tools"
[^24]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), "Restrict which subagents can be spawned"
[^25]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), "Restrict which subagents can be spawned" final paragraph
[^26]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), Note block: "In version 2.1.63, the Task tool was renamed to Agent"
[^27]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), "Scope MCP servers to a subagent"
[^28]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), "Scope MCP servers to a subagent", min-version note 2.1.153; corroborated by changelog v2.1.153, 2026-05-28
[^29]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), "Permission modes"
[^30]: web | 2026-06-18 | changelog v2.1.178, 2026-06-15
[^31]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), "Permission modes" Warning block
[^32]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), "Preload skills into subagents"
[^33]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), "Enable persistent memory"
[^34]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), "Conditional rules with hooks"
[^35]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), "Define hooks for subagents"
[^36]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), "Understand automatic delegation"
[^37]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), "Invoke subagents explicitly"
[^38]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), "Auto-compaction"
[^39]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), "Choose between subagents and main conversation" final paragraph
[^40]: web | 2026-06-18 | [anthropic.com/engineering/multi-agent-research-system](https://anthropic.com/engineering/multi-agent-research-system)
[^41]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), "Choose a model"
[^42]: web | 2026-06-18 | [code.claude.com/docs/en/model-config](https://code.claude.com/docs/en/model-config), CLAUDE_CODE_SUBAGENT_MODEL env-var row
[^43]: web | 2026-06-18 | [code.claude.com/docs/en/model-config](https://code.claude.com/docs/en/model-config), "Set the effort level"
[^44]: web | 2026-06-18 | [code.claude.com/docs/en/model-config](https://code.claude.com/docs/en/model-config), "Restrict model selection"
[^45]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), "Run parallel research"
[^46]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), "Run parallel research" Warning block
[^47]: web | 2026-06-18 | changelog v2.1.172, 2026-06-10; corroborated by code.claude.com/docs/en/sub-agents, "Spawn nested subagents", min-version note 2.1.172
[^48]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), "Spawn nested subagents"
[^49]: web | 2026-06-18 | changelog v2.1.181, 2026-06-17
[^50]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), "Fork the current conversation"
[^51]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), "Fork the current conversation" + "Limitations"
[^52]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), "How forks differ from named subagents" table
[^53]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), "Run subagents in foreground or background"
[^54]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), "Write subagent files" + "isolation" frontmatter row
[^55]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), "Choose between subagents and main conversation"
[^56]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), "Isolate high-volume operations"
[^57]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), "Chain subagents"
[^58]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), "Code reviewer" example
[^59]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), "Debugger" example
[^60]: web | 2026-06-18 | [code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents), "Enable persistent memory" example
