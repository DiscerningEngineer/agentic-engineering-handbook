# Chapter 16 -- Reference: Decision Tables, Commands and Glossary

**TL;DR.** This is the chapter you keep open in a second pane. Three decision tables turn the judgment calls from earlier chapters into lookups: which extension mechanism (hook vs. skill vs. MCP vs. subagent vs. memory), which model and effort level, and which verification pattern. A slash-command quick reference covers the built-in and bundled commands, grouped by the point in a session where you reach for it, with version flags on the ones that shipped recently. A glossary defines the vocabulary the rest of the handbook leans on. The whole thing is perishable. Claude Code ships near-daily, model names churn, prices drift. Treat it as a snapshot taken 2026-06-18 against Claude Code ~v2.1.181, and verify anything load-bearing against the live changelog (`/release-notes` or `code.claude.com/docs/en/changelog`) and the live models and pricing pages before you teach it or pin it in a runbook. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/changelog]

> **How to read the version flags.** A note like `(v2.1.169+)` means the feature first appeared in that build. If a command or flag in your install behaves differently from what's described here, the likeliest culprit is version skew. Run `/status` to see your version and `/release-notes` to diff. Items marked **research preview** are explicitly unstable and may change shape without a major-version bump.

---

## 16.1 Decision Table A -- Which Extension Mechanism?

The most common architecture mistake is reaching for the wrong primitive: building an MCP server when a skill would do, or stuffing a guardrail into `CLAUDE.md` where the model can talk its way around it. The five primitives sort cleanly by what they actually are. A hook fires as deterministic code at a lifecycle event. A skill is a directory containing a `SKILL.md` that gives the agent additional capabilities it applies itself. MCP is the open standard that connects the agent to external systems. A subagent isolates context-heavy work in its own window. Memory (`CLAUDE.md` and rules) carries static conventions as context. The mature move is to compose them. ^[source: web | 2026-06-18 | https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills]

### The lookup

| If you need... | Reach for | Why this and not the others |
|---|---|---|
| A rule that must **always** hold, no exceptions | **Hook** (deterministic command hook on `PreToolUse`) | Hooks fire as code at lifecycle events -- the model can't reason its way past them. `PreToolUse` blocks with exit code 2. Skills and `CLAUDE.md` are *context*: persuasive, not enforced (see Ch. 03). ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/hooks] |
| Reusable **knowledge or a procedure** Claude should apply itself | **Skill** (`SKILL.md` folder) | Progressive disclosure means it costs ~nothing until invoked: only the name and description live in context until Claude judges the skill relevant. A skill is "a directory containing a `SKILL.md` file." ^[source: web | 2026-06-18 | https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills] |
| Access to an **external system or live data** | **MCP server** | MCP is the connector layer, "a USB-C port for AI applications." A skill is instructions, not a live connection. ^[source: web | 2026-06-18 | https://modelcontextprotocol.io/] |
| **Isolated or parallel** work whose verbose output shouldn't pollute the main window | **Subagent** | A subagent runs in its own context window and returns only a summary. Context isolation *is* the feature (see Ch. 07). ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/sub-agents] |
| **Sustained** multi-stream parallelism with peer coordination | **Agent team** or a **dynamic workflow** | Subagents are subordinate (results flow back to a lead); agent teams are peers; dynamic workflows orchestrate work across tens to hundreds of background agents. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/changelog] |
| **Static project conventions** (build/test commands, architecture map, hard rules) | **`CLAUDE.md` + `.claude/rules/`** | Loaded as context every session; a `paths`-scoped rule defers its cost until Claude touches matching files (see Ch. 03). ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory] |
| A trigger that runs **automatically on a schedule** | **Routine** (`/schedule`) or **`/loop`** | `/schedule` runs on Anthropic-managed cloud cron; `/loop` repeats while the local session stays open. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/commands] |

### Disambiguating the close calls

Four pairings cause most of the thrash. Commit them to memory and you stop second-guessing the rest.

Skill vs. MCP server comes down to one question: does this need a live connection, or just instructions? "How to write a conventional-commits message" is a skill. "Read open issues from Linear" is an MCP server. If the thing has credentials and an endpoint, it's MCP. If it's a paragraph of how-to, it's a skill. ^[source: web | 2026-06-18 | https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills]

Skill vs. slash command is barely a distinction at all. Custom commands were merged into skills: a file at `.claude/commands/deploy.md` and a skill at `.claude/skills/deploy/SKILL.md` both create `/deploy` and work the same way. A bundled slash command like `/code-review` or `/batch` is a skill with a `/`-invocable name, documented in the commands table and marked **Skill**. Write a skill folder for anything reusable. The legacy `.claude/commands/*.md` form still works, but the skill folder is the modern shape. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/skills]

Hook vs. skill is the question of whether you can afford to be wrong. If the consequence of the model ignoring the rule is unacceptable (a force-push to `main`, a `DROP TABLE`, a secret walking out the door), use a deterministic hook. It blocks at `PreToolUse` with exit code 2 and there's no talking it out of that. If the rule is a preference the model should usually honor, like style or naming, a skill or a `CLAUDE.md` line is right and cheaper. The litmus is simple: could a clever rationalization in the transcript defeat this? If yes, and that matters, it's a hook. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/hooks]

