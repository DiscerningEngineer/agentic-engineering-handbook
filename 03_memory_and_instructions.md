# Memory and Instructions

**TL;DR.** Every Claude Code session opens with a fresh context window that remembers nothing of yesterday. Two mechanisms carry knowledge across that gap. CLAUDE.md files hold the instructions you write; auto memory holds the notes Claude writes to itself. Both load at the start of every conversation, and both arrive as context, not enforced configuration. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "CLAUDE.md vs auto memory"] Claude reads them and tries to comply, and that trying is the whole of the guarantee. Memory is the most token-efficient lever you have and the easiest one to ruin, because you pay for it on every single session, which makes signal density the only thing that matters. This chapter walks the instruction hierarchy (managed, user, project, local), how files load and resolve, `@`-imports and their cost model, path-scoped rules in `.claude/rules/`, auto memory and `MEMORY.md`, subagent memory, `/init` and `/memory`, what survives `/compact`, and the quiet discipline of a short, load-bearing memory file. When a behavior has to hold no matter what Claude decides, the answer is a hook, not a memory file (see Ch. 09).

> **Version posture.** Claude Code ships near-daily. The line counts, byte thresholds, settings keys, and version floors below are accurate as of 2026-06-18 against the official docs. Verify version-pinned details against the live changelog (`https://code.claude.com/docs/en/changelog.md`) before relying on them in a live or teaching context.

---

## The mental model: memory is context you pay for every session

Here is the reframe that changes how a senior treats this whole subject. A CLAUDE.md is not a config file. Its content is delivered as a user message after the system prompt, every session, into a finite and degrading context window. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "CLAUDE.md content is delivered as a user message after the system prompt, not as part of the system prompt itself"] Once you see it that way, the rest follows on its own.

Three consequences land immediately.

It costs tokens forever. A 400-line CLAUDE.md is 400 lines of tax on every conversation, competing for attention with the work you actually opened the session to do. The official guidance is to target under 200 lines per CLAUDE.md file, because "longer files consume more context and reduce adherence." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "Write effective instructions" / Size]

It is influence. "Claude treats them as context, not enforced configuration. To block an action regardless of what Claude decides, use a PreToolUse hook instead." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "CLAUDE.md vs auto memory"] A rule that must hold (no force-push to `main`, no `rm -rf`, run the formatter after every edit) belongs in a hook or in `permissions.deny`, somewhere Claude cannot rationalize its way past in prose.

Specificity drives adherence. "The more specific and concise your instructions, the more consistently Claude follows them." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "CLAUDE.md vs auto memory"] A vague rule ("format code properly") is nearly free to write and nearly worthless. A concrete rule ("use 2-space indentation"; "run `npm test` before committing") is the one that moves behavior.

### The two complementary systems

| | CLAUDE.md files | Auto memory |
|---|---|---|
| Who writes it | You | Claude |
| What it contains | Instructions and rules | Learnings and patterns Claude discovers |
| Scope | Project, user, or org | Per repository, shared across worktrees |
| Loaded into | Every session | Every session (first 200 lines or 25KB of `MEMORY.md`) |
| Use for | Coding standards, workflows, architecture | Build commands, debugging insights, preferences Claude discovers |

^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "CLAUDE.md vs auto memory" comparison table]

The division of labor is clean once you say it plainly. CLAUDE.md is for the standing knowledge you already hold and can state up front. Auto memory is for what Claude figures out by working in the repo and getting corrected. A senior uses both, and a senior audits both.

---

## The instruction hierarchy (managed, user, project, local)

CLAUDE.md files live in several places, each with a different reach. Anthropic documents them in load order, from broadest scope to most specific, and the part that catches people is that all discovered files are concatenated, not overridden. A project instruction lands in context after a user instruction, so when guidance conflicts, the more-local file is read last and tends to win Claude's attention. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "Choose where to put CLAUDE.md files"]

### The four tiers and their exact locations

| Scope | Location | Purpose | Shared with |
|---|---|---|---|
| **Managed policy** | macOS: `/Library/Application Support/ClaudeCode/CLAUDE.md`<br>Linux/WSL: `/etc/claude-code/CLAUDE.md`<br>Windows: `C:\Program Files\ClaudeCode\CLAUDE.md` | Org-wide instructions managed by IT/DevOps (security, compliance, company standards) | All users on the machine |
| **User** | `~/.claude/CLAUDE.md` | Personal preferences across all projects | Just you, every project |
| **Project** | `./CLAUDE.md` **or** `./.claude/CLAUDE.md` | Team-shared project instructions (architecture, build/test commands, conventions) | Team via source control |
| **Local** | `./CLAUDE.local.md` | Personal project-specific preferences; add to `.gitignore` | Just you, this project |

