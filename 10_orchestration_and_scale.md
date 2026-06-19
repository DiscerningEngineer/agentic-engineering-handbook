# Chapter 10 -- Multi-Agent Orchestration and Scale

**TL;DR.** Claude Code gives you several ways to run more than one agent at once, and the hard part was never running them. The hard part is knowing which one to reach for, and knowing when to reach for none of them. Three questions resolve almost everything: who holds the plan, do the workers need to talk to each other, and do they touch the same files. Subagents delegate side work inside one conversation. Agent view hands off independent sessions you check back on. Agent teams put a lead in charge of peer sessions that message each other. Dynamic workflows lift the plan out of Claude's head and into a JavaScript script that fans out to dozens-to-hundreds of subagents and cross-checks their results. The Agent SDK and headless `claude -p` run all of this programmatically for CI and production. Underneath every one of them sit git worktrees, the quiet primitive that isolates file edits so parallel work doesn't collide. Anthropic's own numbers are the compass: their multi-agent research system beat single-agent Opus 4 by 90.2% on an internal eval, and token usage alone explained about 80% of the performance variance, which says the lever is good decomposition that puts more high-signal tokens against the problem without duplicating work, rather than simply running more agents.

> **Version warning.** Almost everything here is version-pinned, and several features are explicitly experimental or in research preview. Claude Code ships near-daily; agent teams, workflows, and agent view are moving fast. Treat every version number and feature flag as "true as of mid-June 2026," and re-verify anything you plan to script against the changelog before you rely on it. [^1]

---

## 10.1 The orchestration landscape: surfaces and a decision tree

There is no single "multi-agent" feature in Claude Code. There are four ways to run agents in parallel, and Anthropic's "Run agents in parallel" page organizes them by the question that actually matters in the moment: who coordinates the work. [^2]

| Surface | What it gives you | Who holds the plan | Reach for it when |
|---|---|---|---|
| **Subagents** | Delegated workers inside one session that do a side task in their own context and return a summary | Claude, turn by turn | A side task would flood your main conversation with logs, search results, or file contents you won't reference again |
| **Agent view** (`claude agents`) -- research preview | One screen to dispatch and monitor background sessions; step in only when one needs you | You | You have several independent tasks to hand off and check on at a glance |
| **Agent teams** -- experimental, off by default | Multiple coordinated sessions with a shared task list and inter-agent messaging, run by a lead | The lead agent, turn by turn | You want Claude to split a project, assign pieces, and keep workers in sync -- and the workers need to talk |
| **Dynamic workflows** | A script that runs many subagents and cross-checks their results | The script | A job outgrows a handful of subagents, or you want findings verified against each other: a codebase-wide audit, a 500-file migration, cross-checked research |

[^2]

Layered on top of those, the **Agent SDK** and headless **`claude -p`** run the whole loop programmatically, for CI and for products built on the harness (10.6, 10.7). Two more things support parallel work without being a coordination style of their own. Git worktrees give each session a separate git checkout so parallel sessions never edit the same files; they are the isolation primitive several of the surfaces above quietly sit on top of. [^3] And `/batch` is a bundled skill, "a packaged use of subagents and worktrees," that splits one large change into 5 to 30 worktree-isolated subagents that each open a pull request. It rides on the surfaces below it rather than standing as a fifth one. [^2]

Three questions resolve almost every routing decision: [^2]

1. **Who coordinates?** Claude inside one conversation -> subagents. You, checking back -> agent view. Claude supervising peers -> agent teams. A script holding the plan -> workflows.
2. **Do the workers need to talk to each other?** Subagents only report back to the conversation that spawned them; agent-view sessions report only to you; only agent-team teammates message each other directly.
3. **Do the tasks touch the same files?** If yes, isolate with worktrees -- subagents and your own sessions can each get one. Agent teams do not worktree-isolate teammates, so you partition the work so each owns a different set of files.

> One blanket caution Anthropic repeats everywhere: running several sessions or subagents at once multiplies token usage. Multi-agent is a power tool, not a default. [^2]

---

## 10.2 Subagents vs. agent teams: the core distinction

This is the distinction senior engineers get wrong most often, so it is worth nailing precisely. Both parallelize work. What separates them is the communication topology, the shape of who is allowed to talk to whom. [^4]

|  | Subagents | Agent teams |
|---|---|---|
| **Context** | Own context window; results return to the caller | Own context window; fully independent |
| **Communication** | Report results back to the main agent only | Teammates message each other directly |
| **Coordination** | Main agent manages all work | Shared task list with self-coordination |
| **Best for** | Focused tasks where only the result matters | Complex work requiring discussion and collaboration |
| **Token cost** | Lower -- results summarized back to main context | Higher -- each teammate is a separate Claude instance |

[^4]

The one-line heuristic, lifted from the docs: "Use subagents when you need quick, focused workers that report back. Use agent teams when teammates need to share findings, challenge each other, and coordinate on their own." [^4]