Subagent vs. agent team vs. workflow is a question of shape. A subagent is one isolated, subordinate task that reports back, and it's the cheapest of the three. An agent team is multiple peer sessions with a shared task list and peer messaging, still experimental, costing roughly Nx tokens, with diminishing returns past three to five. A dynamic workflow orchestrates tens to hundreds of background agents with handoff, watchable via `/workflows` (v2.1.154+). ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/changelog] Mind the depth ceiling: subagents can spawn subagents up to a fixed five-level depth, where a subagent at depth five no longer receives the Agent tool and cannot spawn further (v2.1.172+), and the auto-mode classifier now evaluates each subagent's task before it spawns (v2.1.178+). The mechanics live in Ch. 07. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/sub-agents]

> **Decomposition is the lever, not agent count.** The orchestration win comes from carving a task into independent pieces that don't duplicate each other's work -- parallelize what's independent, serialize anything with sequential dependencies or same-file edits. Reaching for more agents on a task that won't decompose cleanly just multiplies the token bill. The numbers behind this, and the full treatment, are in Ch. 10.

---

## 16.2 Decision Table B -- Which Model and Effort Level?

Two dials govern cost and capability, and they're independent. Model is the tier. Effort is how much the model thinks per turn. They compose. The mid-2026 lineup churns fast enough that you should verify model IDs, context windows, and prices against the live models and pricing pages before pinning anything. ^[source: web | 2026-06-18 | https://platform.claude.com/docs/en/about-claude/models/overview]

### The model lineup (snapshot 2026-06-18)

| Model | Alias | Context | Reach for it when... |
|---|---|---|---|
| **Claude Fable 5** | `fable` / `claude-fable-5` | 1M | The longest, hardest, multi-day autonomous work -- Anthropic's most capable widely released model. GA on the API June 9, 2026; available in Claude Code from v2.1.170. ^[source: web | 2026-06-18 | https://platform.claude.com/docs/en/about-claude/models/overview] |
| **Claude Opus 4.8** | `opus` / `claude-opus-4-8` | 1M | Default Opus tier; the everyday driver for hard reasoning and long agentic runs. Default model in Claude Code as of v2.1.154, `high` effort by default. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/model-config] |
| **Claude Sonnet 4.6** | `sonnet` / `claude-sonnet-4-6` | 1M | Balanced daily feature work; best speed/intelligence trade. ^[source: web | 2026-06-18 | https://platform.claude.com/docs/en/about-claude/models/overview] |
| **Claude Haiku 4.5** | `haiku` / `claude-haiku-4-5` | **200K** (not 1M) | High-throughput, bounded subagents -- search, format checks, first-pass triage. ^[source: web | 2026-06-18 | https://platform.claude.com/docs/en/about-claude/models/overview] |

A few aliases earn their keep beyond the plain tier names. There's `best` (Fable 5 where your org has access, otherwise the latest Opus), `sonnet[1m]` and `opus[1m]` to force the 1M window where it isn't the default, and `opusplan`, which uses Opus in plan mode and switches to Sonnet for execution. Set the default with `/model` (press `Enter` or type the name to persist; `s` on a row to switch session-only), and override per-run with the `ANTHROPIC_MODEL` env var, which applies only to the session you launch with it. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/model-config]

> **Do not quote prices or benchmark scores as fixed facts.** Per-MTok pricing, Fable 5 pricing, and SWE-bench numbers all drift. Describe the trend and point to the live pricing page and the models overview for current figures. ^[source: web | 2026-06-18 | https://platform.claude.com/docs/en/about-claude/pricing]

### The effort dial

Effort is the primary reasoning-depth-and-spend control, set with `/effort` or the left/right arrows in the `/model` picker. It takes effect immediately, without waiting for the current response to finish. The available levels depend on the model: if you set a level the active model doesn't support, Claude Code falls back to the highest supported level at or below it. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/model-config]

| Level | Use for | Availability |
|---|---|---|
| `low` | Latency-critical work; cheap bounded subagents; simple lookups | All effort-capable models |
| `medium` | Routine feature work where you're cost-conscious | All effort-capable models |
| `high` | Default for most intelligence-sensitive work (the default on Fable 5, Opus 4.8, Opus 4.6, Sonnet 4.6) | All effort-capable models |
| `xhigh` | Deeper reasoning at higher spend -- the sweet spot for tough work; the default on Opus 4.7 | Fable 5, Opus 4.8, Opus 4.7 only |
| `max` | Correctness matters more than cost; session-only; prone to overthinking, so test before adopting broadly | All effort-capable models |
| `ultracode` | Not a true effort level -- sends `xhigh` to the model *and* has Claude orchestrate dynamic workflows for substantive tasks; session-only | Claude Code setting |

Use `/effort auto` to reset to the model default. The effort-capable models are Fable 5, Opus 4.8, Opus 4.7, Opus 4.6, and Sonnet 4.6; Haiku 4.5 has no effort dial, so `/effort` is a no-op there and its speed comes from the tier itself. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/model-config]

### The combined lookup

| Situation | Model | Effort |
|---|---|---|
| Longest / hardest / multi-day autonomous run | Fable 5 (or Opus 4.8) | `high`-`xhigh` (or `max` if correctness is paramount) |
| Daily feature work | Sonnet 4.6 | `medium`-`high` |
| Cheap parallel subagents (search, lint, triage) | Haiku 4.5 | n/a (no effort dial) |
| Large codebase / 100+-file refactor / monorepo cross-package | `opus[1m]` / `sonnet[1m]` | `high` |
| Plan-then-build (separate reasoning from execution) | `opusplan` | `high` |
| Latency-sensitive interactive loop | Sonnet 4.6 | `low`-`medium` |

### Thinking: adaptive replaces budgets

Modern models use adaptive thinking. Depth is decided per step, governed by `effort`. On Opus 4.7 and later, and on Fable 5, adaptive thinking is the *only* supported mode and thinking cannot be turned off; on Opus 4.6 and Sonnet 4.6 it's the recommended mode but you can revert to a fixed budget with `CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING=1`. At the API level, the legacy `budget_tokens` parameter is deprecated on Opus 4.6 and Sonnet 4.6, rejected with a 400 on Opus 4.7, Opus 4.8, and Fable 5, and mandatory only on pre-4.6 models. ^[source: web | 2026-06-18 | https://platform.claude.com/docs/en/build-with-claude/adaptive-thinking] The `ultrathink` keyword still works in Claude Code as a per-turn nudge: it adds an in-context instruction for deeper reasoning on that turn and leaves the API effort level unchanged (see Ch. 05). Teach `effort` as the primary control. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/model-config]

> **Model routing for subagents is a cost lever.** Route cheap, bounded subagent work to Haiku and reserve Opus/Fable for the orchestrator and the genuinely hard reasoning. The resolution order is `CLAUDE_CODE_SUBAGENT_MODEL` env var > per-invocation `model` parameter > subagent frontmatter `model:` > the main session's model. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/sub-agents]

> **A 200K curated window usually beats an uncurated 1M.** Reach for the 1M variants when the task genuinely calls for it: 100+-file refactors, large-codebase onboarding, monorepo cross-package work. Context rot still applies even there, so compact proactively (see Ch. 02). The 1M window being available is not a reason to default to it. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/model-config]