^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "Choose where to put CLAUDE.md files" scope table]

Two subtleties earn a senior's attention. The first is that managed policy cannot be excluded. Organizations deploy it via MDM, Group Policy, or Ansible, and "this file cannot be excluded by individual settings." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "Deploy organization-wide CLAUDE.md"] This is the mechanism by which a regulated org pins compliance reminders into every session on every machine, whether the engineer at the keyboard wants them there or not.

The second is the `claudeMd` settings key. Instead of shipping a separate managed CLAUDE.md file, an org can embed the content directly in `managed-settings.json` via the `claudeMd` key. It is honored in managed and policy settings only; "setting `claudeMd` in user, project, or local settings has no effect." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "The claudeMd key" / "Where it's honored"]

```json
// managed-settings.json
{
  "claudeMd": "Always run `make lint` before committing.\nNever push directly to main."
}
```

### Managed CLAUDE.md vs. managed settings -- pick the right enforcement layer

This is the distinction that trips up teams. "Settings rules are enforced by the client regardless of what Claude decides to do. CLAUDE.md instructions shape Claude's behavior but are not a hard enforcement layer." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "A managed CLAUDE.md and managed settings serve different purposes"] One is a suggestion the model weighs. The other is a wall the model meets.

| Concern | Configure in |
|---|---|
| Block specific tools/commands/paths | Managed settings: `permissions.deny` |
| Enforce sandbox isolation | Managed settings: `sandbox.enabled` |
| Env vars / API provider routing | Managed settings: `env` |
| Auth method / org lock | Managed settings: `forceLoginMethod`, `forceLoginOrgUUID` |
| Code style and quality guidelines | Managed CLAUDE.md |
| Data-handling / compliance reminders | Managed CLAUDE.md |
| Behavioral instructions for Claude | Managed CLAUDE.md |

^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "A managed CLAUDE.md and managed settings serve different purposes" table]

> **Rule of thumb.** If the failure mode is "Claude forgot the convention," reach for CLAUDE.md or a rule. If the failure mode is "Claude did the dangerous thing anyway," reach for a hook or `permissions.deny`.

---

## How CLAUDE.md files load and resolve

The loading model is a directory-tree walk. Knowing it precisely is the line between flailing at "my rule isn't working" and a five-second diagnosis.

### Ancestor walk and concatenation

Claude Code reads CLAUDE.md files by walking up the directory tree from your current working directory, checking each directory for `CLAUDE.md` and `CLAUDE.local.md`. Run Claude in `foo/bar/` and it loads `foo/bar/CLAUDE.md`, `foo/CLAUDE.md`, and any `CLAUDE.local.md` alongside them. "All discovered files are concatenated into context rather than overriding each other." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "How CLAUDE.md files load"]

The concatenation order is precise. Across the tree, content is ordered from the filesystem root down to your working directory, so `foo/CLAUDE.md` appears in context before `foo/bar/CLAUDE.md`, and "instructions closer to where you launched Claude are read last." Within a directory, `CLAUDE.local.md` is appended after `CLAUDE.md`, so your personal notes are the last thing Claude reads at that level. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "How CLAUDE.md files load"]

The net effect is that most-local, most-personal wins the recency battle. Read this carefully, because it is a tendency rather than a formal override: the docs warn that when two files give different guidance for the same behavior, "Claude may pick one arbitrarily." Audit for conflicts. Do not lean on ordering to resolve them quietly for you. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "Consistency" / "Look for conflicting instructions"]

### Front-loaded vs. on-demand (the monorepo lever)

"CLAUDE.md and CLAUDE.local.md files in the directory hierarchy above the working directory are loaded in full at launch. Files in subdirectories load on demand when Claude reads files in those directories." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "Choose where to put CLAUDE.md files"]

That asymmetry is the whole foundation of the monorepo pattern. You keep a lean root CLAUDE.md and add per-package `packages/web/CLAUDE.md`, `packages/api/CLAUDE.md` that cost nothing until Claude reaches into that package. For the full layout, the docs point to a dedicated "Monorepos and large repos" page. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/large-codebases]

### Startup load order (what's in context before you type)