A subagent is a specialized worker with its own context window, system prompt, tools, model, and permissions. It works alone and returns only a summary to whoever called it. The isolation is the entire point. You push the verbose work into a subagent -- the test runs, the log scans, the wide file reads -- and only the distilled answer comes back to land in your main window. The noise stays where the noise belongs. The full mechanics of subagents -- the built-in roster, custom-agent frontmatter, the five-level nesting limit, forks, and the model knobs -- are the subject of Ch. 07; this chapter only needs the orchestration-shaped facts about them. Two of those bite people who skip them. First, a subagent can never ask you anything: the tools that depend on the interactive session (`AskUserQuestion`, `EnterPlanMode`, `ScheduleWakeup`, `WaitForMcpServers`) "are not available to subagents, even when listed in the `tools` field." That is the actual reason an agent "can't ask a clarifying question" -- it has no mouth pointed at you, so front-load the context it needs. Second, parallel subagents that edit the same files overwrite each other unless you isolate them in worktrees (10.3). [^5]

### Agent teams in depth (experimental)

Agent teams coordinate multiple full Claude Code sessions. One session is the team lead; the teammates work independently in their own context windows and talk to each other directly. The thing that separates this from subagents, the thing worth holding onto, is that, "unlike subagents, which run within a single session and can only report back to the main agent, you can also interact with individual teammates directly without going through the lead." The lead is a coordinator, not a chokepoint. [^4]

Enabling it (off by default):

```json
// settings.json
{ "env": { "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1" } }
```

Without that variable, "no team is set up at session start, no team directories are written, and Claude does not spawn or propose teammates." [^4]

> **Version-sensitive.** The docs describe agent teams as of v2.1.178. As of that version, spawning a teammate no longer needs a setup step and cleanup is automatic on session exit. Before v2.1.178 you asked Claude to create and name a team first via the `TeamCreate` and `TeamDelete` tools -- both tools no longer exist. The `team_name` input on the Agent tool is now accepted but ignored. The default display mode also changed from `auto` to `in-process` in v2.1.179. Verify your version before scripting against any of this. [^4]

Starting a team is natural language. You describe the task and the teammates:

```text
I'm designing a CLI tool that tracks TODO comments across a codebase.
Spawn three teammates to explore this from different angles:
one on UX, one on technical architecture, one playing devil's advocate.
```

Claude populates a shared task list, spawns the teammates, sets them exploring, and synthesizes what comes back. You can specify count and model ("Spawn 4 teammates... Use Sonnet for each"), and you can require plan approval before a teammate touches anything ("Require plan approval before they make any changes"), which keeps the teammate in read-only plan mode until the lead signs off. Here is the part people skip: "the lead makes approval decisions autonomously," with nobody watching, so the only steering you get is the criteria you wrote into your prompt. "Only approve plans that include test coverage" is the kind of sentence that earns its keep. [^4]

**Architecture** -- four components: the team lead (the main session), teammates (separate Claude Code instances), a shared task list (work items teammates claim and complete), and a mailbox (messaging between agents). State lives locally under a session-derived name, "`session-` followed by the first eight characters of the session ID": team config at `~/.claude/teams/{team-name}/config.json` (removed when the session ends) and the task list at `~/.claude/tasks/{team-name}/` (persists locally, never uploaded, retention governed by `cleanupPeriodDays`). Do not hand-edit the team config; it holds runtime state and gets overwritten on the next state update. Each teammate loads CLAUDE.md, MCP servers, and skills like a normal session, but does not inherit the lead's conversation history -- put what it needs in the spawn prompt. [^4]

A few task semantics carry real weight for correctness. Tasks move through three states (pending, in-progress, completed) and can declare dependencies, so "a pending task with unresolved dependencies cannot be claimed until those dependencies are completed." Claiming "uses file locking to prevent race conditions" when two teammates try to grab the same task. The lead can assign work explicitly, or a teammate self-claims the next unblocked task the moment it finishes the last one. [^4]

**Display modes:** `in-process` (the default -- all teammates in your main terminal; arrow keys select a teammate, Enter opens its transcript so you can message it, `x` stops it, Ctrl+T toggles the task list) or split panes (one pane each, requires tmux or iTerm2 with the `it2` CLI). Set `teammateMode` in settings (`in-process`, `auto`, or `tmux`) or pass `--teammate-mode auto`. Split panes are not supported in VS Code's integrated terminal, Windows Terminal, or Ghostty. [^4]

You can reuse subagent definitions as teammate roles. When spawning, reference a subagent type by name ("Spawn a teammate using the security-reviewer agent type..."). The teammate honors that definition's `tools` allowlist and `model`, and "the definition's body is appended to the teammate's system prompt as additional instructions rather than replacing it." The catch lives in the fine print: the `skills` and `mcpServers` frontmatter fields "are not applied when that definition runs as a teammate." Teammates load skills and MCP from project and user settings the way any normal session does. [^4]