---

## 16.3 Decision Table C -- Which Verification Pattern?

As agents generate more code, the human's job shifts from author to reviewer and verifier, and verification becomes the load-bearing skill. The pattern you pick depends on what kind of correctness signal you can get and how much it matters. The patterns themselves live in Ch. 11; this table is the lookup. ^[source: web | 2026-06-18 | https://claude.com/blog/how-anthropic-teams-use-claude-code]

### The lookup

| Situation | Pattern | How |
|---|---|---|
| Tests exist or are writable | **TDD loop** | Have Claude write tests from the spec *first*, confirm they fail, then implement to green. Hallucinated signatures fail at test time, not in prod (Ch. 11). ^[source: web | 2026-06-18 | https://claude.com/blog/how-anthropic-teams-use-claude-code] |
| High-stakes correctness | **Adversarial generator + reviewer** | A separate agent tasked to break the code catches more than self-review. Different agent, different prompt, hostile stance (Ch. 11). |
| Multi-dimension quality (security / perf / coverage / regression) | **Parallel specialized verifiers** | One agent per dimension, lead synthesizes. This is how managed Code Review works: "multiple agents analyze the diff and surrounding code in parallel ... then a verification step checks candidates against actual code behavior to filter out false positives." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/code-review] |
| Automated PR gate | **Managed Code Review / `/code-review`** | `/code-review` locally (with `--fix`, `--comment`, effort levels, targeting), or the managed GitHub app for inline severity comments on every PR. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/code-review] |
| Cleanup without bug-hunting | **`/simplify`** | From v2.1.154, `/simplify` runs a cleanup-only review (four agents: reuse, simplification, efficiency, altitude) and applies fixes -- it does *not* hunt for correctness bugs. Use `/code-review` for bugs. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/code-review] |
| Long autonomous run | **Milestone verification (tests as oracle)** | Verify every milestone against the suite; advance only on green (Ch. 11). |
| "Did the change actually work in the app?" | **`/verify` / `/run`** | Build and drive the running app to observe real behavior, not just tests or type checks (v2.1.145+). ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/skills] |

### Code Review internals worth knowing

The managed and local review tools share an architecture worth understanding, because it generalizes to any verifier you build yourself.

The shape is fan-out then verify-against-behavior. A fleet of specialized agents each hunts a different issue class, and then a verification step checks every candidate finding against actual code behavior to cut the false positives. Findings are deduplicated and ranked by severity. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/code-review]