Anthropic's interactive context-window visualization documents the actual startup sequence. Before your first prompt, in order, the window fills with the system prompt, then auto memory (`MEMORY.md`), then environment info, then MCP tool names (deferred), then the skill index (one-line descriptions), then your CLAUDE.md files, with the git branch, status, and recent commits loading "as a separate block at the very end of the system prompt." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/context-window -- startup event sequence] The token counts shown there are illustrative and shift with your CLAUDE.md size, MCP servers, and file lengths.

> **From the doc, verbatim.** "A lot loads before you type anything. CLAUDE.md, memory, skills, and MCP tools are all in context before your first prompt." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/context-window -- startup takeaway] Your prompt is a small thing dropped into a room already crowded with furniture, which is exactly why a bloated memory file hurts.

### Exclusions, additional dirs, and HTML comments

`claudeMdExcludes` skips specific CLAUDE.md or rules files by path or glob, "matched against absolute file paths using glob syntax." Put it in `.claude/settings.local.json` to keep exclusions machine-local. It is configurable at any settings layer, and arrays "merge across layers." Managed-policy CLAUDE.md cannot be excluded. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "Exclude specific CLAUDE.md files"]

```json
// .claude/settings.local.json
{
  "claudeMdExcludes": [
    "**/monorepo/CLAUDE.md",
    "/home/user/monorepo/other-team/.claude/rules/**"
  ]
}
```

The `--add-dir` flag gives Claude access to extra directories, but "by default, CLAUDE.md files from these directories are not loaded." To opt in, set `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1`, which loads `CLAUDE.md`, `.claude/CLAUDE.md`, `.claude/rules/*.md`, and `CLAUDE.local.md` from the additional directory (local is skipped if you exclude `local` from `--setting-sources`). ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "Load from additional directories"]

Block-level HTML comments (`<!-- maintainer notes -->`) in CLAUDE.md "are stripped before the content is injected into Claude's context," which gives you free notes for human maintainers that cost zero context tokens. Comments inside code blocks are preserved, and all comments stay visible if you open the file with the Read tool. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "How CLAUDE.md files load"]

---

## `@`-imports -- organization, not token savings

CLAUDE.md can import other files with `@path/to/import` syntax. Here is the fact a senior has to hold onto, because it is the one that gets misunderstood: "Imported files are expanded and loaded into context at launch alongside the CLAUDE.md that references them." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "Import additional files"]

```text
See @README for project overview and @package.json for available npm commands.

# Additional Instructions
- git workflow @docs/git-instructions.md
```

The rules of the road, all from the same doc section: ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "Import additional files"]

Relative and absolute paths both work, and relative paths "resolve relative to the file containing the import, not the working directory." Recursive imports are allowed, "with a maximum depth of four hops." To mention a path without importing it, wrap it in backticks: `` `@README` `` stays literal while `@README` outside backticks imports, and import parsing "skips Markdown code spans and fenced code blocks." The first time Claude encounters external imports it shows an approval dialog listing the files, and "if you decline, the imports stay disabled and the dialog does not appear again."

The teaching point that matters most lives just under the syntax. The `@`-import helps you organize, splitting a long file into named pieces a human can navigate, and that is the entire benefit. It does nothing for context cost. "Splitting into `@path` imports helps organization but does not reduce context, since imported files load at launch." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "My CLAUDE.md is too large"] When you genuinely want to defer the token cost, the tools are path-scoped rules (next section) and skills (see Ch. 06). The import only rearranges the furniture; it never makes the room bigger.

### Cross-worktree personal instructions

A gitignored `CLAUDE.local.md` exists only in the worktree where you created it. To share personal instructions across every worktree of the same repo, "import a file from your home directory instead." The home file is shared, and the import line rides along in each worktree. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "Import additional files"]

```text
# Individual Preferences
- @~/.claude/my-project-instructions.md
```

### AGENTS.md interop

"Claude Code reads `CLAUDE.md`, not `AGENTS.md`." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "AGENTS.md"] AGENTS.md is itself a real, primary-documented standard: per the format's own site, "AGENTS.md is now stewarded by the Agentic AI Foundation under the Linux Foundation," it emerged from collaboration among Codex, Amp, Jules, Cursor, and Factory, and it is "used by over 60k open-source projects." ^[source: web | 2026-06-18 | https://agents.md -- overview / "stewarded by the Agentic AI Foundation under the Linux Foundation" / "used by over 60k open-source projects"] If your repo already standardizes on it, do not duplicate the content into a second file that will drift. Import it and append your Claude-specific notes:

```markdown
@AGENTS.md

## Claude Code
Use plan mode for changes under `src/billing/`.
```

A symlink (`ln -s AGENTS.md CLAUDE.md`) works when you do not need Claude-specific content, with one exception: "On Windows, creating a symlink requires Administrator privileges or Developer Mode, so use the `@AGENTS.md` import instead." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "AGENTS.md"] Running `/init` in a repo that already has `AGENTS.md` "reads it and incorporates the relevant parts into the generated `CLAUDE.md`. It also reads other tool configs like `.cursorrules`, `.devin/rules/`, and `.windsurfrules`." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "AGENTS.md"]

> **A figure to distrust.** The "AGENTS.md cuts bugs by 35-55%" claim that circulates online appears nowhere on the format's own site or in Anthropic's docs. There is no primary support for any bug-reduction metric, so treat it as folklore and do not repeat it as fact.

---

## `.claude/rules/` -- path-scoped just-in-time memory

This is the primary lever for keeping memory lean in a large project, and the feature most under-used by engineers who only ever learned about CLAUDE.md.

Place markdown files in `.claude/rules/`. Each file covers one topic with a descriptive name (`testing.md`, `api-design.md`). "All `.md` files are discovered recursively," so you can nest them in `frontend/`, `backend/`, and so on. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "Set up rules"]

```text
your-project/
+-- .claude/
|   +-- CLAUDE.md           # Main project instructions
|   +-- rules/
|       +-- code-style.md
|       +-- testing.md
|       +-- security.md
```

### Two loading behaviors, controlled by frontmatter

Rules without a `paths` field "are loaded at launch with the same priority as `.claude/CLAUDE.md`." These are unconditional. Rules with a `paths` field are conditional and just-in-time: "Path-scoped rules trigger when Claude reads files matching the pattern, not on every tool use." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "Set up rules" / "Path-specific rules"]

```markdown
---
paths:
  - "src/api/**/*.ts"
---

# API Development Rules
- All API endpoints must include input validation
- Use the standard error response format
- Include OpenAPI documentation comments
```

Glob support covers extension matching, directory matching, project-root matching, and brace expansion across extensions: ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "Path-specific rules" glob table]

| Pattern | Matches |
|---|---|
| `**/*.ts` | All TypeScript files, any directory |
| `src/**/*` | Everything under `src/` |
| `*.md` | Markdown files in the project root |
| `src/components/*.tsx` | React components in one directory |

```markdown
---
paths:
  - "src/**/*.{ts,tsx}"
  - "lib/**/*.ts"
  - "tests/**/*.test.ts"
---
```

### Why this solves the monolithic-CLAUDE.md problem

Cram everything into one CLAUDE.md and every instruction competes for attention on every task. Your API rules tug at Claude even while it is editing CSS, which is wasted pull and wasted tokens. Path-scoped rules instead let instructions "load only when Claude works with matching files, reducing noise and saving context space." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "Organize rules with .claude/rules/"] For a CLAUDE.md that has crept past 200 lines, this is the single highest-leverage refactor on the table.

### Sharing, user-level, and priority

The `.claude/rules/` directory supports symlinks, so you can link a shared rule set or an individual file into many projects, and "circular symlinks are detected and handled gracefully." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "Share rules across projects with symlinks"]

```bash
ln -s ~/shared-claude-rules .claude/rules/shared
ln -s ~/company-standards/security.md .claude/rules/security.md
```

User-level rules live in `~/.claude/rules/` and apply to every project on your machine. They "are loaded before project rules, giving project rules higher priority." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "User-level rules"]

### Rules vs. CLAUDE.md vs. skills -- the decision

| You want | Use |
|---|---|
| Facts Claude should hold **every** session (build cmds, layout, "always X") | **CLAUDE.md** |
| Conventions that only matter when touching **certain files** | **Path-scoped rule** in `.claude/rules/` |
| A **multi-step procedure** or task-specific knowledge that need not be in context all the time | **Skill** (loads on invoke or when relevant -- see Ch. 06) |
| A rule that **must always hold** regardless of Claude's judgment | **Hook** / `permissions.deny` (see Ch. 09) |

The doc puts it without hedging: "If an entry is a multi-step procedure or only matters for one part of the codebase, move it to a skill or a path-scoped rule instead." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "When to add to CLAUDE.md"]