The quality gates are where this earns its place in a regulated shop. Hooks let you intercept the moments that matter: `TeammateIdle` runs when a teammate is about to go idle (exit code 2 sends feedback and keeps it working); `TaskCreated` runs when a task is being created (exit 2 prevents creation); `TaskCompleted` runs when a task is being marked complete (exit 2 prevents completion and sends feedback). This is the lever that stops a teammate from quietly stamping work "done" before it clears the bar you set. [^4]

Teach the limitations honestly, because it is experimental and it shows. `/resume` and `/rewind` do not restore in-process teammates, so after resuming, "the lead may attempt to message teammates that no longer exist" (tell it to spawn new ones). Task status can lag; teammates sometimes fail to mark tasks complete, which blocks the dependents waiting on them. Shutdown is slow, since teammates "finish their current request or tool call before shutting down." And the structural ceilings are firm: one team per session, no nested teams, the lead is fixed for the session's lifetime, and permissions are set at spawn from the lead's mode. [^4]

### The two canonical agent-team patterns

The docs ship two worked examples worth memorizing as templates.

**Parallel code review by lens.** A single reviewer "tends to gravitate toward one type of issue at a time," then runs out of attention. Split the criteria and each lens gets its own thorough pass:

```text
Spawn three teammates to review PR #142:
- One focused on security implications
- One checking performance impact
- One validating test coverage
Have them each review and report findings.
```

**Competing-hypothesis debugging.** A single agent "tends to find one plausible explanation and stop looking." Make the teammates adversarial and you turn anchoring bias into a feature:

```text
Users report the app exits after one message instead of staying connected.
Spawn 5 agent teammates to investigate different hypotheses. Have them talk to
each other to try to disprove each other's theories, like a scientific
debate. Update the findings doc with whatever consensus emerges.
```

The debate is the mechanism. In Anthropic's words: "With multiple independent investigators actively trying to disprove each other, the theory that survives is much more likely to be the actual root cause." [^4]

### Sizing a team

There is no hard limit, but the docs are unusually opinionated here, and the opinion is worth listening to. Start with 3 to 5 teammates. Aim for 5 to 6 tasks per teammate. Remember that token cost climbs in a straight line while coordination overhead and diminishing returns climb faster, so the math turns on you sooner than your enthusiasm does. "Three focused teammates often outperform five scattered ones." Fifteen independent tasks make a fine starting point for three teammates. Scale up when the work genuinely rewards doing things at the same time, and not a moment before. [^4]

---

## 10.3 Git worktrees: the isolation primitive

Worktrees are the unglamorous foundation that makes safe parallelism possible, and nobody ever puts them on a slide. A git worktree is "a separate working directory with its own files and branch, sharing the same repository history and remote as your main checkout," so edits in one session never reach over and touch another's. Hold the division clearly: worktrees isolate the file edits, and subagents and agent teams coordinate the work itself. One is about where the writing lands, the other is about who is writing. [^3]

### Starting Claude in a worktree

```bash
claude --worktree feature-auth   # or -w
```

By default this creates the worktree under `.claude/worktrees/<value>/` at the repo root, on a new branch `worktree-<value>`. Run it again with a different name in another terminal for a second isolated session. Omit the name and Claude generates one such as `bright-running-fox`. Mid-session, you can say "work in a worktree" and Claude uses the `EnterWorktree` tool; it can switch to another worktree under `.claude/worktrees/` by calling `EnterWorktree` with the target path, and the previous worktree stays on disk untouched. [^3]

> **Trust gotcha:** before using `--worktree` interactively in a directory for the first time, accept the workspace-trust dialog by running plain `claude` once there, or `--worktree` exits with an error. Non-interactive `-p` runs skip the trust check, so `claude -p --worktree` proceeds. Add `.claude/worktrees/` to `.gitignore`. [^3]

### Choosing the base branch

Worktrees branch from `origin/HEAD` (your default branch) so they start clean against the remote; if there is no remote or the fetch fails, they fall back to local `HEAD`. To always branch from local `HEAD`, carrying your unpushed commits and feature-branch state -- useful when isolating subagents that need in-progress work -- set `worktree.baseRef` to `"head"`:

```json
{ "worktree": { "baseRef": "head" } }
```

The setting accepts only `"fresh"` or `"head"`, not arbitrary refs. To branch from a specific PR, pass the number prefixed with `#` (Claude fetches `pull/<number>/head` from origin and creates the worktree at `.claude/worktrees/pr-<number>`):

```bash
claude --worktree "#1234"
```

For total control, a `WorktreeCreate` hook replaces the default `git worktree` logic entirely. That is also how you support non-git VCS -- SVN, Perforce, Mercurial -- via `WorktreeCreate`/`WorktreeRemove` hooks. (When the hook replaces git, `.worktreeinclude` is not processed, so copy local config inside the hook script.) [^3]

### Getting your env into the worktree

A worktree is a fresh checkout, so untracked files like `.env` are not present. Add a `.worktreeinclude` file (`.gitignore` syntax) at the project root; "only files that match a pattern and are also gitignored are copied, so tracked files are never duplicated":

```
.env
.env.local
config/secrets.json
```