Severity is informational, not blocking. Findings come tagged Important (red), Nit (yellow), and Pre-existing (purple), and they never auto-approve or block a PR, so your existing review workflow stays intact. The check run completes "neutral" and branch protection doesn't trip. If you want a merge gate, build it yourself by reading the machine-readable severity counts from the check-run output (the `normal` key holds the count of Important findings). ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/code-review]

`REVIEW.md` is the high-priority knob. A repo-root `REVIEW.md` is injected into every review agent's system prompt as the highest-priority instruction block. Use it to redefine what "Important" means for your repo, cap nit volume, skip generated paths, add repo-specific checks, and raise the verification bar (for example, "behavior claims need a `file:line` citation, not an inference from naming"). `CLAUDE.md` violations surface as nits. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/code-review]

Effort tunes the precision/recall trade. Lower effort returns fewer, higher-confidence findings; `high` through `max` give broader coverage and may include uncertain ones. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/code-review]

> **Version and availability flags.** Local `/code-review` reporting cleanups (not just bugs) requires v2.1.151+; the command was named `/simplify` before v2.1.147 and behaved differently. The managed GitHub Code Review is a **research preview**, Team/Enterprise only, and **not available with Zero Data Retention**. Anthropic's own figure is that each managed review averages $15-25 and completes in about 20 minutes, scaling with PR size and complexity -- check your analytics dashboard for your actual spend. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/code-review]

> **The verifier must be near-flawless.** The lesson from Anthropic's parallel-Claudes C-compiler work, stated plainly: "it's important that the task verifier is nearly perfect, otherwise Claude will solve the wrong problem." Pick the pattern in this table by how trustworthy a correctness signal you can build, because the leverage of autonomous generation is bounded by the fidelity of the check. The case study and its figures are in Ch. 11. ^[source: web | 2026-06-18 | https://www.anthropic.com/engineering/building-c-compiler]

---

## 16.4 Permission Modes Reference

Permission modes set the baseline for what runs without asking. Layer allow/ask/deny rules on top with `/permissions`, and know that deny and explicit ask rules apply in every mode, including `bypassPermissions`. Cycle with `Shift+Tab` (`default` -> `acceptEdits` -> `plan`, with optional modes slotting in after `plan`), set at startup with `--permission-mode`, or persist via `defaultMode` in settings. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/permission-modes]

| Mode | Runs without asking | Best for | Entry |
|---|---|---|---|
| `default` | Reads only | Getting started; sensitive work | The baseline |
| `acceptEdits` | Reads, file edits, and common filesystem commands (`mkdir`, `touch`, `rm`, `rmdir`, `mv`, `cp`, `sed`) on in-scope paths | Iterating on code you review after the fact | `Shift+Tab` once, or `--permission-mode acceptEdits` |
| `plan` | Reads only -- research and propose, no edits | Exploring a codebase before changing it | `Shift+Tab`, `/plan`, or `--permission-mode plan` |
| `auto` | Everything, with a background safety classifier per action | Long tasks; reducing prompt fatigue | Cycle to it after opting in (v2.1.83+; meets requirements below) |
| `dontAsk` | Only pre-approved (allow-rule) tools + read-only Bash; explicit ask rules are **denied**, not prompted | Locked-down CI and scripts | `--permission-mode dontAsk` (never in the cycle) |
| `bypassPermissions` | Everything (incl. protected-path writes as of v2.1.126) | Isolated containers / VMs only | Start with `--permission-mode bypassPermissions` or `--dangerously-skip-permissions` |

### The guardrails that hold across modes

Some protections don't move no matter which mode you're in, and they're the reason you can run loose without running reckless.

Protected paths are never auto-approved except in `bypassPermissions`. The set covers `.git`, `.config/git`, `.vscode`, `.idea`, `.husky`, `.cargo`, `.devcontainer`, `.yarn`, `.mvn`, `.claude` (except `.claude/worktrees`), shell rc files (`.bashrc`, `.zshrc`, `.profile`, `.envrc`, and kin), package-manager configs (`.npmrc`, `.yarnrc`, `bunfig.toml`), `.mcp.json`, `.claude.json`, and more. In `default`/`acceptEdits`/`plan` these prompt; in `auto` they route to the classifier; in `dontAsk` they're denied. A `permissions.allow` rule in settings does not pre-approve a protected-path write, because the safety check runs first. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/permission-modes]

`bypassPermissions` still circuit-breaks on `rm -rf /` and `rm -rf ~`, and it refuses to start as root/sudo on Linux/macOS (skipped inside a recognized sandbox). What it does not offer is any protection against prompt injection, which is why `auto` is the better answer when you want fewer prompts and still safe. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/permission-modes]

A repo can't grant itself elevated modes. Cloud sessions ignore `defaultMode: "bypassPermissions"` and `"dontAsk"` from checked-in settings, and from v2.1.142 `defaultMode: "auto"` is ignored from project/local settings and must live in `~/.claude/settings.json`. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/permission-modes]

### Auto mode specifics (research preview)

Auto mode runs a separate classifier model that reviews each action before it runs, blocking anything that escalates beyond your request, targets unrecognized infrastructure, or appears driven by hostile content Claude read. A few behaviors are worth carrying in your head.