---

## Auto memory and `MEMORY.md`

Auto memory lets Claude accumulate knowledge across sessions without you writing anything. "Claude saves notes for itself as it works: build commands, debugging insights, architecture notes, code style preferences, and workflow habits." It does not save something every session; "it decides what's worth remembering based on whether the information would be useful in a future conversation." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "Auto memory"]

> **Version floor.** Auto memory "requires Claude Code v2.1.59 or later." Check with `claude --version`. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "Auto memory" note]

### Storage and scoping

"Each project gets its own memory directory at `~/.claude/projects/<project>/memory/`." The `<project>` path "is derived from the git repository, so all worktrees and subdirectories within the same repo share one auto memory directory." Outside a git repo, the project root is used. Auto memory "is machine-local," and "files are not shared across machines or cloud environments." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "Storage location"]

```text
~/.claude/projects/<project>/memory/
+-- MEMORY.md          # Concise index, loaded into every session
+-- debugging.md       # Detailed notes on debugging patterns
+-- api-conventions.md # API design decisions
+-- ...                # Any other topic files Claude creates
```

Relocate it with `autoMemoryDirectory` in `settings.json`, read "from any settings scope: user, project, local, policy, or `--settings`." The value "must be an absolute path or start with `~/`." When set in a project's `.claude/settings.json` or `.claude/settings.local.json`, it "is honored only after you accept the workspace trust dialog for that folder, the same gate that governs hooks." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "Storage location"]

```json
{ "autoMemoryDirectory": "~/my-custom-memory-dir" }
```

### The load mechanics and the one hard limit

This is the detail seniors most often get wrong, so here it is stated flat.

"The first 200 lines of `MEMORY.md`, or the first 25KB, whichever comes first, are loaded at the start of every conversation. Content beyond that threshold is not loaded at session start." This limit applies only to `MEMORY.md`: "CLAUDE.md files are loaded in full regardless of length, though shorter files produce better adherence." And topic files such as `debugging.md` "are not loaded at startup. Claude reads them on demand using its standard file tools when it needs the information." `MEMORY.md` "acts as an index of the memory directory," and Claude "keeps `MEMORY.md` concise by moving detailed notes into separate topic files." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "How it works"]

So `MEMORY.md` plays the same role for auto memory that a lean root CLAUDE.md plays for instructions: a small, high-signal index pointing at deferred detail. "When you see 'Writing memory' or 'Recalled memory' in the Claude Code interface, Claude is actively updating or reading from" that directory. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "How it works"]

### Enable / disable

On by default. Toggle the auto-memory switch in `/memory`, or set it in settings:

```json
{ "autoMemoryEnabled": false }
```

Or disable via env var: "set `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1`." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "Enable or disable auto memory"]

### Audit it -- non-negotiable for a senior

"Auto memory files are plain markdown you can edit or delete at any time." Run `/memory` "and select the auto memory folder to browse what Claude has saved." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "Audit and edit your memory" / "I don't know what auto memory saved"] Because Claude writes this on its own, an un-audited `MEMORY.md` quietly collects stale build commands and the occasional wrong assumption, and that wrong assumption then steers every session that comes after it. Periodic review is the price of the convenience, and it is a price worth paying.

### Telling Claude what to remember, and where

"When you ask Claude to remember something, like 'always use pnpm, not npm' or 'remember that the API tests require a local Redis instance,' Claude saves it to auto memory." To land it in CLAUDE.md instead, "ask Claude directly, like 'add this to CLAUDE.md,' or edit the file yourself via `/memory`." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "View and edit with /memory"] The distinction is load-bearing. Auto memory is machine-local and Claude-curated. CLAUDE.md is committed, team-shared, and yours.

---

## Subagent memory

Subagents can keep their own persistent memory, sealed off from the main session. "The `memory` field gives the subagent a persistent directory that survives across conversations," a place where it builds up "codebase patterns, debugging insights, and architectural decisions" over time. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/sub-agents -- "Enable persistent memory"]

```yaml
---
name: code-reviewer
description: Reviews code for quality and best practices
memory: user
---
You are a code reviewer. As you review code, update your agent memory with
patterns, conventions, and recurring issues you discover.
```

Scope determines location. The doc names `project` as "the recommended default scope" because it makes subagent knowledge shareable via version control. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/sub-agents -- "Enable persistent memory" table + "Persistent memory tips"]

