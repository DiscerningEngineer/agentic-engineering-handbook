# Chapter 07 -- Agent Skills

**TL;DR.** A skill is a folder. It holds a `SKILL.md` plus whatever optional scripts and reference files the job calls for, and it packages procedural knowledge that loads just-in-time. Only the `name` and `description` sit in context at all times, which is cheap, around 100 tokens per skill. The body loads when the skill is invoked. The bundled files load only when something references them, and they cost zero until then. That progressive disclosure is the entire point. Skills follow an open standard published at agentskills.io, and they travel across claude.ai, Claude Code, the Claude Agent SDK, and the Claude Developer Platform. Custom commands have been merged into skills, so `.claude/commands/deploy.md` and `.claude/skills/deploy/SKILL.md` both create `/deploy`. The mastery lives in the details: the frontmatter knobs (`disable-model-invocation`, `user-invocable`, `allowed-tools`/`disallowed-tools`, `context: fork`, `paths`, `model`/`effort`), the **lifecycle gotcha** (an invoked skill enters context once and is not re-read each turn, so you write standing instructions, not "do this now"), and knowing when to reach for a skill versus a command, an MCP server, or a subagent.

> **A note on version pins.** Claude Code ships near-daily, and frontmatter fields, bundled-skill rosters, and budget knobs all move with it. Version-pinned claims below carry a build number so you know roughly when a feature landed; treat them as "introduced around here," not "fixed there forever." Re-verify any version-sensitive claim against the live changelog (`https://code.claude.com/docs/en/changelog.md`) and the skills doc (`https://code.claude.com/docs/en/skills`) before you script against it.

---

## 7.1 What a skill is (and why it exists)

A skill starts as a recurring annoyance. You keep pasting the same instructions, the same checklist, the same multi-step procedure into chat. Or a section of `CLAUDE.md` has quietly grown from a fact into a procedure while you weren't watching. [^1] Capture that procedure once as a folder, and Claude adds it to its toolkit. `CLAUDE.md` content loads in full every session and costs tokens forever; a skill's body loads only when used, so long reference material costs almost nothing until you need it. [^1]

Anthropic's mental model is onboarding a new hire. [^2] You don't hand a new engineer the entire wiki on day one. You give them a short orientation that says here's how we deploy, here's the form-filling guide if you ever touch PDFs, here's the API reference back in the appendix. A skill is that orientation, structured so the agent walks to the appendix only when the task sends it there.

The design philosophy is composability. As models get more capable, the bottleneck moves. It stops being "can the model do this" and becomes "does it have the procedural knowledge for this domain, loaded at the right moment." So you specialize an agent with composable capabilities and let the model assemble them, instead of building a separate fragmented agent for every use case. [^2]