The requirements all have to hold at once: an account on any plan (a Team/Enterprise admin must enable it), and a model that's Opus 4.6+ or Sonnet 4.6 on the Anthropic API (only Opus 4.7/4.8 on Bedrock/Vertex/Foundry, where it's gated behind `CLAUDE_CODE_ENABLE_AUTO_MODE=1`, working in v2.1.158+). Older models, Sonnet 4.5, Opus 4.5, and Haiku among them, are unsupported. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/permission-modes]

Conversational boundaries are block signals. Tell Claude "don't push" or "wait until I review" and the classifier blocks matching actions even when the default rules would allow them, re-reading the boundary from the transcript on each check. The catch is that a boundary can be lost if compaction removes the message that stated it. For a hard guarantee, add a deny rule. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/permission-modes]

The fallback thresholds aren't configurable. After 3 consecutive blocks or 20 total blocks, auto mode pauses and resumes prompting. In `-p` (headless) mode, repeated blocks abort the session. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/permission-modes]

Broad allow rules are dropped on entry (`Bash(*)`, wildcarded interpreters, package-manager run commands, `Agent` rules) and restored on exit. Narrow rules like `Bash(npm test)` carry over. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/permission-modes]

---

## 16.5 Slash-Command Quick Reference

Type `/` to see what's available in your install. Availability varies by platform, plan, and environment. A command is only recognized at the start of a message; text after it is passed as arguments (`<arg>` required, `[arg]` optional). Entries marked **[Skill]** are bundled skills (a prompt Claude can also auto-invoke); **[Workflow]** entries fan work across many background subagents. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/commands]

> This is the high-frequency subset, grouped by *when in a session you reach for it*. It is not exhaustive -- `/help` lists everything, and `/release-notes` shows what changed. The cosmetic/account commands (`/theme`, `/color`, `/login`, `/stickers`, `/radio`, etc.) are omitted.

### Setting up a repo

| Command | Purpose |
|---|---|
| `/init` | Bootstrap a starter `CLAUDE.md` from codebase analysis. `CLAUDE_CODE_NEW_INIT=1` runs a multi-phase flow covering skills, hooks, and personal memory. |
| `/memory` | Edit `CLAUDE.md` files; enable/disable auto-memory; view auto-memory entries. First thing to run when "Claude isn't following my rule." |
| `/agents` | Manage subagent configurations. |
| `/mcp [reconnect\|enable\|disable]` | Manage MCP server connections and OAuth; reconnect or toggle one server or `all`. |
| `/permissions` | Manage allow/ask/deny rules; review recent auto-mode denials. Alias `/allowed-tools`. |
| `/hooks` | View hook configurations for tool events. |
| `/fewer-permission-prompts` | **[Skill]** Scan transcripts for common read-only calls and add an allowlist to project settings. |

### During a task

| Command | Purpose |
|---|---|
| `/plan [description]` | Enter plan mode; optional description starts plan mode on that task immediately. |
| `/model [model]` | Switch model (saved as default) or open the picker; `s` per-row = session-only; arrows adjust effort. |
| `/effort [level\|auto]` | Set effort (`low`/`medium`/`high`/`xhigh`/`max`/`ultracode`); `auto` resets to model default. Takes effect immediately. |
| `/context [all]` | Visualize context usage as a colored grid with optimization suggestions and capacity warnings. |
| `/compact [instructions]` | Summarize the conversation to free context; optional focus instructions. |
| `/clear [name]` | Start fresh with empty context (previous stays in `/resume`). Aliases `/reset`, `/new`. |
| `/btw <question>` | Quick side question that doesn't bloat the conversation. |
| `/goal [condition\|clear]` | **(v2.1.139+)** Set a completion condition; Claude works across turns until met, with a live overlay of elapsed time/turns/tokens. `clear`/`stop`/`off` cancels. |
| `/advisor [model\|off]` | **(v2.1.98+)** Enable the advisor tool -- a second model consulted at key moments (`opus`/`sonnet`/`fable` (v2.1.170+)/full ID). |
| `/usage` | Session cost, plan limits, activity. On Pro/Max/Team/Enterprise, breaks usage down by skill, subagent, plugin, and MCP server (v2.1.149+). Aliases `/cost`, `/stats`. |

### Running work in parallel

| Command | Purpose |
|---|---|
| `/background [prompt]` | Detach the session to run as a background agent; monitor with `claude agents`. Alias `/bg`. |
| `/tasks` | View/manage everything running in the background. Alias `/bashes`. |
| `/fork <directive>` | **(v2.1.161+ default; v2.1.117+ via env var)** Spawn a forked subagent that inherits the full conversation and works in the background; its result returns to you. (Before v2.1.161, an alias for `/branch` unless `CLAUDE_CODE_FORK_SUBAGENT=1`.) |
| `/branch [name]` | Branch the conversation to try a different direction without losing the current state. |
| `/batch <instruction>` | **[Skill]** Decompose a large cross-codebase change into 5-30 independent units; one background subagent per unit in its own worktree, each opening a PR. Requires git. |
| `/workflows` | Open the workflow progress view to watch/pause/resume/save runs. |
| `/schedule [description]` | Create/list/run **routines** on Anthropic-managed cloud cron. Alias `/routines`. |
| `/loop [interval] [prompt]` | **[Skill]** Run a prompt repeatedly while the session stays open; omit interval to self-pace. Alias `/proactive`. |