This applies to `--worktree`, subagent worktrees, and desktop parallel sessions. Remember to initialize each new worktree -- install deps, set up venvs, run your project's setup. [^3]

### Isolating subagents in worktrees

Ask Claude to "use worktrees for your agents," or set `isolation: worktree` in a custom subagent's frontmatter. Each subagent gets a temporary worktree, "removed automatically when the subagent finishes without changes." Subagent worktrees use the same base branch as `--worktree`. Agent view also moves each dispatched background session into its own worktree automatically when it needs to edit files. [^3] [^2]

### Cleanup semantics (read this carefully)

On exit, cleanup depends on state. No uncommitted changes, no untracked files, no new commits -> the worktree and branch are removed automatically (or you are prompted if the session is named). Any uncommitted, untracked, or new-commit state -> Claude prompts to keep or remove; removing discards that work. Non-interactive `--worktree` runs alongside `-p` are never auto-cleaned (no exit prompt), so remove them with `git worktree remove`. Subagent and background-session worktrees are swept once older than `cleanupPeriodDays`, but only if they are clean; worktrees you made with `--worktree` are never swept. While an agent runs, Claude runs `git worktree lock` on its worktree so concurrent cleanup cannot remove it. [^3]

> **Critical safety reminder.** Worktree isolation protects files, but it does not make actions reversible. Checkpoints and rewind cover only edit-tool file changes; Bash side effects -- `git push`, `rm`, DB and API writes, package installs -- are permanent regardless of worktree. Isolation reduces blast radius; see Ch. 04 for the full treatment.

---

## 10.4 Orchestrator-worker: fan-out / fan-in, and the token-variance lesson

The canonical pattern for scale is the orchestrator and its workers. A lead agent reads the task, works out a strategy, spawns workers to chase different aspects at the same time (fan-out), then gathers what they found and stitches it together (fan-in). Anthropic's multi-agent research system is the reference implementation, and its numbers are the thing to anchor on rather than the architecture diagram. This is the canonical home for those figures. [^6]

A multi-agent system with Claude Opus 4 as lead and Claude Sonnet 4 subagents "outperformed single-agent Claude Opus 4 by 90.2% on our internal research eval." [^6] Then comes the number that runs this whole section: on the BrowseComp evaluation, "token usage by itself explains 80% of the variance" in performance (three factors together explained 95%). Sit with that. The win came from putting more high-signal tokens against the problem, which means the lever is decomposition that adds tokens without duplicating work, and "more agents" is only a proxy for that, and a leaky one. [^6] The price of all that throughput has a shape too: "agents typically use about 4x more tokens than chat interactions, and multi-agent systems use about 15x more tokens than chats." That is the budget you spend to buy back wall-clock time and breadth. [^6]

### What makes the orchestrator good

The early failure modes tell you exactly what to watch for. Some versions would spawn "50 subagents for simple queries," or duplicate effort whenever the task descriptions were vague. In one documented case, "one subagent explored the 2021 automotive chip crisis while 2 others duplicated work investigating current 2025 supply chains." Three agents, one job, two of them wasted. [^6]

The fix is delegation discipline. The orchestrator has to hand each worker a clear objective, an output format, tool guidance, and explicit task boundaries -- the way a good tech lead writes a ticket instead of saying "look into it." Anthropic's prompting principles for orchestrators are worth lifting directly: [^6]

- **Teach delegation** -- each subagent "needs an objective, an output format, guidance on the tools and sources to use, and clear task boundaries." Vague instructions like "research the semiconductor shortage" left subagents to misinterpret the task or repeat each other's searches.
- **Scale effort to the task** -- embed explicit rules; "simple fact-finding requires just 1 agent with 3-10 tool calls," complex research wants more subagents with clearly divided responsibilities.
- **Start broad, then narrow** -- "start with short, broad queries, evaluate what's available, then progressively narrow focus," rather than long specific queries up front.
- **Parallelize aggressively** -- the lead "spins up 3-5 subagents in parallel rather than serially," and each subagent "use[s] 3+ tools in parallel"; together these "changes cut research time by up to 90% for complex queries."
- **Let Claude improve its own prompts** -- models proved "able to diagnose why the agent is failing and suggest improvements."

### When fan-out helps vs. hurts

Fan-out helps when the work is breadth-first: several independent directions worth chasing at once, parallel exploration that adds signal rather than redundant noise, wall-clock pressure, saturated context windows. It hurts the moment the agents all need to share the same context, or there are many dependencies tangling them together, or the work is tight and sequential. Coding is the headline example: Anthropic notes "most coding tasks involve fewer truly parallelizable tasks than research," because step N is waiting on what step N-1 decided. [^6]

This is the senior judgment call, the one no flag makes for you. The cheat sheet: [^2] [^6]

| Parallelism helps | Parallelism hurts |
|---|---|
| Truly independent tasks | Sequential dependencies (step N needs step N-1) |
| Breadth-first research / exploration | All agents need the same shared context |
| Wall-clock pressure (you want it faster) | Same-file edits (overwrites) |
| Parallel exploration that adds signal | Tiny scope (coordination cost beats the work) |
| Saturated / near-full context window | Fuzzy task boundaries (duplicated effort) |