Skills are an open standard. Anthropic first shipped Agent Skills on October 16, 2025, then on December 18, 2025 published the spec as an open standard at [agentskills.io](https://agentskills.io) for cross-platform portability, working across claude.ai, Claude Code, the Claude Agent SDK, and the Claude Developer Platform. [^3] Claude Code extends the base standard with its own features: invocation control, subagent execution (`context: fork`), and dynamic context injection. [^1]

### Skills, tools, MCP, and prompts are different things

Keep the categories distinct (full decision table in section 7.8):

- **Skills** are procedural knowledge Claude applies itself: workflows, conventions, checklists, domain expertise, loaded on demand. They complement MCP, teaching the agent the more complex workflows that involve external tools and software. [^2]
- **MCP** integrates external tools and live systems (GitHub, databases, Slack).
- **Prompts** are conversation-level instructions for one-off tasks; skills load on demand and eliminate the need to repeat the same guidance across conversations. [^4]

---

## 7.2 Anatomy and progressive disclosure

Every skill is a directory with `SKILL.md` as the required entrypoint. Everything else is optional: [^1]

```text
my-skill/
+-- SKILL.md           # required -- overview + navigation; the only file always discoverable
+-- reference.md       # detailed reference, loaded only when referenced
+-- examples.md        # example outputs, loaded only when needed
+-- template.md        # a template for Claude to fill in
+-- scripts/
    +-- validate.py    # script Claude executes -- code never enters context, only its output
```

`SKILL.md` has two parts: YAML frontmatter between `---` markers (tells Claude when to use the skill) and a markdown body (the instructions Claude follows when it runs). [^1]

### The three levels of loading

This is the load-bearing concept. Official docs lay it out as a three-level architecture: [^4]

| Level | What | When loaded | Token cost |
|-------|------|-------------|--------------------|
| **1 -- Metadata** | `name` + `description` from frontmatter | Always, at startup (in the system prompt) | ~100 tokens per skill [^5] |
| **2 -- Instructions** | The `SKILL.md` body | When the skill is triggered (Claude reads it via bash) | Under 5k tokens (keep it lean) [^5] |
| **3+ -- Resources/code** | Bundled `.md` files, datasets, scripts | As needed, only when referenced/executed | Effectively unlimited; zero until accessed [^5] |

The mechanism that makes this work is the filesystem. Skills aren't injected wholesale into context. They live as files on the VM, and Claude navigates them with bash the way you would. When a skill triggers, Claude reads `SKILL.md`. If that body references `FORMS.md`, Claude reads that too, but only at that point. A task that touches the sales schema and leaves the finance schema alone loads only the sales file. Scripts get executed, so `validate_form.py`'s code never touches the window, only its stdout does. [^4]

Why a senior should care: this is the same just-in-time-retrieval principle from context engineering (see Ch. 02), turned on procedural knowledge instead of facts. The lever it hands you is real. Because files cost nothing until they're read, the amount of context you can bundle into a skill is effectively unbounded: you install many skills, ship comprehensive API docs and large datasets alongside them, and pay for none of it until a task reaches for it. [^4] [^2]

### Two kinds of skill content

How you intend to invoke a skill should shape what you put in it: [^1]

- **Reference content** adds knowledge Claude applies to your current work: conventions, style guides, patterns, domain facts. It runs inline alongside conversation context. Example: an `api-conventions` skill ("Use RESTful naming; return consistent error formats; include request validation").
- **Task content** gives step-by-step instructions for a specific action such as deploy, commit, or code-gen. These are usually things you want to invoke explicitly with `/skill-name`, often with `disable-model-invocation: true` so Claude doesn't decide on its own to, say, deploy because the code looks ready.

---

## 7.3 Invocation: how a skill gets triggered

Two paths, and the distinction matters for authoring: [^1]

1. **Direct (`/skill-name`).** You type it. Deterministic, you control timing. The command name comes from where the file lives, not the frontmatter `name` (see the naming table below).
2. **Semantic auto-match.** Claude reads every skill's `description` at startup and decides, on its own, when a request matches. Description quality is discoverability. A vague description means the skill never fires when you need it; an overbroad one means it fires when you don't.

### How a skill gets its command name

A common gotcha: the frontmatter `name` sets the display label in listings, but (except for a plugin-root `SKILL.md`) does not set what you type after `/`. The command name comes from the path: [^1]

| Location | Command name source | Example |
|----------|---------------------|---------|
| `~/.claude/skills/<dir>/SKILL.md` or `.claude/skills/<dir>/SKILL.md` | Directory name | `deploy-staging/` -> `/deploy-staging` |
| Nested `.claude/skills/` when the name clashes | Subdir path + dir name | `apps/web/.claude/skills/deploy/` -> `/apps/web:deploy` |
| `.claude/commands/<file>.md` (legacy) | File name | `deploy.md` -> `/deploy` |
| Plugin `skills/<dir>/` | Dir name, plugin-namespaced | `my-plugin/skills/review/` -> `/my-plugin:review` |
| Plugin root `SKILL.md` | Frontmatter `name` (the one exception) | `name: review` -> `/my-plugin:review` |

The plugin-root case is the one place `name` sets the command, because there's no skill directory to take it from; the plugin directory name is the fallback. [^1]

---

## 7.4 The complete frontmatter reference

All fields are optional; only `description` is recommended, so Claude knows when to use the skill. [^1] This table consolidates the Claude Code frontmatter reference, and the senior-relevant nuances are spelled out beneath it.

| Field | Purpose |
|-------|---------|
| `name` | Display label in listings; defaults to directory name. **Does not** set the `/` command for non-plugin-root skills. |
| `description` | What the skill does and when to use it. Drives auto-invocation. If omitted, the first paragraph of the body is used. **Put the key use case first** -- it gets truncated (see section 7.7). |
| `when_to_use` | Extra trigger phrases / example requests, appended to `description` in the listing. Counts toward the same listing cap. |
| `argument-hint` | Autocomplete hint, e.g. `[issue-number]` or `[filename] [format]`. |
| `arguments` | Named positional args for `$name` substitution. Space-separated string or YAML list; names map to positions in order. |
| `disable-model-invocation` | `true` -> only you can invoke (manual-only). Use for side-effecting workflows (`/commit`, `/deploy`). Also removes the description from context and blocks subagent preload. Default `false`. |
| `user-invocable` | `false` -> hidden from the `/` menu; only Claude can invoke. Use for background knowledge that's not a meaningful user command. Default `true`. |
| `allowed-tools` | Tools Claude may use **without per-use approval** while the skill is active. Does not restrict the pool; it pre-approves. Space/comma-separated or YAML list. |
| `disallowed-tools` | Tools removed from the pool while the skill is active. Clears on your next message. (Added v2.1.152.) [^6] |
| `model` | Model to use while active (`/model` values, or `inherit`). Applies for the rest of the current turn only; session model resumes next prompt. |
| `effort` | Reasoning depth while active: `low`/`medium`/`high`/`xhigh`/`max` (availability depends on model). Overrides session effort. |
| `context` | `fork` -> run the skill in a forked subagent context. See section 7.6. |
| `agent` | Which subagent type to use when `context: fork` is set (`Explore`, `Plan`, `general-purpose`, or a custom agent). Defaults to `general-purpose`. |
| `hooks` | Hooks scoped to this skill's lifecycle (see Ch. 10). |
| `paths` | Glob patterns that gate auto-activation: the skill loads automatically only when working on matching files. Same format as path-specific memory rules. |
| `shell` | `bash` (default) or `powershell` for inline `` !`cmd` `` execution (PowerShell requires `CLAUDE_CODE_USE_POWERSHELL_TOOL=1`). |

[^1]

### The two invocation-control fields (memorize this matrix)

`disable-model-invocation` and `user-invocable` get confused constantly. They are orthogonal controls, and the confusion was common enough that it sat open as a docs issue until Anthropic added a warning to the metadata table clarifying that `user-invocable` is a UI setting only. [^7]

| Frontmatter | You can invoke | Claude can invoke | When loaded into context |
|-------------|:--------------:|:-----------------:|--------------------------|
| (default) | Yes | Yes | Description always in context; full skill loads when invoked |
| `disable-model-invocation: true` | Yes | **No** | Description **not** in context; full skill loads when you invoke |
| `user-invocable: false` | **No** | Yes | Description always in context; full skill loads when invoked |

[^1]

The senior takeaways:
- For side-effecting actions (deploy, commit, send-message), use **`disable-model-invocation: true`**. You don't want the model deciding to ship. As a bonus, it removes the description from context entirely, freeing budget.
- For pure background knowledge ("how this legacy system works"), use **`user-invocable: false`**. A command like `/legacy-system-context` is a poor fit for a human-typed action, but Claude should pull it in when relevant.
- `user-invocable` controls **menu visibility** and nothing more. It does not stop Claude from calling the skill programmatically through the Skill tool. To actually block the model, you need `disable-model-invocation: true`. [^1]
- Version-sensitive bug to know about: plugin-defined skills have not reliably honored `disable-model-invocation` the way user/project skills do, forcing all plugin skills into context regardless of relevance. Reported on v2.1.29 and still open. [^8] Verify current behavior before relying on it for plugin-distributed skills.

### `allowed-tools` is pre-approval, not a sandbox

Here's the misread that bites people: `allowed-tools` does not lock the skill down to only those tools. Every tool stays callable. The field grants permission to use the listed tools without prompting while the skill is active, and your baseline permission settings still govern everything else. To actually remove tools, reach for `disallowed-tools` (or deny rules in permission settings for an all-skills block). [^1]

```yaml
---
name: commit
description: Stage and commit the current changes
disable-model-invocation: true
allowed-tools: Bash(git add *) Bash(git commit *) Bash(git status *)
---
```

Security corollary: for skills checked into a project's `.claude/skills/`, `allowed-tools` only takes effect after you accept the workspace-trust dialog, the same gate that governs `.claude/settings.json` permission rules. A skill can grant itself broad tool access, so review project skills before trusting a repo (more in section 7.9). [^1]

You can also block skills wholesale through permissions. Deny the `Skill` tool to disable all skills, or use `Skill(name)` for an exact match and `Skill(name *)` for a prefix match to allow or deny specific ones. A handful of built-in commands (`/init`, `/review`, `/security-review`) are reachable through the Skill tool; most (like `/compact`) are not. [^1]

### Arguments and string substitution

Both you and Claude can pass arguments. The substitutions: [^1]

| Variable | Meaning |
|----------|---------|
| `$ARGUMENTS` | The full argument string as typed. If absent from the body, Claude Code appends `ARGUMENTS: <value>` so the input is still seen. |
| `$ARGUMENTS[N]` | A specific 0-based argument, e.g. `$ARGUMENTS[0]`. |
| `$N` | Shorthand for `$ARGUMENTS[N]` -- `$0`, `$1`, ... |
| `$name` | A named arg declared in the `arguments` frontmatter list (positions map in order). |
| `${CLAUDE_SESSION_ID}` | Current session ID -- for logging or session-scoped files. |
| `${CLAUDE_EFFORT}` | Current effort level (`low`...`max`; "ultracode" reports as `xhigh`). Adapt instructions to effort. |
| `${CLAUDE_SKILL_DIR}` | Directory of this skill's `SKILL.md`. **Essential** for referencing bundled scripts regardless of cwd or install scope. |

Indexed args use shell-style quoting, so `/my-skill "hello world" second` makes `$0` = `hello world`, `$1` = `second`. Escape a literal `$` before a digit/`ARGUMENTS`/arg-name with a backslash (`\$1.00`); this escape was added in v2.1.163. [^9]

### Dynamic context injection (`` !`cmd` ``)

This is the single most powerful Claude-Code-specific feature, and it's worth slowing down for. You run shell commands before Claude ever sees the skill, and splice their output straight into the prompt. [^1]

```yaml
---
description: Summarizes uncommitted changes and flags anything risky. Use when the user asks what changed, wants a commit message, or asks to review their diff.
---

## Current changes

!`git diff HEAD`

## Instructions

Summarize the changes above in two or three bullet points, then list any risks
(missing error handling, hardcoded values, tests that need updating). If the
diff is empty, say there are no uncommitted changes.
```

Mechanics worth knowing:
- This is preprocessing, not something Claude executes. Each `` !`cmd` `` runs first; its stdout replaces the placeholder; Claude only ever sees the rendered result. The skill arrives grounded in your actual working tree rather than what Claude can guess from open files. [^1]
- Substitution runs once over the original file. Command output is inserted as plain text and is not re-scanned, so a command can't emit a placeholder for a later pass.
- The inline form is recognized only when `!` is at line start or after whitespace. `KEY=!`cmd`` is left literal and does not run.
- For multi-line commands, use a fenced block opened with ` ```! `.
- To request deeper reasoning when a skill runs, include `ultrathink` anywhere in the skill content.
- Org-wide kill switch: `"disableSkillShellExecution": true` in settings replaces each command with `[shell command execution disabled by policy]`. Most useful in managed settings (users can't override). Bundled and managed skills are exempt. [^1]

---

## 7.5 Scopes, precedence, and lifecycle

### Where skills live (and who wins)

| Location | Path | Applies to |
|----------|------|------------|
| Enterprise/managed | via managed settings | All users in the org |
| Personal | `~/.claude/skills/<name>/SKILL.md` | All your projects |
| Project | `.claude/skills/<name>/SKILL.md` | This project only |
| Plugin | `<plugin>/skills/<name>/SKILL.md` | Where the plugin is enabled |

Precedence on a name clash runs enterprise over personal over project, and a skill at any of those levels overrides a bundled skill of the same name (a project `code-review` skill replaces bundled `/code-review`). Plugin skills are `plugin-name:skill-name`-namespaced, so they can't conflict. If a skill and a legacy `.claude/commands/` file share a name, the skill wins. [^1]

**Nested / monorepo discovery (v2.1.178).** Project skills load from `.claude/skills/` in your starting directory and every parent up to the repo root, so launching in a subdirectory still picks up root skills. Additionally, when Claude reads or edits a file under a subdirectory, skills from that subdirectory's `.claude/skills/` become available on demand, so a monorepo package ships skills that activate when you work on that package. On a name clash, the nested one appears under a directory-qualified name (`apps/web:deploy`) and both stay available; `/deploy` runs the root one, `/apps/web:deploy` the nested one. [^9] [^1]

**Additional directories.** `--add-dir`/`/add-dir` is normally a file-access grant, but skills are an exception: `.claude/skills/` inside an added dir is loaded automatically. The `permissions.additionalDirectories` setting grants file access only and does not load skills. [^1]

**Live change detection.** Claude Code watches skill directories: adding, editing, or removing a `SKILL.md` under a watched dir takes effect within the current session, no restart. Caveats: (a) creating a top-level skills directory that didn't exist at startup requires a restart so it can be watched; (b) live reload covers `SKILL.md` text only, so for a skill folder that's also a plugin, changes to `hooks/`, `.mcp.json`, `agents/`, `output-styles/` need `/reload-plugins`. There's also `/reload-skills` (v2.1.152) to re-scan manually, and `SessionStart` hooks can return `reloadSkills: true` to surface hook-installed skills in the same session. [^1] [^9]

### The lifecycle gotcha -- the single most important thing to teach

When you or Claude invoke a skill, the rendered `SKILL.md` content enters the conversation as a single message and stays for the rest of the session. Claude Code does not re-read the skill file on later turns. [^1]

Two consequences a senior has to internalize:

1. **Write standing instructions, not one-time commands.** Phrase the body as durable policy ("Whenever editing API routes, validate the request body") and avoid a "do X now" framing, which reads as already-satisfied on every subsequent turn.
2. **Every body line is a recurring token cost.** Because the body persists, conciseness in `SKILL.md` matters even though it loaded lazily. State what to do, and skip the at-length how and why. Apply the same conciseness test you would to `CLAUDE.md`. [^1]

**Behavior across compaction.** Auto-compaction carries invoked skills forward within a token budget. On summarization, Claude Code re-attaches the most recent invocation of each skill after the summary, keeping the first 5,000 tokens of each. Re-attached skills share a combined 25,000-token budget, filled starting from the most recently invoked, so if you fired many skills in one session, the older ones can be dropped entirely after compaction. (Compaction internals live in Ch. 02.) [^1]

**"My skill stopped working."** Most of the time the content is still sitting there and the model is simply choosing other approaches. It's rarely a loading failure. Remedies, in order: (1) strengthen the `description` and instructions so the model keeps preferring it; (2) for rules that must hold, enforce deterministically with a hook instead of leaning on the skill; (3) if it was large or buried under later skills, re-invoke it after compaction to restore the full content. [^1]

### Overriding visibility without editing the skill

For skills you can't or don't want to edit (shared repo, MCP-provided), the `skillOverrides` setting controls visibility from your settings. The `/skills` menu writes it for you (highlight, `Space` to cycle states, `Enter` to save to `.claude/settings.local.json`). [^1]

| Value | Listed to Claude | In `/` menu |
|-------|------------------|-------------|
| `"on"` (default if absent) | Name + description | Yes |
| `"name-only"` | Name only | Yes |
| `"user-invocable-only"` | Hidden | Yes |
| `"off"` | Hidden | Hidden |

Plugin skills aren't affected by `skillOverrides`; manage those via `/plugin`. [^1]

---

## 7.6 Running a skill in a subagent (`context: fork`)

Add `context: fork` to run the skill in an isolated context. The skill content becomes the prompt that drives a subagent, with no access to your conversation history. [^1]

```yaml
---
name: pr-summary
description: Summarize changes in a pull request
context: fork
agent: Explore
allowed-tools: Bash(gh *)
---

## Pull request context
- PR diff: !`gh pr diff`
- PR comments: !`gh pr view --comments`
- Changed files: !`gh pr diff --name-only`

## Your task
Summarize this pull request...
```

Skills and subagents pair in two directions, both valid: [^1]

| Approach | System prompt | Task | Also loads |
|----------|---------------|------|-----------|
| **Skill with `context: fork`** | From the agent type | `SKILL.md` content | `CLAUDE.md`, except when the agent is `Explore`/`Plan` |
| **Subagent with a `skills` field** | The subagent's markdown body | Claude's delegation message | Preloaded skills + `CLAUDE.md` |

Critical caveat: `context: fork` only makes sense for skills that carry explicit instructions. If the skill is pure guidelines ("use these API conventions") with no actionable task, the subagent gets the guidelines and no prompt, and it returns nothing useful. [^1] Pairing `context: fork` with `agent: Explore` is the lean choice for read-only research: Explore and Plan skip `CLAUDE.md` and git status, so the fork sees only `SKILL.md` plus the agent's own system prompt. Minimal context, fast. [^1] (Subagent mechanics, built-ins, and nesting are the home of Ch. 08.)

> Note: subagent preload behaves differently from a normal session. In a regular session only the description sits in context until invocation; for a subagent with preloaded skills, the full skill content is injected at startup. `disable-model-invocation: true` also blocks a skill from being preloaded into subagents. [^1]

---

## 7.7 The two character limits (a real source of confusion)

Two different limits, two different mechanisms. Both are real, and they look like they conflict until you pull them apart:

1. **The platform validation cap: `description` at most 1024 characters.** This is a hard validation rule on the SKILL.md frontmatter, enforced across Anthropic surfaces. The `description` must be non-empty and carry no XML tags. `name` is at most 64 chars, lowercase letters/numbers/hyphens only, no XML tags, no reserved words (`anthropic`, `claude`). [^10] [^4]

2. **The Claude Code listing truncation: combined `description` + `when_to_use` truncated at 1,536 characters in the skill listing.** This is about how much of your skill's metadata Claude Code shows the model at startup, not a hard write limit. It's configurable via `maxSkillDescriptionChars`. [^1]

The practical guidance reconciles both: keep the description well under 1024 chars and put the key use case first. The platform won't accept more than 1024, and Claude Code may shorten the listing under budget pressure, dropping the trailing keywords the model needed to match your request. [^1]

**The skill-listing budget.** All skill names are always included, but if you have many skills, descriptions get shortened to fit a budget that scales at 1% of the model's context window by default. When it overflows, the descriptions of the skills you invoke least are dropped first, so the ones you actually use keep their full text. Levers: raise it with `skillListingBudgetFraction` (e.g. `0.02` = 2%) or `SLASH_COMMAND_TOOL_CHAR_BUDGET` (a fixed char count); free budget by setting low-priority skills to `"name-only"` in `skillOverrides`. Run `/doctor` to see how many descriptions are being shortened or dropped and which skills are affected; the per-launch startup notices were quieted in favor of `/doctor`. [^1]

---

## 7.8 Skill vs command vs MCP vs subagent -- the decision

Anthropic's framing, distilled: MCP gives access to external systems; a skill is procedural knowledge Claude applies itself; a subagent is isolated, context-heavy work; a slash command is now a skill with fewer features, since commands were merged into skills. [^1] The engineering framing is that skills complement MCP by teaching agents the more complex workflows that involve external tools and software, so the right move is usually to compose them. [^2]

| You need... | Reach for | Why |
|-----------|-----------|-----|
| A reusable procedure or knowledge that loads on demand | **Skill** | Progressive disclosure; cheap until used; auto-invokes on description match |
| An explicit, repeatable trigger you control timing on | **Skill with `disable-model-invocation: true`** (the modern "command") | No auto-fire; you decide when it runs |
| Access to an external system / live data | **MCP server** | Skills don't connect to systems; MCP integrates tools |
| Context isolation or parallel work | **Subagent** (or `context: fork` skill for a one-shot forked run) | Own window; only a summary returns |
| A rule that must always hold, no matter what | **Hook** | Skills can be ignored by the model; hooks are deterministic |
| Static project conventions every session | **CLAUDE.md / path-scoped rules** | Always-on facts, not procedures |

The move is to compose these, not pick one. A slash command (skill) can spell out that it should spin up a subagent and call a particular MCP tool, pipelining the whole job. [^1] A skill is excellent for auto-applying a richer workflow with supporting files; reserve subagents for genuine parallelism or context isolation, since each one runs its own window (subagent cost and orchestration are covered in Ch. 08 and Ch. 11).

**Heuristic flowchart:**
- Is it knowledge or a procedure you keep re-explaining? Reach for a **skill**. Front-load only the trigger phrasing; push detail into bundled files.
- Does it have side effects (deploy, send, write to prod)? Skill, but `disable-model-invocation: true`.
- Does it require talking to an outside system? **MCP** (the skill can then orchestrate the MCP tool).
- Is it long, verbose, or context-polluting (test runs, log scans, deep research)? **Subagent** / `context: fork`.
- Must it hold deterministically (no `rm -rf`, always lint)? **Hook**, not a skill.

---

## 7.9 Bundled skills

Claude Code ships bundled skills, available in every session unless disabled with `disableBundledSkills` (or `CLAUDE_CODE_DISABLE_BUNDLED_SKILLS=1`, added v2.1.169). Built-in commands run fixed logic; bundled skills are prompt-based, handing Claude detailed instructions and letting it orchestrate with its tools. You invoke them like any skill (`/name`). [^1] [^9]

The roster, with descriptions drawn from the commands reference (which churns, so verify against `/en/commands`): [^11]

| Skill | What it does |
|-------|--------------|
| `/code-review` | Reviews the current diff for correctness bugs plus reuse/simplification/efficiency cleanups; takes an effort level (`low`...`max`), `--fix` to apply findings, `--comment` to post inline GitHub PR comments, `ultra` for a cloud review, and a path/PR target. |
| `/simplify` | Cleanup-only review: four agents run in parallel over reuse, simplification, efficiency, and altitude (abstraction level), and apply fixes. From v2.1.154 it does not hunt for correctness bugs (use `/code-review` for that); on earlier versions it was equivalent to `/code-review --fix`. [^6] |
| `/batch` | Large-scale parallel codebase changes: researches the codebase, decomposes the work into 5 to 30 independent units, presents a plan, then on approval spawns one background subagent per unit in an isolated git worktree to implement, test, and open a PR. Requires a git repo. |
| `/debug` | Enables debug logging for the current session and troubleshoots by reading the session debug log. Optionally describe the issue to focus the analysis. |
| `/loop` | Runs a prompt repeatedly while the session stays open; omit the interval to let Claude self-pace, omit the prompt to run a maintenance check or `.claude/loop.md`. |
| `/claude-api` | Loads Claude API/SDK reference for your project's language (Python, TypeScript, Java, Go, Ruby, C#, PHP, cURL), auto-activates when code imports `anthropic`/`@anthropic-ai/sdk`, and `migrate` upgrades existing API code to a newer model. (Opus 4.8 support and 4.7->4.8 migration guidance added v2.1.154.) [^6] |
| `/run` | Launches and drives your app to see a change working in the running app, not just tests. (Requires v2.1.145+.) |
| `/verify` | Builds and runs your app to confirm a change does what it should, without falling back to tests or type checks. (Requires v2.1.145+.) |
| `/run-skill-generator` | Records a per-project launch recipe (install commands, env vars, launch script) as `.claude/skills/run-<name>/` so `/run` and `/verify` stop rediscovering it. (Requires v2.1.145+.) |

Anthropic also ships pre-built document skills (PowerPoint `pptx`, Excel `xlsx`, Word `docx`, PDF `pdf`), but those live on claude.ai and the Claude API, not Claude Code (which supports custom skills only). [^4] The open-source skills repository (`github.com/anthropics/skills`) hosts the source for the bundled `claude-api` skill plus others. [^4]

The `/run` + `/verify` + `/run-skill-generator` trio earns a senior's attention. Inference works fine for a standard launch (CLI, server, TUI, browser-driven), then gets unreliable the moment your app needs a database, an env file, a graphical session, or a multi-step build. `/run-skill-generator` gets the app running from a clean environment once, captures the recipe, and commits it as a project skill. After that, every agent in the repo follows the recorded recipe. Run it once per project, and again whenever the build changes. [^1]

---

## 7.10 Authoring good skills

Anthropic's authoring guidance is unusually opinionated; here's the senior-grade distillation. [^10]

### Principle 1 -- Concise is key; the context window is a public good

The default assumption is that Claude is already very smart, so you only add context Claude doesn't already have. Challenge every line. Does Claude really need this? Can I assume it knows this? Does this paragraph justify its token cost? The doc's example makes it concrete: a roughly 50-token "use pdfplumber, here's the snippet" beats a 150-token explanation of what a PDF is. [^10]

### Principle 2 -- Set the right degrees of freedom

Match specificity to the task's fragility: [^10]
- **High freedom** (prose instructions) when many approaches are valid and decisions are contextual, as in code review. Open field, no hazards: give direction, trust Claude to find the route.
- **Medium freedom** (pseudocode / parameterized scripts) when a preferred pattern exists but variation is fine.
- **Low freedom** (exact scripts, no parameters, "do not modify this command") when operations are fragile and consistency is critical, as in a DB migration. Narrow bridge with cliffs: provide exact guardrails.

### Principle 3 -- Write the description for discovery

The `description` is how Claude picks this skill out of potentially 100+ available. Rules: [^10]
- **Always third person.** It's injected into the system prompt; first or second person ("I can help you..." / "You can use this to...") causes discovery problems. Write "Processes Excel files and generates reports."
- **State both what it does and when to use it,** with specific trigger keywords users actually say. Good: "Extract text and tables from PDF files, fill forms, merge documents. Use when working with PDF files or when the user mentions PDFs, forms, or document extraction." Bad: "Helps with documents."
- Consider gerund-form names (`processing-pdfs`, `analyzing-spreadsheets`) and avoid vague ones (`helper`, `utils`, `tools`).

### Principle 4 -- Progressive disclosure in the file layout

- **Keep `SKILL.md` under 500 lines.** Split detail into separate files past that. [^10]
- **Keep references one level deep from `SKILL.md`.** Claude may only partially read files reached through a chain of references (it'll `head -100` a deeply nested file and miss the rest). Link all reference files directly from `SKILL.md`. [^10]
- **Give reference files over 100 lines a table of contents** so a partial read still reveals the full scope. [^10]
- Organize by domain when content is mutually exclusive (`reference/finance.md`, `reference/sales.md`) so a sales question never loads finance schemas. [^10]

### Principle 5 -- Workflows, checklists, and feedback loops

For multi-step tasks, give Claude a copyable checklist it can tick off as it goes, and bake in validator-then-fix-then-repeat loops ("validate after each XML edit; only proceed when it passes"). For high-stakes batch operations, use plan-validate-execute: have Claude write an intermediate plan file (`changes.json`), validate it with a script, then execute, catching errors before they're applied. [^10]

### Principle 6 -- Scripts: solve, don't punt

When a skill bundles scripts: handle error conditions in the script instead of failing and leaving Claude to figure it out; justify every constant (no "voodoo" `TIMEOUT = 47`); list required packages and don't assume they're installed; use forward slashes in paths (Windows backslashes break on Unix); make execution intent explicit ("Run `analyze_form.py`" for execution vs "See `analyze_form.py` for the algorithm" for reference). Pre-made scripts beat generated code: more reliable, fewer tokens, consistent. If your skill uses MCP tools, write fully qualified names (`ServerName:tool_name`) to avoid "tool not found." [^10]

### Principle 7 -- Avoid time-sensitive content

Don't write "before August 2025 use the old API." Put deprecated material in a collapsed "Old patterns" section so it's historical context, and keep the live instructions current. [^10]

### Principle 8 -- Build evals first, then iterate Claude-with-Claude

The highest-leverage habit is to write evaluations before extensive docs. Run Claude on representative tasks without the skill, document the failures, build around three scenarios that test those gaps, establish a baseline, then write the minimal instructions to pass them. That way the skill solves real problems instead of imagined ones. [^10]

The recommended development loop is Claude-with-Claude: use one Claude ("A") to author and refine the skill, and test it with a fresh Claude ("B") that uses it on real tasks, bringing B's failures back to A. A fresh session matters here, because leftover authoring context masks the gaps in the written instructions. Claude understands the skill format natively, so you can simply ask it to write a skill. [^10]

To automate that loop, install the official `skill-creator` plugin (`/plugin install skill-creator@claude-plugins-official`): it stores test cases in `evals/evals.json`, runs each in an isolated subagent recording tokens and duration, grades assertions to `grading.json`, benchmarks with-skill vs without-skill (pass rate, tokens, time) into `benchmark.json`, runs a blind A/B between two versions, and tunes the description against should-trigger / should-not-trigger prompts. [^1]

### Test across the models you'll run

A skill is an addition to a model, so its effectiveness depends on the model. Test with all of them: Haiku (is there enough guidance?), Sonnet (is it clear and efficient?), Opus (does it avoid over-explaining?). What works for Opus may underspecify for Haiku. [^10]

### Authoring checklist (lift to a slide)

Core: description is specific with key terms and states what + when; body under 500 lines; detail in separate files; no time-sensitive info (or in "old patterns"); consistent terminology; concrete examples; references one level deep. Scripts: solve don't punt; explicit error handling; no voodoo constants; packages listed and verified available; forward-slash paths; validation steps for critical ops. Testing: at least 3 evals; tested on Haiku/Sonnet/Opus; tested on real scenarios. [^10]

---

## 7.11 Security -- treat a skill like installing software

A skill grants Claude new capabilities through instructions and code, which is exactly why a malicious one is dangerous. It can direct Claude to invoke tools or run code in ways that have nothing to do with its stated purpose: data exfiltration, unauthorized access, tool misuse. Anthropic's posture is plain. Use skills only from trusted sources (your own, or Anthropic's), and audit any third-party skill thoroughly before use, reading every bundled file (SKILL.md, scripts, images) and watching for unexpected network calls or file access. Skills that fetch from external URLs deserve extra suspicion, since fetched content can carry injected instructions, and even a trustworthy skill can be compromised if its external dependencies change over time. [^4]

For a senior shipping in a team or regulated environment, three concrete controls:
1. **Project skills are committed code.** A PR can add a `.claude/skills/` skill that grants itself broad `allowed-tools`. Review skills in PR review like any other code; the workspace-trust dialog is the gate, not a guarantee. [^1]
2. **`disableSkillShellExecution: true`** in managed settings disables `` !`cmd` `` injection org-wide for user/project/plugin/added-dir skills (bundled and managed exempt). [^1]
3. **`disableBundledSkills`** hides bundled skills, workflows, and built-in slash commands from the model where policy requires a locked-down surface. [^9]

> Data-retention note: Agent Skills is not ZDR-eligible. Skill definitions and execution data are retained per Anthropic's standard policy. Relevant if your org has ZDR requirements. [^4]

---

## 7.12 Troubleshooting quick reference

| Symptom | Fix |
|---------|-----|
| Skill never triggers | Add the keywords users actually say to `description`; confirm via "What skills are available?"; rephrase the request; or invoke directly with `/name`. [^12] |
| Skill triggers too often | Make `description` more specific; or `disable-model-invocation: true` for manual-only. [^12] |
| Skill "stops working" after the first turn | Content is usually still there -- strengthen `description`/instructions, enforce with a hook, or re-invoke after compaction. (See section 7.5 lifecycle.) [^12] |
| Descriptions look truncated | Run `/doctor`; raise `skillListingBudgetFraction`; set low-priority skills to `"name-only"`; put the key use case first in `description`. [^12] |
| New skill not picked up | If you created a top-level skills dir mid-session, restart; otherwise edits hot-reload. Try `/reload-skills`. [^12] |
| Plugin skill loads even with `disable-model-invocation` | Known version-sensitive bug (issue #22345, v2.1.29, still open) -- verify current behavior. [^13] |
| `/skill-name` vs `/dir:skill-name` confusion | Command name comes from the path, not frontmatter `name`; nested clashes get a `dir:name` qualifier. (See section 7.3.) [^12] |

`/doctor` and the [Debug your configuration](https://code.claude.com/docs/en/debug-your-config) page diagnose why a skill isn't appearing or triggering. [^1]

---

## Sources

Official Anthropic:
- Agent Skills (Claude Code) -- https://code.claude.com/docs/en/skills
- Slash commands / bundled skills reference -- https://code.claude.com/docs/en/commands
- Claude Code changelog -- https://code.claude.com/docs/en/changelog.md
- Debug your configuration -- https://code.claude.com/docs/en/debug-your-config
- Agent Skills overview (Claude Developer Platform) -- https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview
- Agent Skills best practices -- https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices
- Equipping agents for the real world with Agent Skills -- https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills
- Introducing Agent Skills -- https://claude.com/blog/skills
- Agent Skills open standard -- https://agentskills.io
- claude-code issue #19141 (user-invocable clarification) -- https://github.com/anthropics/claude-code/issues/19141
- claude-code issue #22345 (plugin disable-model-invocation bug) -- https://github.com/anthropics/claude-code/issues/22345

[^1]: web | 2026-06-18 | [https://code.claude.com/docs/en/skills](https://code.claude.com/docs/en/skills)
[^2]: web | 2026-06-18 | [https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)
[^3]: web | 2026-06-18 | [https://claude.com/blog/skills](https://claude.com/blog/skills)
[^4]: web | 2026-06-18 | [https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview)
[^5]: web \ | 2026-06-18 \ | [https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview)
[^6]: web \ | 2026-06-18 \ | [https://code.claude.com/docs/en/changelog.md](https://code.claude.com/docs/en/changelog.md)
[^7]: web | 2026-06-18 | [https://github.com/anthropics/claude-code/issues/19141](https://github.com/anthropics/claude-code/issues/19141)
[^8]: web | 2026-06-18 | [https://github.com/anthropics/claude-code/issues/22345](https://github.com/anthropics/claude-code/issues/22345)
[^9]: web | 2026-06-18 | [https://code.claude.com/docs/en/changelog.md](https://code.claude.com/docs/en/changelog.md)
[^10]: web | 2026-06-18 | [https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)
[^11]: web | 2026-06-18 | [https://code.claude.com/docs/en/commands](https://code.claude.com/docs/en/commands)
[^12]: web \ | 2026-06-18 \ | [https://code.claude.com/docs/en/skills](https://code.claude.com/docs/en/skills)
[^13]: web \ | 2026-06-18 \ | [https://github.com/anthropics/claude-code/issues/22345](https://github.com/anthropics/claude-code/issues/22345)