### Before you ship

| Command | Purpose |
|---|---|
| `/diff` | Interactive diff viewer for uncommitted changes and per-turn diffs. |
| `/code-review [low\|medium\|high\|xhigh\|max\|ultra] [--fix] [--comment] [target]` | **[Skill]** Review the diff for correctness bugs + cleanups. `--fix` applies findings; `--comment` posts inline PR comments; `ultra` runs a cloud multi-agent review. |
| `/simplify [target]` | **[Skill] (v2.1.154+)** Four parallel agents review for reuse/simplification/efficiency/altitude and apply fixes -- no bug-hunting. |
| `/review [PR]` | Review a PR locally in the current session. |
| `/security-review` | Analyze pending branch changes for injection, auth, and data-exposure risks. |
| `/ultrareview [PR]` | Deep multi-agent cloud review; preferred form is `/code-review ultra`. |
| `/verify` | **[Skill] (v2.1.145+)** Confirm a change works by building and driving the running app. |
| `/run` | **[Skill] (v2.1.145+)** Launch and drive the project's app to see a change live. |

### Between / recovering sessions

| Command | Purpose |
|---|---|
| `/resume [session]` | Resume by ID/name or open the picker; background sessions show with `bg` (v2.1.144+). Alias `/continue`. |
| `/rewind` | Roll code and/or conversation back to a checkpoint, or summarize from a message. Aliases `/checkpoint`, `/undo`. |
| `/cd <path>` | **(v2.1.169+)** Move the session to a new working directory, preserving the prompt cache (new `CLAUDE.md` appended, not rebuilt). |
| `/add-dir <path>` | Add a working directory for file access this session. |
| `/teleport` | Pull a Claude Code on the web session into this terminal. Alias `/tp`. |
| `/remote-control` | Make this session controllable from claude.ai. Alias `/rc`. |

### Diagnostics & maintenance

| Command | Purpose |
|---|---|
| `/doctor` | Diagnose your install/settings; press `f` to have Claude fix issues. |
| `/debug [description]` | **[Skill]** Enable debug logging and troubleshoot from the session debug log. |
| `/status` | Settings interface (Status tab): version, model, account, connectivity. Works mid-response. |
| `/config [key=value ...]` | Open settings, or (v2.1.181+) set a key directly, e.g. `/config thinking=false`; `/config help` lists keys. Alias `/settings`. |
| `/sandbox` | Toggle sandbox mode (supported platforms only). |
| `/reload-skills` | **(v2.1.152+)** Re-scan skill/command directories so on-disk changes apply without restart. |
| `/release-notes` | Interactive changelog/version picker. |
| `/feedback [report]` | Submit feedback / report a bug with session context. Aliases `/bug`, `/share`. |

### Headless & CI (flags, not slash commands)

For scripted pipelines, the relevant surface is launch flags, not in-session commands: `claude -p "<prompt>"` for headless runs, with `--output-format text|json|stream-json`, `--json-schema`, `--resume`, `--permission-mode`, and `--bare` (skip auto-discovery for reproducible CI). MCP-server prompts surface as commands too, in the form `/mcp__<server>__<prompt>`, dynamically discovered from connected servers. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/commands]

---

## 16.6 Glossary

Terms are defined as used across this handbook. Where a term is version- or vendor-specific, that's flagged.

**Adaptive thinking** -- The reasoning mode where the model decides depth per step, governed by the `effort` setting, rather than a fixed token budget. The only mode on Opus 4.7+ and Fable 5 (thinking can't be disabled there); recommended but optional on Opus 4.6 and Sonnet 4.6. Replaces the deprecated `budget_tokens` parameter. (`ultrathink` is a per-turn in-context nudge in Claude Code that does not change API effort; see Ch. 05.) ^[source: web | 2026-06-18 | https://platform.claude.com/docs/en/build-with-claude/adaptive-thinking]

**Agent team** -- An experimental mode (`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`) where multiple *peer* sessions share a task list and message each other, each with a full independent context window. Distinct from subagents (which are subordinate). Costs ~Nx tokens; diminishing returns past 3-5 agents (Ch. 07). ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/agent-teams]

**`AGENTS.md`** -- A tool-agnostic project-instructions file read by many agents. Claude Code reads `CLAUDE.md`, not `AGENTS.md`, so if a repo uses it, point `CLAUDE.md` at it with `@AGENTS.md` (or `/init`, which incorporates an existing `AGENTS.md`). ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory]

**Auto mode** -- A permission mode (research preview, v2.1.83+) where a background classifier model approves safe actions and blocks risky ones, eliminating most prompts while keeping a safety net. Pauses to prompting after 3 consecutive / 20 total blocks. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/permission-modes]

**Auto-compaction** -- Claude Code automatically clearing old tool outputs and then summarizing as the context window fills (Ch. 02). ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/context-window]