---

## 10.5 Dynamic workflows: moving the plan into code

Subagents, skills, and agent teams all keep the plan inside Claude's head, in a context window or in turn-by-turn judgment. A dynamic workflow is "a JavaScript script that orchestrates subagents at scale." Claude writes the script for the task you describe, and a runtime executes it in the background while your session stays responsive. The intermediate results live in script variables instead of Claude's context, "so Claude's context holds only the final answer," and the orchestration stops being a one-time performance and becomes a thing you can read, rerun, and hand to someone else. [^7]

> **Requirements / version:** dynamic workflows require Claude Code v2.1.154 or later and are "available on all paid plans, with Anthropic API access, and on Amazon Bedrock, Google Cloud Vertex AI, and Microsoft Foundry." On Pro you turn them on from the Dynamic workflows row in `/config`. [^7]

### When to use a workflow (vs. everything else)

The official comparison sorts these by who holds the plan: [^7]

|  | Subagents | Skills | Agent teams | Workflows |
|---|---|---|---|---|
| **What it is** | A worker Claude spawns | Instructions Claude follows | A lead supervising peer sessions | A script the runtime executes |
| **Who decides next** | Claude, turn by turn | Claude, per the prompt | The lead, turn by turn | The script |
| **Where results live** | Claude's context | Claude's context | A shared task list | Script variables |
| **What's repeatable** | The worker definition | The instructions | The team definition | The orchestration itself |
| **Scale** | A few tasks per turn | Same as subagents | A handful of long-running peers | Dozens to hundreds of agents per run |
| **Interruption** | Restarts the turn | Restarts the turn | Teammates keep running | Resumable in the same session |

[^7]

Reach for a workflow when the job outgrows a handful of subagents, or when you want a quality pattern you can run again next month and trust to behave the same way. The point is not just running more agents: a workflow "can have independent agents adversarially review each other's findings before they're reported, or draft a plan from several angles and weigh them against each other, so you get a more trustworthy result than a single pass." The canonical jobs: a codebase-wide bug sweep, a 500-file migration, cross-checked research, a hard plan worth drafting more than one way before you commit. [^7]

### Composition patterns

The launch post names the standard shapes a workflow script takes: [^8]

- **Classify-and-act** -- "use a classifier agent to decide on the type of task, and then route to different agents or behavior based on the task."
- **Fan-out-and-synthesize** -- "split up a task into many smaller steps, run an agent on each step and then synthesize those results."
- **Adversarial verification** -- "for each spawned agent, run a separate spawned agent to adversarially verify its output against a rubric or criteria."
- **Tournament** -- "spawn N agents that each attempt the same task using different approaches," then judge the results pairwise.
- **Loop until done** -- "for tasks with an unknown amount of work, loop spawning agents until a stop condition is met."

The script chooses which model each agent uses and whether each subagent runs in its own worktree: workflows "can decide which models an agent uses and whether subagents are run in their own worktree, allowing Claude to choose the intelligence level and isolation needed." You spend Opus where it counts and Haiku where it does not, decided in code rather than in your head. [^8]

### Invoking, watching, saving

Run the bundled one first, just to feel the shape of the thing. `/deep-research <question>` "fans out web searches on a question across several angles, fetches and cross-checks the sources it finds, votes on each claim, and returns a cited report with claims that didn't survive cross-checking filtered out" (requires the WebSearch tool). [^7]

Have Claude write one with the `ultracode` keyword in your prompt (it lights up in the input), or just ask in plain words ("use a workflow"):

```text
ultracode: audit every API endpoint under src/routes/ for missing auth checks
```

> **Version note:** before v2.1.160 the literal trigger keyword was `workflow`; natural-language requests work in both versions. `/effort ultracode` combines `xhigh` reasoning effort with automatic workflow orchestration, so Claude "plans a workflow for each substantive task instead of waiting for you to ask" -- powerful and expensive. Drop back with `/effort high` for routine work. [^7]

Watch a run with `/workflows`, a progress view that shows each phase's agent count, token total, and elapsed time, with keys to pause or resume (`p`), stop an agent or the whole run (`x`), restart an agent (`r`), and save the script as a command (`s`). Saved workflows go to `.claude/workflows/` (project, shared) or `~/.claude/workflows/` (personal) and then run as `/<name>` like any command. They accept input via a global `args` parameter. That last move is where the value compounds: a one-off run you liked turns into a tool the whole team types one slash to use. [^7]

### Runtime limits and permissions (the load-bearing constraints)

| Constraint | Why |
|---|---|
| **No mid-run user input** | Only agent permission prompts can pause a run; for stage sign-off, run each stage as its own workflow |
| **No direct filesystem or shell access from the workflow itself** | Agents read, write, and run; the script only coordinates |
| **Up to 16 concurrent agents** (fewer on limited-CPU machines) | Bounds local resource use |
| **1,000 agents total per run** | Prevents runaway loops |