| Scope | Location | Use when |
|---|---|---|
| `user` | `~/.claude/agent-memory/<name-of-agent>/` | Learnings apply across all projects |
| `project` | `.claude/agent-memory/<name-of-agent>/` | Project-specific, shareable via version control (**recommended default**) |
| `local` | `.claude/agent-memory-local/<name-of-agent>/` | Project-specific but should not be committed |

When memory is enabled, three things happen: the subagent's system prompt gets read/write instructions for the directory; it "also includes the first 200 lines or 25KB of `MEMORY.md` in the memory directory, whichever comes first, with instructions to curate `MEMORY.md` if it exceeds that limit" (the same threshold as main-session auto memory); and "Read, Write, and Edit tools are automatically enabled so the subagent can manage its memory files." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/sub-agents -- "Enable persistent memory"]

Two facts here are worth internalizing rather than just reading. The first is that a custom subagent's system prompt "replaces the default Claude Code system prompt entirely, the same way `--system-prompt` does," and yet "`CLAUDE.md` files and project memory still load through the normal message flow." So your project conventions reach the subagent even though its system prompt is bespoke. The built-in Explore and Plan agents are the exception: they "skip your CLAUDE.md files and the parent session's git status to keep research fast and inexpensive," and there is "no frontmatter field or per-agent setting to change which agents skip them." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/sub-agents -- "Run the whole session as a subagent" / "Built-in subagents" / "What loads at startup"]

The second is the workflow the docs recommend, which closes the loop on memory being useful at all. Ask the subagent to consult its memory before ("check your memory for patterns you've seen before") and update it after ("save what you learned to your memory"). Bake those instructions into the subagent's markdown body and "it proactively maintains its own knowledge base," session after session, without you in the loop. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/sub-agents -- "Persistent memory tips"] For the full subagent mechanics (built-ins, tools, isolation, frontmatter roster), see Ch. 07.

---

## `/init` and `/memory`

### `/init` -- bootstrap a CLAUDE.md from your codebase

`/init` "analyzes your codebase and creates a file with build commands, test instructions, and project conventions it discovers." If a CLAUDE.md already exists, "`/init` suggests improvements rather than overwriting it," so re-running it on a maturing project is safe. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "Set up a project CLAUDE.md" tip]

There is an advanced flow worth knowing about. "Set `CLAUDE_CODE_NEW_INIT=1` to enable an interactive multi-phase flow." In that mode, `/init` "asks which artifacts to set up: CLAUDE.md files, skills, and hooks. It then explores your codebase with a subagent, fills in gaps via follow-up questions, and presents a reviewable proposal before writing any files." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "Set up a project CLAUDE.md" tip] As an env-gated feature, verify it is still the gate (or has graduated to default) against the changelog before teaching it.

> **Senior posture on `/init`.** Treat its output as a first draft, not the finished file. It captures what is discoverable from the code (commands, structure); it cannot capture the judgment calls, the "we tried X and it broke," or the hard rules that are not visible in the source. The official advice: "Refine from there with instructions Claude wouldn't discover on its own." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "Set up a project CLAUDE.md" tip]

### `/memory` -- the first thing to run when a rule isn't followed

`/memory` "lists all CLAUDE.md, CLAUDE.local.md, and rules files loaded in your current session, lets you toggle auto memory on or off, and provides a link to open the auto memory folder. Select any file to open it in your editor." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "View and edit with /memory"]

Make this your reflex. "If a file isn't listed, Claude can't see it," and that is the answer to nearly every "my rule isn't working." The usual culprits are mundane: the file sits in a subdirectory Claude has not read into yet, so its nested CLAUDE.md never loaded, or it is a path-scoped rule whose glob has matched nothing this session. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "Claude isn't following my CLAUDE.md"]

---

## What survives `/compact`

Compaction swaps the verbatim conversation for a structured summary to buy back space (the internals live in Ch. 02). The detail that matters for memory hygiene is where the startup content lives: it sits outside the message history and reloads automatically. The doc states it cleanly: "Compaction replaces the conversation with a structured summary. System prompt, CLAUDE.md, memory, and MCP tools reload automatically. The skill listing is the one exception. Only skills you actually invoked are preserved." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/context-window -- compaction takeaway]

The breakdown after `/compact`:

| Startup item | Survives `/compact`? |
|---|---|
| System prompt | Reloads |
| Project-root CLAUDE.md | Re-read from disk and re-injected |
| Auto memory / `MEMORY.md` | Reloads |
| MCP tools | Reload |
| Skill index | **Not re-injected** -- only invoked skills are preserved |
| Nested (subdirectory) CLAUDE.md | **Not re-injected automatically** -- reloads on next read in that dir |
| Conversation-only instructions | **Do not survive** -- the summary may drop them |

^[source: web | 2026-06-18 | https://code.claude.com/docs/en/context-window -- compaction takeaway] ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "Instructions seem lost after /compact"]

The doc is explicit on the two exceptions. "Project-root CLAUDE.md survives compaction: after `/compact`, Claude re-reads it from disk and re-injects it into the session. Nested CLAUDE.md files in subdirectories are not re-injected automatically; they reload the next time Claude reads a file in that subdirectory." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "Instructions seem lost after /compact"]

The rule that falls out of all of this is short. Persistent rules belong in files, never in chat. "If an instruction disappeared after compaction, it was either given only in conversation or lives in a nested CLAUDE.md that hasn't reloaded yet. Add conversation-only instructions to CLAUDE.md to make them persist." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "Instructions seem lost after /compact"] This is the real reason a CLAUDE.md beats repeating yourself across a long session. It is one of the few things that walks through a compaction and comes out the other side intact.

---

## What makes a good memory file (short, load-bearing, signal-dense)

Anthropic's writing guidance comes down to four properties. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "Write effective instructions"]

Size. Target under 200 lines per CLAUDE.md, because "longer files consume more context and reduce adherence." When it grows, push detail into path-scoped rules (which defers the cost), not into `@`-imports (which does not).

Structure. Use markdown headers and bullets to group related rules: "Claude scans structure the same way readers do: organized sections are easier to follow than dense paragraphs."

Specificity. Write rules concrete enough to verify:
- "Use 2-space indentation" instead of "Format code properly"
- "Run `npm test` before committing" instead of "Test your changes"
- "API handlers live in `src/api/handlers/`" instead of "Keep files organized"

Consistency. Contradicting rules make Claude "pick one arbitrarily." Periodically review CLAUDE.md, nested CLAUDE.md, and `.claude/rules/` to clear out stale or conflicting instructions.

### When to add an entry, and when not to

The doc frames CLAUDE.md as "the place you write down what you'd otherwise re-explain." Add when: ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "When to add to CLAUDE.md"]

- Claude makes the same mistake a second time.
- A code review catches something Claude should have known about this codebase.
- You type the same correction into chat that you typed last session.
- A new teammate would need the same context to be productive.

> **The litmus test.** Would removing this line cause a specific, predictable mistake? If you cannot name the mistake, the line is fluff, so delete it. Every line you keep is taxed on every session forever.

### What to keep out

Some things look like they belong in a memory file and quietly earn their keep nowhere. Anything CI already enforces is the first of them: if the linter fails the build, "use 2-space indentation" in CLAUDE.md is redundant, unless you specifically want Claude to produce conforming code on the first pass, which is a legitimate reason to keep it as long as you know that is why it is there. Personality and tone fluff rarely changes outcomes and always costs tokens. Secrets never go in at all, because project CLAUDE.md is committed and local and managed files get read by tooling. Stale architecture is the dangerous one, because a wrong rule actively points Claude in the wrong direction, which is worse than a gap. And multi-step procedures and file-specific conventions belong in skills and path-scoped rules respectively, where the doc itself sends them. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "When to add to CLAUDE.md"]

### A worked skeleton (project CLAUDE.md)

```markdown
# <Project> -- Agent Instructions

## Commands
- Build: `make build`
- Test:  `npm test` (run before every commit)
- Lint:  `make lint`

## Architecture (terse map)
- API handlers: `src/api/handlers/`
- Domain logic: `src/core/`
- Migrations:   `db/migrations/` -- never edit applied migrations; add a new one

## Hard rules
- Never push directly to `main`.
- All new endpoints need input validation + an integration test.
- Money is cents-as-integer; never floats.

<!-- maintainer note: billing rules live in .claude/rules/billing.md (path-scoped) -->
```

Keep the root file at this altitude and let `.claude/rules/billing.md` (`paths: ["src/billing/**"]`) carry the billing-specific detail, so it only shows up in context when there is billing work in front of it.

---

## Diagnosing "Claude isn't following my instructions"

A repeatable triage, in order, drawn from the official troubleshooting section: ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "Claude isn't following my CLAUDE.md"]