**Auto-memory (`MEMORY.md`)** -- Notes Claude writes itself, stored at `~/.claude/projects/<project>/memory/`, keyed by git repo (worktrees share one directory). The first 200 lines or 25KB of `MEMORY.md` (whichever comes first) loads at session start; topic files load on demand. Requires v2.1.59+. Audit via `/memory`; disable with `autoMemoryEnabled: false` or `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1`. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory]

**Background agent / session** -- A session detached from the terminal (`/background`, alias `/bg`) that keeps running; monitor with `claude agents`. Survives the terminal closing. Resumable via `/resume` (marked `bg`, v2.1.144+). ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/commands]

**Bundled skill** -- A skill that ships inside Claude Code and is invocable as a slash command (e.g. `/code-review`, `/batch`, `/loop`, `/verify`). Works like a skill you write: a prompt Claude can also auto-invoke. Disable the set with `disableBundledSkills`. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/skills]

**`bypassPermissions`** -- The permission mode that skips all prompts and safety checks (containers/VMs only). Still circuit-breaks on `rm -rf /` and `rm -rf ~`; offers no prompt-injection protection. The `--dangerously-skip-permissions` flag is equivalent. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/permission-modes]

**Checkpoint** -- An automatic snapshot of file state taken before each edit, enabling `/rewind`. **Critical limit:** checkpoints cover only edit-tool file changes -- Bash side effects (`git push`, `rm`, DB/API writes, installs) are permanent and not rewindable (Ch. 04). ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/checkpointing]

**`CLAUDE.md`** -- Claude-native project-instruction file, auto-loaded at session start and treated as *context*, not enforced config. Layered (managed policy > user > project > local); all discovered files concatenate, and instructions closer to your working directory are read last (Ch. 03). ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory]

**Context editing** -- A server-side mechanism (`clear_tool_uses_20250919`, `clear_thinking_20251015`) that *prunes* stale tool results and thinking blocks above a threshold, replacing each with placeholder text -- distinct from compaction, which *summarizes* (Ch. 02). ^[source: web | 2026-06-18 | https://platform.claude.com/docs/en/build-with-claude/context-editing]

**Context engineering** -- Curating *what* enters the context window to maximize the odds of the desired outcome. Governing insight: more context is not better -- recall and reasoning degrade as the window fills (Ch. 02). ^[source: web | 2026-06-18 | https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents]

**Context rot** -- The decline in output quality as the context window fills with tokens, helpful or noise alike. It lives in the attention mechanism: recall and reasoning degrade as the window grows, which is why a fresh start often beats compaction late in a long session (Ch. 02). ^[source: web | 2026-06-18 | https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents]

**Compaction** -- Summarizing conversation history to free context (`/compact`, or server-side compaction in the API). Project-root `CLAUDE.md` survives it (re-read from disk); conversation-only instructions may not (Ch. 02). ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory]

**Dynamic workflow** -- A bundled orchestration (v2.1.154+) that fans work across tens to hundreds of background agents with parallel execution and handoff; watchable via `/workflows`. The `ultracode` setting triggers automatic workflow orchestration (Ch. 10). ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/changelog]

**Effort** -- The primary reasoning-depth-and-spend dial (`low`/`medium`/`high`/`xhigh`/`max`, plus `ultracode`), set with `/effort` or in the `/model` picker. Composes with model choice; available levels depend on the model (`xhigh` only on Fable 5, Opus 4.8, Opus 4.7; Haiku 4.5 has no effort dial at all). ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/model-config]

**Fork** -- A subagent that *inherits the full parent conversation* (`/fork`, default v2.1.161+), reusing the parent's prompt cache -- cheaper for same-context side tasks. A fork can't fork (Ch. 07). ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/sub-agents]

**Headless mode** -- Running Claude Code non-interactively with `claude -p`, with `--output-format`, `--json-schema`, and `--bare`. The backbone of CI and scripted agent pipelines. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/commands]

**Hook** -- A command/HTTP/MCP-tool/prompt/agent handler that fires deterministically at lifecycle events (`PreToolUse`, `PostToolUse`, `UserPromptSubmit`, `SessionStart`, and others). The mechanism for rules that must always hold -- guardrails the model can't reason past. `PreToolUse` can block (exit code 2); `PostToolUse` can't, because the tool already ran (Ch. 09). ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/hooks]

**Just-in-time retrieval** -- Anthropic's default context strategy: keep lightweight identifiers (paths, queries, links) and load data at runtime, letting an agent work over a codebase larger than its window. Contrast with front-loading (root `CLAUDE.md` and `@`-imports, which load in full at start) (Ch. 02). ^[source: web | 2026-06-18 | https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents]