[^7]

The permissions have a sharp edge worth understanding before a long run, not during one. Your session's permission mode "controls only the launch prompt." The subagents a workflow spawns "always run in `acceptEdits` mode and inherit your tool allowlist, regardless of your session's mode," so file edits get auto-approved, but "shell commands, web fetches, and MCP tools that aren't in your allowlist can still prompt you mid-run." Pre-approve what the agents will need, or you will come back to a job that paused an hour ago waiting for a click. "In `claude -p` and the Agent SDK there is no one to prompt, so tool calls follow your configured permission rules without interactive confirmation." [^7]

If you stop a run you can resume it -- "agents that already completed return their cached results, and the rest run live" -- but only within the same session. "If you exit Claude Code while a workflow is running, the next session starts the workflow fresh." [^7]

For org control, disable with `"disableWorkflows": true` in settings or managed settings, or `CLAUDE_CODE_DISABLE_WORKFLOWS=1`. [^7]

---

## 10.6 The Claude Agent SDK: orchestration as code

When you outgrow the CLI, when the work becomes CI/CD or production automation or a product built on the harness itself, you reach for the Claude Agent SDK, which gives you "the same tools, agent loop, and context management that power Claude Code, programmable in Python and TypeScript." The thing you have been driving by hand becomes a thing your code drives. [^9]