1. **Run `/memory`.** Confirm the file is actually loaded. Not listed means Claude cannot see it (wrong location, a nested file not yet reached, or a path-rule whose glob has not matched).
2. **Check location against the load rules.** Is it an ancestor (loads at launch) or a subdirectory (loads only when Claude reads files there)?
3. **Make the rule more specific.** Vague gets ignored.
4. **Hunt for contradictions** across all loaded CLAUDE.md and rules. Conflicts produce an arbitrary choice.
5. **If it must hold deterministically, stop using memory.** Write a hook, which "executes as shell commands at fixed lifecycle events and applies regardless of what Claude decides to do," or use `permissions.deny`.
6. **If you need it at the system-prompt level**, use `--append-system-prompt`, but note it "must be passed every invocation, so it's better suited to scripts and automation than interactive use."

### The deep-debug tool: the `InstructionsLoaded` hook

When you need to see which instruction files loaded, when, and why, reach for the `InstructionsLoaded` hook. It "fires when a `CLAUDE.md` or `.claude/rules/*.md` file is loaded into context. This event fires at session start for eagerly-loaded files and again later when files are lazily loaded." It receives a payload with `file_path`, `memory_type` (`User`, `Project`, `Local`, or `Managed`), `load_reason`, and, for lazy and include loads, `globs`, `trigger_file_path`, and `parent_file_path`. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/hooks -- "InstructionsLoaded"]

The matcher runs against `load_reason`, and the documented values let you tell apart every way a file can enter context: `session_start`, `nested_traversal`, `path_glob_match`, `include`, and `compact`. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/hooks -- "InstructionsLoaded" matcher values] One constraint to internalize: "The hook does not support blocking or decision control. It runs asynchronously for observability purposes." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/hooks -- "InstructionsLoaded"] Use it for audit logging, compliance tracking, and observability. It cannot reject a load.

One sentence holds the whole chapter, so keep it close. CLAUDE.md content "is delivered as a user message after the system prompt, not as part of the system prompt itself. Claude reads it and tries to follow it, but there's no guarantee of strict compliance, especially for vague or conflicting instructions." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/memory -- "Claude isn't following my CLAUDE.md"] Memory is influence. It was never control.

---

## Decision tables

**Where does this instruction go?**

| Property of the instruction | Put it in |
|---|---|
| True everywhere, every project, you | `~/.claude/CLAUDE.md` (user) |
| True for the whole team on this project | `./CLAUDE.md` or `./.claude/CLAUDE.md` (project, committed) |
| Yours alone, this project, not committed | `./CLAUDE.local.md` (gitignored) |
| Org-mandated, cannot be skipped | Managed policy CLAUDE.md / `claudeMd` settings key |
| Only matters when touching certain files | `.claude/rules/<topic>.md` with `paths:` |
| A multi-step procedure / loads on demand | A **skill** (Ch. 06) |
| Must hold no matter what Claude decides | A **hook** / `permissions.deny` (Ch. 09) |
| Something Claude figured out by working here | Let **auto memory** capture it, then audit |

**What actually defers token cost?**

| Mechanism | Defers context cost? | Notes |
|---|---|---|
| `@`-import | **No** -- loads in full at launch | Organization only |
| Ancestor / root CLAUDE.md | No -- loads in full at launch | Keep it under 200 lines |
| Subdirectory CLAUDE.md | **Yes** -- loads when Claude reads that dir | Monorepo pattern |
| `.claude/rules/` **with** `paths:` | **Yes** -- loads on glob match | The lean-memory lever |
| `.claude/rules/` **without** `paths:` | No -- launch, like `.claude/CLAUDE.md` | Unconditional rules |
| Skill | **Yes** -- loads on invoke / when relevant | See Ch. 06 |
| `MEMORY.md` topic file | **Yes** -- read on demand | Index lives in `MEMORY.md` |

---

## Sources

Official Anthropic:
- How Claude remembers your project (CLAUDE.md and auto memory) -- https://code.claude.com/docs/en/memory
- Subagents -- https://code.claude.com/docs/en/sub-agents
- Context window -- https://code.claude.com/docs/en/context-window
- Hooks reference -- https://code.claude.com/docs/en/hooks
- Monorepos and large repos -- https://code.claude.com/docs/en/large-codebases
- Changelog -- https://code.claude.com/docs/en/changelog.md

Open standard:
- AGENTS.md -- https://agents.md