**MCP (Model Context Protocol)** -- The open standard for connecting AI applications to external systems, "a USB-C port for AI applications." Three server primitives: **Tools** (executable functions the model invokes), **Resources** (data sources providing context), **Prompts** (reusable interaction templates surfacing as `/mcp__server__prompt`). Tool *definitions* are deferred -- only names cost context until used. ^[source: web | 2026-06-18 | https://modelcontextprotocol.io/docs/learn/architecture]

**Memory tool** -- An Agent-SDK client-side tool letting Claude read/write a memory directory that persists across sessions; you implement the storage backend. Distinct from Claude Code's `CLAUDE.md`/auto-memory. ^[source: web | 2026-06-18 | https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool]

**Permission mode** -- The session-wide baseline for what runs without asking: `default`, `acceptEdits`, `plan`, `auto`, `dontAsk`, `bypassPermissions`. Cycle with `Shift+Tab`; deny/ask rules apply in every mode. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/permission-modes]

**Plan mode** -- Enforced read-only exploration: Claude reads, searches, and runs read-only commands but can't edit until you approve a plan. The senior workflow primitive: explore, plan, approve, execute. Enter via `Shift+Tab`, `/plan`, or `--permission-mode plan` (Ch. 04). ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/permission-modes]

**Progressive disclosure** -- How skills stay cheap. Three loading levels: (1) `name` + `description` of every skill always in the system prompt; (2) the full `SKILL.md` body loads only when Claude judges the skill relevant; (3) bundled files load only when referenced. A 10,000-line reference costs ~nothing until used. ^[source: web | 2026-06-18 | https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills]

**Protected paths** -- A fixed set of files/dirs (`.git`, `.claude`, `.husky`, `.cargo`, shell rc files, `.mcp.json`, package-manager configs, and more) whose writes are never auto-approved in any mode except `bypassPermissions`. `permissions.allow` rules don't override them. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/permission-modes]

**Routine** -- A scheduled task (`/schedule`, alias `/routines`) that runs on Anthropic-managed cloud cron, independent of any open terminal. Contrast with `/loop` (repeats only while the local session is open). ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/commands]

**Rules (`.claude/rules/`)** -- Modular instruction files; a `paths:` glob makes a rule load only when Claude touches matching files. The primary lever for keeping memory lean in large projects (just-in-time memory) (Ch. 03). ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory]

**Skill** -- A folder (`SKILL.md` + optional scripts/resources) packaging procedural knowledge that loads just-in-time via progressive disclosure. Follows the open Agent Skills standard. Invoked by `/name` or semantic auto-match on the description. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/skills]

**Subagent** -- A specialized assistant with its *own* context window, system prompt, tools, model, and permissions, that works independently and returns only a summary. Context isolation is the point. Defined in `.claude/agents/` (project) or `~/.claude/agents/` (user). Can nest up to five levels deep (v2.1.172+) (Ch. 07). ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/sub-agents]

**Verification** -- Confirming generated code is correct, by tests, adversarial review, specialized verifiers, or running the app (`/verify`). The load-bearing human skill in the agent era -- and only as strong as the verifier itself (Ch. 11). ^[source: web | 2026-06-18 | https://claude.com/blog/how-anthropic-teams-use-claude-code]

**Worktree** -- An isolated git working directory + branch sharing the repo's history, used to run parallel sessions/subagents without file collisions. `/batch` and `isolation: worktree` use these. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/worktrees]

---

## Sources

Official Anthropic:
- Model configuration (models, aliases, effort levels and defaults, ultracode/ultrathink, adaptive thinking, extended context) -- https://code.claude.com/docs/en/model-config
- Models overview (lineup, context windows, pricing, deprecations) -- https://platform.claude.com/docs/en/about-claude/models/overview
- Pricing -- https://platform.claude.com/docs/en/about-claude/pricing
- Adaptive thinking -- https://platform.claude.com/docs/en/build-with-claude/adaptive-thinking
- Context editing -- https://platform.claude.com/docs/en/build-with-claude/context-editing
- Memory tool (Agent SDK) -- https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool
- Hooks -- https://code.claude.com/docs/en/hooks
- Skills -- https://code.claude.com/docs/en/skills
- Subagents -- https://code.claude.com/docs/en/sub-agents
- Agent teams -- https://code.claude.com/docs/en/agent-teams
- Memory (CLAUDE.md, rules, auto-memory) -- https://code.claude.com/docs/en/memory
- Permission modes -- https://code.claude.com/docs/en/permission-modes
- Commands -- https://code.claude.com/docs/en/commands
- Code Review (managed and `/code-review`) -- https://code.claude.com/docs/en/code-review
- Checkpointing -- https://code.claude.com/docs/en/checkpointing
- Context window -- https://code.claude.com/docs/en/context-window
- Worktrees -- https://code.claude.com/docs/en/worktrees
- Changelog (re-verify version-pinned claims here) -- https://code.claude.com/docs/en/changelog
- Effective context engineering for AI agents -- https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
- Equipping agents for the real world with Agent Skills -- https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills
- Building a C compiler with a team of parallel Claudes -- https://www.anthropic.com/engineering/building-c-compiler
- How Anthropic teams use Claude Code -- https://claude.com/blog/how-anthropic-teams-use-claude-code

Model Context Protocol:
- MCP specification and architecture -- https://modelcontextprotocol.io/docs/learn/architecture