> **Naming history (matters if you are reading older material):** "the Claude Code SDK has been renamed to the Claude Agent SDK... This change reflects the SDK's broader capabilities for building AI agents beyond just coding tasks." The Python `ClaudeCodeOptions` type was renamed to `ClaudeAgentOptions`, a breaking change introduced at SDK v0.1.0 (which also stopped loading Claude Code's system prompt by default). [^10]

**Packages:** `npm install @anthropic-ai/claude-agent-sdk` (the TS package "bundles a native Claude Code binary for your platform as an optional dependency, so you don't need to install Claude Code separately") and `pip install claude-agent-sdk` (Python 3.10+). The core entry point is `query()`; options are `ClaudeAgentOptions` (Python) or an options object (TS). [^9]

The agent loop is a stream you consume:

```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions

async def main():
    async for message in query(
        prompt="Find and fix the bug in auth.py",
        options=ClaudeAgentOptions(allowed_tools=["Read", "Edit", "Bash"]),
    ):
        print(message)  # Claude reads the file, finds the bug, edits it

asyncio.run(main())
```

[^9]

The SDK exposes the whole feature surface programmatically:

- **Built-in tools** (Read, Write, Edit, Bash, Glob, Grep, WebSearch, WebFetch, Monitor, AskUserQuestion)
- **Hooks** (`PreToolUse`, `PostToolUse`, `Stop`, `SessionStart`, and more, via `HookMatcher`)
- **Subagents** (via `agents={...}` with `AgentDefinition`; include `Agent` in `allowed_tools` to auto-approve invocations; messages carry a `parent_tool_use_id` so you can attribute output to a subagent)
- **MCP** (`mcp_servers={...}`)
- **Permissions** (`allowed_tools`, `permission_mode`)
- **Sessions** (capture `session_id` from the `init` SystemMessage, then `resume=session_id`)

By default it also loads filesystem config (`.claude/skills`, `.claude/commands`, `CLAUDE.md`, plugins) matching the CLI; restrict that via `setting_sources` / `settingSources` (pass `[]` to run isolated, which "is especially important for CI/CD pipelines, deployed applications, test environments, and multi-tenant systems"). [^9]

Orchestration in the SDK is the fan-out pattern written in code: multiple subagents run concurrently, so "independent subtasks finish in the time of the slowest one rather than the sum of all of them." During a review you run the style-checker, the security-scanner, and the test-coverage subagent all at once instead of waiting for each to clear before the next begins. For runs that coordinate dozens to hundreds of agents, the SDK exposes the `Workflow` tool (TypeScript Agent SDK v0.3.149+), which moves the orchestration into a script the same way 10.5 describes. [^11]

### SDK vs. CLI vs. Managed Agents

| Use case | Best choice |
|---|---|
| Interactive development, one-off tasks | **CLI** (`claude`) |
| CI/CD pipelines, custom apps, production automation | **Agent SDK** |
| Production agents without operating your own sandbox/session infra; long-running async sessions | **Managed Agents** (hosted REST API; Anthropic runs the agent and the sandbox) |

The common path Anthropic recommends is to "prototype with the Agent SDK locally, then move to Managed Agents for production." There is also a lower-level cousin, the Anthropic Client SDK, and the difference between them is who runs the loop: "with the Client SDK, you implement a tool loop. With the Agent SDK, Claude handles it." [^9]

> **Auth caveat:** "Unless previously approved, Anthropic does not allow third party developers to offer claude.ai login or rate limits for their products, including agents built on the Claude Agent SDK." Use API-key auth (`ANTHROPIC_API_KEY`) or a provider (Bedrock `CLAUDE_CODE_USE_BEDROCK=1`, Vertex `CLAUDE_CODE_USE_VERTEX=1`, Foundry `CLAUDE_CODE_USE_FOUNDRY=1`). [^9]

---

## 10.7 Headless mode: `claude -p` for scripts and CI

The SDK's CLI face is headless mode, `claude -p` (or `--print`). This is the backbone of scripted pipelines, project-specific linters, and CI gates, the workhorse that runs when no human is sitting there to watch it. [^12]

```bash
claude -p "Find and fix the bug in auth.py" --allowedTools "Read,Edit,Bash"
```

`--bare` is "the recommended mode for scripted and SDK calls," and "will become the default for `-p` in a future release." It skips the auto-discovery of hooks, skills, plugins, MCP servers, auto memory, and CLAUDE.md, which is the whole point: you get "the same result on every machine," with only the flags you passed explicitly taking effect. No surprises riding in from someone else's local config. It also skips OAuth and keychain reads, so auth has to come from `ANTHROPIC_API_KEY` or an `apiKeyHelper`. Load context back in on purpose with `--append-system-prompt[-file]`, `--settings`, `--mcp-config`, `--agents`, and `--plugin-dir`/`--plugin-url`. [^12]

Structured output is what makes `-p` composable, what lets it slot into a pipe like any other Unix citizen:

- `--output-format text` (default), `json` (result, session ID, and metadata including `total_cost_usd` and a per-model cost breakdown), or `stream-json` (newline-delimited events for real-time streaming).
- `--json-schema '<JSON Schema>'` with `--output-format json` constrains output; the structured result lands in the `structured_output` field. Pipe through `jq` to extract fields.
- Streaming: `--output-format stream-json --verbose --include-partial-messages`. The `system/init` event reports session metadata plus loaded plugins (`plugins` / `plugin_errors`) -- use `plugin_errors` to fail CI when a plugin did not load -- and `system/api_retry` events surface retry progress. [^12]

Piping works like any Unix tool, and there is a quiet bonus to it: piping a diff means Claude does not need Bash permission to read it, because the diff is already on stdin.

```jsonc
// package.json -- Claude as a typo linter in CI
{
  "scripts": {
    "lint:claude": "git diff main | claude -p \"you are a typo linter. for each typo in this diff, report filename:line on one line and the issue on the next. return nothing else.\""
  }
}
```

(Piped stdin is capped at 10 MB as of v2.1.128; for larger inputs, write to a file and reference the path.) [^12]

For a locked-down CI run, pass a baseline permission mode instead of listing tools one by one. `dontAsk` "denies anything not in your `permissions.allow` rules or the read-only command set," which is exactly what you want when nobody is there to approve a prompt. `acceptEdits` auto-approves writes plus the common filesystem commands (`mkdir`, `touch`, `mv`, `cp`), while "other shell commands and network requests still need an `--allowedTools` entry or a `permissions.allow` rule, otherwise the run aborts when one is attempted." The `--allowedTools` syntax does prefix matching, and there is a trap in it: `Bash(git diff *)` works, and "the space before `*` is important: without it, `Bash(git diff*)` would also match `git diff-index`." One missing space and your allowlist quietly widens. [^12]

For conversation continuity in scripts, `--continue` resumes the most recent conversation, or you can capture a session ID and resume a specific one:

```bash
session_id=$(claude -p "Start a review" --output-format json | jq -r '.session_id')
claude -p "Continue that review" --resume "$session_id"
```

Run both from the same directory, since "session ID lookup is scoped to the current project directory and its git worktrees." User-invoked skills and commands work in `-p` (`/skill-name` in the prompt), but interactive dialogs like `/login` do not, because there is nobody to dialog with. For GitHub Actions and GitLab CI there are dedicated integration guides, and the same `-p` and SDK plumbing runs underneath them. [^12]

---

## 10.8 The lesson that governs all autonomous scale: build the verifier first

Everything above scales throughput. None of it decides whether the throughput produces value or garbage. That is decided by one thing, and it is the most important idea in the chapter for anyone running autonomous agents at scale: your verifier must be near-flawless, because a capable agent will work autonomously to solve whatever you actually measure, and if your measure has a gap, it will solve the wrong problem and look productive doing it.

The clearest demonstration is Anthropic's autonomous C-compiler build, a large run that leaned on heavy parallelism across many sessions (see Ch. 11 for the full case study, including the figures, the test-harness design, TDD-with-agents, and adversarial review). The governing lesson, in the words of the engineer who ran it: "Claude will work autonomously to solve whatever problem I give it. So it's important that the task verifier is nearly perfect, otherwise Claude will solve the wrong problem." [^13]

What you are looking at is specification gaming, reward hacking, and you cannot prompt your way out of it because it is not a defect. It is a structural property of optimizing toward a metric. At human pace you catch a wrong-but-green solution in review, because something feels off and you slow down. At agent pace, across thousands of sessions, an imperfect oracle quietly compounds into a mountain of confidently wrong code while you sleep. The verifier is the spec. Whatever it accepts is what you asked for, whether you meant to or not.

The orchestration-specific corollary from the same build is about collision. When the C-compiler agents tried to compile the Linux kernel, "every agent would hit the same bug, fix that bug, and then overwrite each other's changes." The fix was a sharper oracle: a harness that compiled most of the kernel with GCC and handed only the remaining files to Claude's compiler, so the agents owned different files and stopped fighting over the same one. [^13] The lesson generalizes past compilers and ties straight back to 10.4: when parallel agents keep converging on a single shared bottleneck, your decomposition is wrong, not your agents.

Here is the line that connects this chapter to the rest of the handbook. As agents generate more code in parallel, the human's job collapses onto verification. Orchestration multiplies the output, and a near-flawless verifier is the only thing standing between that output and a quiet disaster you will not notice for a week. Scale does not remove the senior engineer. It moves them out of the implementation and into the chair where the oracle gets designed and the work gets reviewed. Build the oracle first.

---

## 10.9 Decision tables (slide-ready)

**Which parallelism surface?**

| You want... | Reach for |
|---|---|
| A noisy side task off your main context | **Subagent** |
| Several independent tasks you'll check on later | **Agent view** (`claude agents`) -- research preview |
| Workers that talk to each other, debate, or divide a feature | **Agent team** (experimental) |
| Dozens-to-hundreds of agents plus cross-checked results, codified | **Dynamic workflow** |
| One big change -> many PRs | **`/batch`** |
| CI gate, scripted pipeline, production automation | **Agent SDK / `claude -p`** |
| Parallel sessions that must not collide on files | **Git worktrees** (under any of the above) |

**Does it need isolation?**

| Workers edit the same files? | Action |
|---|---|
| Subagents / your own sessions | `isolation: worktree` or `claude --worktree` |
| Agent team | Teams do not worktree-isolate -- partition files so each teammate owns a distinct set |
| Workflow | Script can run each subagent in its own worktree |

**Will parallelism pay off?** Yes if the tasks are genuinely independent, breadth-first, and you are under wall-clock pressure. No if the work is sequential, shares context, edits the same files, or is too small to earn back its coordination cost. And whatever the answer, keep the ~15x token multiplier in view.

---

## Sources

- https://code.claude.com/docs/en/agents
- https://code.claude.com/docs/en/sub-agents
- https://code.claude.com/docs/en/agent-teams
- https://code.claude.com/docs/en/worktrees
- https://code.claude.com/docs/en/workflows
- https://code.claude.com/docs/en/headless
- https://code.claude.com/docs/en/agent-sdk/overview
- https://code.claude.com/docs/en/agent-sdk/subagents
- https://code.claude.com/docs/en/agent-sdk/migration-guide
- https://code.claude.com/docs/en/changelog.md
- https://claude.com/blog/a-harness-for-every-task-dynamic-workflows-in-claude-code
- https://www.anthropic.com/engineering/multi-agent-research-system
- https://www.anthropic.com/engineering/building-c-compiler

[^1]: web | 2026-06-18 | [https://code.claude.com/docs/en/changelog.md](https://code.claude.com/docs/en/changelog.md)
[^2]: web | 2026-06-18 | [https://code.claude.com/docs/en/agents](https://code.claude.com/docs/en/agents)
[^3]: web | 2026-06-18 | [https://code.claude.com/docs/en/worktrees](https://code.claude.com/docs/en/worktrees)
[^4]: web | 2026-06-18 | [https://code.claude.com/docs/en/agent-teams](https://code.claude.com/docs/en/agent-teams)
[^5]: web | 2026-06-18 | [https://code.claude.com/docs/en/sub-agents](https://code.claude.com/docs/en/sub-agents)
[^6]: web | 2026-06-18 | [https://www.anthropic.com/engineering/multi-agent-research-system](https://www.anthropic.com/engineering/multi-agent-research-system)
[^7]: web | 2026-06-18 | [https://code.claude.com/docs/en/workflows](https://code.claude.com/docs/en/workflows)
[^8]: web | 2026-06-18 | [https://claude.com/blog/a-harness-for-every-task-dynamic-workflows-in-claude-code](https://claude.com/blog/a-harness-for-every-task-dynamic-workflows-in-claude-code)
[^9]: web | 2026-06-18 | [https://code.claude.com/docs/en/agent-sdk/overview](https://code.claude.com/docs/en/agent-sdk/overview)
[^10]: web | 2026-06-18 | [https://code.claude.com/docs/en/agent-sdk/migration-guide](https://code.claude.com/docs/en/agent-sdk/migration-guide)
[^11]: web | 2026-06-18 | [https://code.claude.com/docs/en/agent-sdk/subagents](https://code.claude.com/docs/en/agent-sdk/subagents)
[^12]: web | 2026-06-18 | [https://code.claude.com/docs/en/headless](https://code.claude.com/docs/en/headless)
[^13]: web | 2026-06-18 | [https://www.anthropic.com/engineering/building-c-compiler](https://www.anthropic.com/engineering/building-c-compiler)
