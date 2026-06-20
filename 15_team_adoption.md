# Chapter 15 -- Leading Team Adoption

**TL;DR.** Agentic engineering does not arrive by decree. It spreads the way every real practice spreads, hand to hand, through someone who uses it well in plain sight. This chapter is the individual-contributor change-agent playbook. You become the visible champion who shares techniques where the team already reads. You codify the team's shared judgment in a checked-in `CLAUDE.md`, path-scoped rules, and skills. You install review gates and deterministic hooks so quality and safety ride on the system instead of on anyone's memory. You win over skeptics one successful task at a time. And you measure adoption by signal that means something, which is artifacts accumulating and delivery moving, the count of seats provisioned being the least of it. Anthropic ships an official Champion kit doc and a `/team-onboarding` command that automate large parts of this. The framework travels across companies. Claude Code ships near-daily, though, so verify every version-pinned detail against the [changelog](https://code.claude.com/docs/en/changelog) and [docs home](https://code.claude.com/docs/en/overview) before you teach it. [^1]

> **Why an IC owns this.** The most reliable adoption pattern Anthropic documents is bottom-up: *"Adoption of a developer tool rarely happens because of a rollout announcement. It happens because someone on the team begins using the tool well, talks about it openly, and makes it easy for others to follow."* [^1] No title is required. The job belongs to whoever's pull requests make everyone else's day look easier.

---

## 15.1 The adoption thesis: practice spreads, mandates stall

The way this falls apart is almost never technical. The failure has a shape, and the shape is a quiet one. Leadership buys seats. A handful of engineers find the tool genuinely useful. The dashboard glows green because everyone is licensed. And the artifacts that would tell you the team actually adopted anything, a shared `CLAUDE.md`, checked-in skills, committed permission allowlists, hooks, simply never show up. Anthropic names the underlying brittleness directly: *"Adoption that depends on a single person is fragile. Adoption that is carried by shared habits continues to compound on its own."* [^1] A practice that lives in one enthusiast is a candle in the wind. A practice that lives in shared habits and shared files keeps burning after that person logs off, and it compounds while they sleep.

The work of a lead or change-agent is to convert leverage that belongs to one person into leverage that belongs to the team. That conversion happens in three places, and this chapter is built around them.

1. **Files in the repo** -- the team `CLAUDE.md`, `.claude/rules/`, `.claude/skills/`, `.claude/agents/`, `.claude/settings.json`, and hooks. These travel with `git clone`, so a new teammate inherits the whole team's accumulated practice on day one. [^2]
2. **Habits in the channel** -- where techniques get shared, questions get answered in public, and momentum survives any single person's attention. [^1]
3. **Gates in the pipeline** -- review and CI integration that make quality the default rather than a discipline.

The tool installs in two minutes. The behavior change is the whole job.

### The phased mental model

Most rollouts settle into the same three-stage rhythm. Hold it as a sequencing heuristic, not a schedule. The "done" markers below are the ones the official kit and the configuration docs actually name as the artifacts that signal real adoption.

| Stage | Goal | What "done" looks like |
|-------|------|------------------------|
| **1. Seed / Configure** | Establish the base layer | Shared `settings.json` + permissions committed; starter team `CLAUDE.md`; 2-3 high-frequency skills; first auto-format hook; a `#claude-code` channel exists |
| **2. Leverage / Habit** | Build compounding artifacts | Skill library growing; `CLAUDE.md` pruned and split into path-scoped rules; second champion identified; weekly show-and-tell running |
| **3. Govern / Durable** | Wire safety and measurement | Shared review subagent; managed PR review installed; permissions audited; analytics dashboard or OTel pipeline live; quarterly review on the calendar |

The enterprise-scale variant adds **gate criteria between phases**: hold the pilot small and refuse to expand into the wider department until the pilot shows sustained usage, gating on real usage rather than seats. Set your own number; there is no official threshold, and the principle is what matters.

---

## 15.2 The champion role (Anthropic's official kit)

Anthropic publishes a doc titled, plainly, **Champion kit**, with the subtitle "A playbook for engineers advocating Claude Code internally." It is the best-sourced artifact for this chapter, and worth reading start to finish with your team. [^1] Its core framing names you a multiplier: *"You are acting as a multiplier for your team, not a help desk."* [^1] Three reinforcing behaviors carry the role.

| Behavior | In practice | Why it works |
|----------|-------------|--------------|
| **Share what you discover** | Post prompts, screenshots, and small wins where the team already reads | *"Examples drawn from your own codebase are more persuasive than any external documentation"* |
| **Be the person people ask** | Answer questions with the actual prompt you used | *"A concrete, runnable example removes the gap between curiosity and a first successful use"* |
| **Grow the circle** | Establish a few lightweight recurring habits | *"Adoption that is carried by shared habits continues to compound on its own"* |
[^1]

### Keep it cheap (so it's sustainable)

The kit is blunt that this fits inside a normal week and stays there: about 15 minutes posting wins, about 20 minutes answering in a shared channel, about 5 minutes hosting a show-and-tell thread, and an optional 0 to 30 minutes pairing with whoever is genuinely blocked. [^1] Set that expectation with your lead up front, so *"the role should remain a multiplier on your existing work rather than an additional support responsibility."* [^1]

### Share techniques, not status updates

The post that earns its keep hands a colleague something they can run *tomorrow*: *"The most useful posts describe a technique a colleague can reuse tomorrow rather than an outcome that is already complete. Techniques compound as they spread through a team; status updates do not."* [^1] The shape that lands is a screenshot and one or two sentences. The long write-up gets a "save for later" tap and dies there; the short post with a screenshot gets copied and tried before lunch. Where to post:

- **`#claude-code` or general eng channel** -- discoveries, prompts, "today I learned."
- **PR descriptions** -- one line like *"Claude and I did this refactor; happy to walk through the approach."* This drops examples directly into the path reviewers already walk.
- **Standups / weekly written updates** -- one sentence, one concrete outcome. Normalizes usage with leads and skips.
- **Team wiki** -- durable patterns, custom skills, `CLAUDE.md` examples. [^1]

### Answer with a prompt, point at the feature

When someone asks how you pulled something off, paste the prompt you actually ran. *"They will learn more from running that prompt against their own problem than from any description you could write."* [^1] And in the moment, *"Try plan mode, press `Shift+Tab` until you see it"* gets them moving faster than a link to the docs ever will. The depth can come later, once they have the thing in their hands.

### The 30-day playbook

Anthropic's suggested sequence, with the explicit signal-that-it-is-working for each week. [^1]

- **Week 1 -- Seed the channel.** Create the channel, pin the Quickstart, post two or three of your own examples with prompts included. *Working signal:* a few colleagues react or reply, and at least one question is asked.
- **Week 2 -- Start the rhythm.** Begin the weekly show-and-tell thread ("What did Claude help you with this week?"), answer every question publicly, share one custom skill or `CLAUDE.md` snippet. *Working signal:* someone other than you posts an example.
- **Week 3 -- Pair and consolidate.** Offer two or three short pairing sessions, consolidate common Q&A into a pinned FAQ. *Working signal:* repeat usage, the same people returning rather than trying once and stopping.
- **Week 4 -- Hand off.** Identify a second champion (usually the person who asks you the most questions), share a what's-working/what's-not summary with your lead. *Working signal:* questions in the channel are being answered by people other than you.

That last signal is the one that counts. When the channel starts answering itself, the champion role has done its job.

---

## 15.3 Shared standards: the team `CLAUDE.md` and the rules layer

The team's accumulated judgment has to live somewhere version-controlled, or it lives only in the heads of whoever happened to learn it. This is the layer where agentic engineering stops being a collection of personal habits and becomes a team capability. The full memory hierarchy lives in Chapter 03; here the point narrows to making the shared layer carry the team.

### Check it into git -- that's the whole point

The official guidance leaves no room: *"Check CLAUDE.md into git so your team can contribute. The file compounds in value over time."* [^2] The locations and their sharing semantics:

- `./CLAUDE.md` or `./.claude/CLAUDE.md` (project root) -- **the shared team file.** Commit it. [^3]
- `./CLAUDE.local.md` -- personal project notes; **add to `.gitignore`** so it isn't shared.
- `~/.claude/CLAUDE.md` -- applies to all your sessions; personal, not team.
- Parent/child directory `CLAUDE.md` files -- monorepo-friendly; child files load on demand when Claude reads files in those directories. [^2]

For team adoption the rule is one sentence: **the project root file is your shared substrate; everything else is personal or governance.** Precedence runs Managed policy > User > Project > Local, all layers concatenate, and the more-specific layer is read last (see Ch. 03). [^3]

### What a team `CLAUDE.md` should contain

Anthropic's official include/exclude table is the canonical rubric. Teach it verbatim.

| Include | Exclude |
|---------|---------|
| Bash commands Claude can't guess | Anything Claude can figure out by reading code |
| Code style rules that differ from defaults | Standard language conventions Claude already knows |
| Testing instructions and preferred test runners | Detailed API documentation (link to docs instead) |
| Repository etiquette (branch naming, PR conventions) | Information that changes frequently |
| Architectural decisions specific to your project | Long explanations or tutorials |
| Developer environment quirks (required env vars) | File-by-file descriptions of the codebase |
| Common gotchas or non-obvious behaviors | Self-evident practices like "write clean code" |
[^2]

Run every line through one question: *"Would removing this cause Claude to make mistakes?"* If the answer is no, cut it. [^2] Here is the sentence that matters more than any other in this section, and it is the official wording: **Bloated `CLAUDE.md` files cause Claude to ignore your actual instructions.** [^2] The symptoms read like a diagnosis. *"If Claude keeps doing something you don't want despite having a rule against it, the file is probably too long and the rule is getting lost. If Claude asks you questions that are answered in CLAUDE.md, the phrasing might be ambiguous."* [^2] So **treat `CLAUDE.md` like code: review it when things go wrong, prune it regularly, and test changes by observing whether Claude's behavior actually shifts.** [^2]

On length, there is now an official number rather than a practitioner guess: **target under 200 lines per `CLAUDE.md` file.** *"Longer files consume more context and reduce adherence."* [^3] When the root file grows past what's broadly relevant, that growth is the signal to push detail down into the rules and skills layers below.

### The maintenance ritual that makes it compound

Here is the habit that defines a team, and the official guidance states it plainly. The Anthropic memory doc says to add to `CLAUDE.md` when *"Claude makes the same mistake a second time"* or when *"a code review catches something Claude should have known about this codebase."* [^3] Pair that with the best-practices instruction to *"check CLAUDE.md into git so your team can contribute"* [^2] and you have the whole loop: anytime the team catches Claude doing something incorrectly, that correction becomes a committed line everyone inherits, so Claude knows not to do it next time.

That distinction carries the whole weight for a team. **A conversational correction is a patch that helps you once; a written rule prevents the error in every future session, for everyone who clones the repo.** This is the machine that takes one engineer's hard-won lesson and turns it into something the whole team knows without ever having learned it the hard way. To operationalize it:

- Make updating the file a **deliberate habit**, not an afterthought. The memory doc frames `CLAUDE.md` as *"the place you write down what you'd otherwise re-explain,"* which is a thing you do continually as the team works, not once at setup. [^3]
- **Let the code-review flow surface gaps.** When the managed PR review (section 15.5) reads your `CLAUDE.md`, a newly introduced violation shows up as a nit, and if a change makes a `CLAUDE.md` statement outdated, the reviewer flags that the docs need updating too. [^4]

There is also **auto memory**, a second, Claude-written layer: notes Claude saves itself based on your corrections and preferences, stored per repository. It is machine-local and *not* shared across teammates, so it complements the committed `CLAUDE.md` rather than replacing it. For anything the team must share, write it into `CLAUDE.md`. [^3]

### The rules layer: `.claude/rules/` for path-scoped, just-in-time standards

The most common way team standards go wrong is the single flat `CLAUDE.md`: tech stack, conventions, testing rules, deploy procedures, and API patterns all piled into one wall of text that runs straight past the model's effective attention. The structural fix is to **split standards by relevance**, and the rules directory is a first-class, documented mechanism for it.

- **`CLAUDE.md`** -- only what applies *broadly, every session*. [^2]
- **`.claude/rules/*.md` with a `paths:` glob** -- modular rule files that load *only when Claude is working with files matching the specified patterns*. A rule scoped to `src/api/**/*.ts` enters context only when Claude touches the API layer. Rules without a `paths` field load unconditionally, with the same priority as `.claude/CLAUDE.md`. This is the primary lever for keeping team memory lean in a large or polyrepo codebase, and the directory supports symlinks so a company can maintain one shared rule set and link it into many repos. [^3]
- **Skills** -- domain knowledge or workflows that are only relevant *sometimes*. Official guidance: *"CLAUDE.md is loaded every session, so only include things that apply broadly. For domain knowledge or workflows that are only relevant sometimes, use skills instead. Claude loads them on demand without bloating every conversation."* [^2]

```markdown
# .claude/rules/api.md
---
paths:
  - "src/api/**/*.ts"
---
# API Development Rules
- All API endpoints must include input validation
- Use the standard error response format
- Include OpenAPI documentation comments
```
[^3]

### Standards that serve humans too

A team `CLAUDE.md` is onboarding documentation that happens to also get machine-read every session. The memory doc names a new teammate explicitly as one of the reasons to write something down: add it when *"a new teammate would need the same context to be productive."* [^3] The strongest standards earn their keep with both audiences at once, the model and the new human across the hall: make tacit knowledge explicit, write instructions concrete enough to verify ("Use 2-space indentation," not "format code properly"), and lean on enforcing tooling, linters and formatters and hooks, instead of trusting prose to hold the line on its own. [^3] And settle the rules *with* the team rather than for them. A say in the guidelines is how adoption quietly becomes ownership.

---

## 15.4 Shared configuration: settings, permissions, skills, agents

Standards prose is one half of what travels. The other half is the *setup* itself, and four kinds of committed artifacts make it portable.

### `.claude/settings.json` -- commit your guardrails

The big win at the project level is the **shared permission allowlist.** Without it, each engineer ends up reaching for `--dangerously-skip-permissions` in a moment of impatience. The documented alternative is to pre-allow the commands you already trust: the best-practices guide names *"permission allowlists: permit specific tools you know are safe, like `npm run lint` or `git commit`"* as one of the ways to cut interruptions while staying in control, built interactively with `/permissions`. [^2] Commit those rules to `.claude/settings.json` and the whole team inherits the same safe baseline instead of each person disabling prompts wholesale. What else is worth sharing: default model and effort, hook definitions, plugin/marketplace config, and output styles.

```jsonc
// .claude/settings.json (committed) -- illustrative
{
  "permissions": {
    "allow": [
      "Bash(npm run build:*)",
      "Bash(npm run lint)",
      "Bash(npm run test *)",
      "Bash(git commit:*)"
    ]
  }
}
```
[^5]

Build the allowlist interactively with `/permissions` rather than hand-editing, then commit. The permission-rule grammar is `ToolName(pattern)` with `*` and `**` globs, settable in `allow`, `ask`, and `deny` arrays. [^5] The grammar and key names are version-sensitive, so confirm against the current [settings](https://code.claude.com/docs/en/settings) and [permissions](https://code.claude.com/docs/en/permissions) docs before standardizing across the team.

### Shared skills and slash commands

Skills are plain Markdown, which means a teammate adopts one by copying a file: *"Because skills are plain Markdown, colleagues can adopt them immediately."* [^1] Commit your highest-frequency workflows to `.claude/skills/<name>/SKILL.md`. The canonical team starter the kit itself names is a `/ship` skill that *"runs tests and lint before committing."* [^1] A side-effecting workflow skill (the official `fix-issue` example) should set `disable-model-invocation: true` so it only fires when a human runs `/fix-issue 1234`, never on the model's own initiative. [^2] Sharing a skill is among the cheapest, highest-leverage moves a champion has: post the `SKILL.md` with a one-line description and the team is running it that day.

### Shared subagents -- especially for review

Define role-specialized assistants in `.claude/agents/` and commit them, so the whole team draws the same reviewer, the same read-only explorer, and so on. The official security-reviewer template is the archetype: a subagent with scoped tools (`Read, Grep, Glob, Bash`), a pinned model, and a focused system prompt that enumerates exactly what to hunt for, *"Injection vulnerabilities (SQL, XSS, command injection), Authentication and authorization flaws, Secrets or credentials in code, Insecure data handling,"* and ends with *"Provide specific line references and suggested fixes."* [^2] (See section 15.5 for how these wire into review gates, and Ch. 08 for subagent mechanics.)

### Org-level / managed settings (when you go enterprise)

Once a rollout outgrows a single team, administrators can push a `managed-settings.json` that sits at the top of the precedence chain. The official order is **Managed (highest, can't be overridden by anything) > command-line args > Local > Project > User (lowest).** [^5] The file lives at OS-specific managed-policy paths (macOS `/Library/Application Support/ClaudeCode/`, Linux/WSL `/etc/claude-code/`, Windows `C:\Program Files\ClaudeCode\`) and deploys via MDM, Group Policy, or similar. [^3]

This is where org-wide defaults and hard floors live: model defaults (`model`, `availableModels`, `enforceAvailableModels`), permission rules, mandatory or blocked MCP servers (`allowedMcpServers`, `deniedMcpServers`), hooks, and telemetry env vars. Several keys exist *only* in managed settings precisely so users cannot loosen them, including `allowManagedPermissionRulesOnly` (*"Prevent user and project settings from defining allow, ask, or deny permission rules. Only rules in managed settings apply"*) and `allowManagedHooksOnly`. [^5] Default the org model to a balanced daily-driver tier rather than the top tier, which holds cost down while leaving capability intact. Model ids change often, so confirm a current id against the [models page](https://platform.claude.com/docs/en/about-claude/models/overview) before pinning one org-wide.

---

## 15.5 Review gates: where verification becomes a team norm

As the agents write more of the code, the bottleneck slides. It used to sit at *writing*. Now it sits at *reviewing*, and a team feels that shift in its bones before it can name it. Anthropic ships managed Code Review precisely to answer it, dispatching *"a fleet of specialized agents"* that *"examine the code changes in the context of your full codebase, looking for logic errors, security vulnerabilities, broken edge cases, and subtle regressions."* [^4] The team-adoption answer is to make verification the default state of the world rather than a thing people remember to do. Deep verification mechanics (TDD with agents, the C-compiler case study, adversarial review) live in Ch. 12; this section is about wiring them into a *team's* defaults.

### The principle: give every change a check it can run

Anthropic's framing names the trap exactly: *"Claude stops when the work looks done. Without a check it can run, 'looks done' is the only signal available, and you become the verification loop."* Hand the agent something that returns pass or fail and the loop closes itself. [^2] At the team level, your `CLAUDE.md` and skills should *name* the checks, the test command, the build, the linter, the screenshot-vs-design, so every engineer's agent verifies the same way and nobody has to argue about what "done" means. The escalating gate ladder, straight from the docs:

1. **In one prompt** -- ask Claude to run the check and iterate in the same message.
2. **Across a session** -- set the check as a `/goal` condition; a separate evaluator re-checks after every turn until it holds.
3. **As a deterministic gate** -- a **Stop hook** runs your check as a script and blocks the turn from ending until it passes. *"Claude Code overrides the hook and ends the turn after 8 consecutive blocks."* [^2]
4. **By a second opinion** -- a verification subagent or workflow where a fresh model tries to refute the result, so the agent doing the work isn't the one grading it. [^2]

The official bottom line on why this is worth the setup is the whole reason that section of the docs exists: *"Give Claude a check it can run: tests, a build, a screenshot to compare. It's the difference between a session you watch and one you walk away from."* [^2]

### Adversarial / fresh-context review as a team gate

A reviewer in a **fresh subagent context** sees the diff and the criteria and nothing else. The docs spell out why that blindness is the feature: *"A fresh context improves code review since Claude won't be biased toward code it just wrote."* [^2] Two team-portable patterns:

- **Bundled `/code-review`** -- reviews the current diff in a fresh subagent and returns findings to the session. It reports correctness bugs plus reuse/simplification/efficiency cleanups; lower effort levels return fewer, higher-confidence findings while `high` through `max` give broader coverage; pass `--fix` to apply findings, `--comment` to post them as PR comments, or a target (file, PR number, branch, or `main...feature` range) to scope it. Standardize on it in your `CLAUDE.md` so review is one command for everyone. [^4]
- **Writer/Reviewer split** -- Session A implements, Session B reviews in a clean window, feedback flows back. The docs spell this out as a table workflow; it scales to an agent team for sustained loops. [^2]

> **The over-engineering trap (teach this to the whole team).** *"A reviewer prompted to find gaps will usually report some, even when the work is sound, because that is what it was asked to do. Chasing every finding leads to over-engineering: extra abstraction layers, defensive code, and tests for cases that can't happen. Tell the reviewer to flag only gaps that affect correctness or the stated requirements, and treat the rest as optional."* [^2] A team that obediently auto-fixes every nit a review agent coughs up will drown in defensive abstraction it never needed.

### Managed GitHub Code Review -- the team-wide gate

For a gate that lives in the pipeline itself, Anthropic's managed **Code Review** dispatches a fleet of specialized agents on each PR. They *"analyze the diff and surrounding code in parallel on Anthropic infrastructure. Each agent looks for a different class of issue, then a verification step checks candidates against actual code behavior to filter out false positives,"* and findings post as inline comments tagged by severity: red Important (a bug to fix before merge), yellow Nit, purple Pre-existing. [^4] The accurate operational facts for a team lead:

- **It's a research preview, Team and Enterprise only, and not available under Zero Data Retention.** [^4]
- **Install once** via the Claude GitHub App from admin settings; an admin picks which repos and sets each repo's trigger (once after PR creation, after every push, or manual). [^4]
- **Trigger or re-run on demand** with a top-level PR comment: `@claude review` (subscribes the PR to push-triggered reviews) or `@claude review once` (a single review, no subscription). [^4]
- **The check run never blocks merging** (always neutral conclusion); if you want to gate, parse the machine-readable severity counts out of the check-run output in your own CI. [^4]
- **Your standards enforce themselves.** Code Review reads your repo's `CLAUDE.md` files and flags newly introduced violations as nits, *bidirectionally* (it also flags when a change makes a `CLAUDE.md` statement stale). For review-only behavior, a root `REVIEW.md` is *"injected into the system prompt of every agent in the review pipeline as the highest-priority instruction block,"* which is where you recalibrate severity, cap nit volume, set skip rules, and add repo-specific checks like "new API routes must have an integration test." [^4]
- **Cost is real and worth saying out loud.** Each review averages $15-25, scaling with PR size, billed separately through usage credits; reviews complete in about 20 minutes on average; "after every push" multiplies cost by the number of pushes. Set a monthly spend cap in admin settings. [^4]

This is the cleanest way to make the team's standards enforce themselves on every PR while no human has to remember to run anything. The cost and the "every push" multiplier are exactly the levers a lead has to get wrong, so put the trigger choice and the spend cap on the rollout checklist.

---

## 15.6 Guardrails via hooks: deterministic team policy

`CLAUDE.md` rules are *advisory*. The model usually follows them, and "usually" is exactly the word that should make you nervous about anything safety-critical. The official line draws it cleanly: *"Unlike CLAUDE.md instructions which are advisory, hooks are deterministic and guarantee the action happens."* [^2] The heuristic is one sentence: *"Use hooks for actions that must happen every time with zero exceptions."* [^2] For a team, hooks are how you encode the non-negotiable policy that can no longer depend on every engineer, or every agent, remembering. Commit them in `.claude/settings.json`.

There's a quiet affordance worth pausing on: **Claude can write hooks for you.** *"Write a hook that runs eslint after every file edit"* or *"Write a hook that blocks writes to the migrations folder."* Browse what's configured with `/hooks`. [^2] That lowers the authoring bar far enough that hooks turn into a normal part of team setup instead of a specialist's chore.

### The team-relevant hook patterns

| Event | Pattern | Team value |
|-------|---------|-----------|
| `PostToolUse` on `Write\|Edit` | **Auto-format** (`prettier` / `eslint --fix`), often `async: true` | The documented pattern: PostToolUse hooks "commonly auto-format code after file edits using tools like prettier or eslint," so formatting lands automatically instead of failing CI later [^6] |
| `PreToolUse` on `Bash` | **Deny dangerous commands** (block writes to `migrations/`, `rm -rf`, force-push) by returning `permissionDecision: "deny"` | A guardrail the model can't talk its way around; *"Before a tool call executes. Can block it."* [^6] |
| `Stop` | **Block the turn until a check passes** / notify on completion | Makes long runs self-terminating; the official champion example is a Stop hook for a desktop notification when a long task finishes. [^1] |
| `SessionStart` | **Inject context** (error logs, recent PRs, ticket/STATUS state) | Every session starts with current context without baking it into every prompt [^6] |
| `PostToolUse` (`type: http`) matching all tools | **Audit trail** -> POST every tool call to a central endpoint | Compliance evidence; directly relevant to regulated-industry teams [^6] |

The audit/policy pattern is the spine of enterprise governance, and it is a first-class documented feature, not a workaround. An `http` hook *"send[s] the event's JSON input as an HTTP POST request to a URL,"* and *"Non-2xx responses, connection failures, and timeouts all produce non-blocking errors that allow execution to continue"*, so an audit endpoint that goes down degrades gracefully instead of wedging itself between your developers and their work. A `PreToolUse` hook denies disallowed commands before they ever run, and `allowManagedHooksOnly` in managed settings lets an org force these hooks on and block all others. [^6]

> **Version caution.** The hook event roster is large and growing (`SessionStart`, `UserPromptSubmit`, `PreToolUse`, `PostToolUse`, `Stop`, `SubagentStop`, `FileChanged`, `PreCompact`, `SessionEnd`, and many more), the exit-code/JSON control protocol evolves, and the Stop-hook override count is version-pinned. Verify against the [hooks docs](https://code.claude.com/docs/en/hooks) before standardizing a team hook library. [^6]

**Rule of thumb for the team:** reach for deterministic command/http hooks on the safety-critical rules; lean on `CLAUDE.md` and rules for judgment and style; and save prompt/agent hooks for the judgment calls that genuinely need a model behind them.

---

## 15.7 Onboarding skeptics (and new hires)

A senior engineer who eyes this tool with suspicion is behaving correctly. The kit agrees: *"Healthy skepticism is expected; engineers should be cautious about tools that touch their code."* [^1] The official move is to leave the general argument alone and instead *"acknowledge the concern, offer a brief reframe, and propose one concrete demonstration on the person's own code. Most concerns are resolved by a single successful experience."* [^1]

### The objection-handling table (memorize this)

| Concern | Reframe | Evidence to offer |
|---------|---------|-------------------|
| "I'm faster without it." | True for code they write routinely; point them at the work they *avoid*: legacy files, unfamiliar services, test scaffolding, where leverage is highest. | Time one tedious task both ways. |
| "I don't trust AI to touch production code." | Agree nothing lands unread. Plan mode + normal diff review means nothing applies that they haven't inspected, the same standard as any PR. | Demonstrate plan mode on a real file. |
| "It'll make junior engineers weaker." | Used well it's an effective explainer; have juniors ask it to explain a file and its call sites before changing anything. | Run "Explain @file and where it is called from" together. |
| "I tried it once and it hallucinated." | Usually a context problem, not a model problem: `@`-mention the files, run `/init`, paste the actual error. | Re-run their original prompt with proper `@`-context. |
| "We don't have time to learn another tool." | It's a terminal command, not a platform; if it doesn't return value in the first session, fair to set it aside. | A two-minute install + one real bug. |
| "Is this just autocomplete?" | Offer a task requiring reasoning across the repo, trace a bug across services or draft a migration plan. | A two-minute live demonstration. |
[^1]

### Plan mode is the trust on-ramp

The single most effective thing you can put in front of a skeptic is **plan mode.** The kit's own answer to "How do I trust it with my code?" is to introduce it: *"pressing `Shift+Tab` cycles into it, Claude proposes exactly what it intends to change, and nothing is modified until the user approves."* [^1] For the senior who says they don't trust AI with their code, this is the demo that lands, because it slots cleanly into the inspect-before-merge contract a pull request already gives them. Pair it with the official **explore -> plan -> implement -> commit** workflow and what they watch is discipline, methodical and reviewable, instead of autopilot. [^2]

### Pick the right first task

For "what should I try it on first?" the kit is specific: *"Recommend a real but contained task, ideally a bug or chore the person has been postponing because it is tedious rather than difficult."* [^1] One good outcome on their *own* code outpersuades any presentation, and a single fifteen-minute pairing session does more than a slide deck. One boundary to hold cleanly: questions about *security and data handling* go to your administrator, not improvised in the moment, because *"Your organization's deployment and data-handling policy is already configured, and champions should not improvise this answer."* [^1]

### `/team-onboarding` -- auto-generated ramp-up guides

Claude Code ships a `/team-onboarding` command that, per the official Commands reference, *"analyzes your sessions, commands, and MCP server usage from the past 30 days and produces a markdown guide a teammate can paste as a first message to get set up quickly. For claude.ai subscribers on Pro, Max, Team, and Enterprise plans, [it] also returns a share link teammates can open directly in Claude Code."* [^7] The champion move the kit names: run it *"in a project you have spent real time in"* and pin the output in the channel. [^1]

### Onboarding new engineers *through* the tool

There's a second, deeper onboarding pattern hiding in plain sight: hand the codebase to the new hire and let them ramp by interrogating it through Claude Code itself. *"You can ask Claude the same sorts of questions you would ask another engineer"*, how does logging work, how do I make a new API endpoint, what edge cases does `CustomerOnboardingFlowImpl` handle, why does this code call `foo()` instead of `bar()` on line 333. The docs call it *"an effective onboarding workflow, improving ramp-up time and reducing load on other engineers."* [^2] Anthropic runs this internally: *"New data scientists on our Infrastructure team feed Claude Code their entire codebase to get productive quickly. Claude reads the codebase's CLAUDE.md files, identifies relevant ones, explains data pipeline dependencies, and shows which upstream sources feed into dashboards."* [^8] The loop feeds itself. A good team `CLAUDE.md` makes the new hire ramp faster, and the new hire's questions surface every gap the `CLAUDE.md` has been hiding.

---

## 15.8 Measuring adoption: signal over vanity

The measurement trap is sitting right there, and it is comfortable. Count **seats provisioned** or **lines accepted**, watch the number climb, declare victory, move on. The thing those vanity metrics paper over is whether the practice is actually compounding. Anthropic's own analytics doc says the quiet part: even its contribution metrics are *"deliberately conservative and represent an underestimate of Claude Code's actual impact."* [^9] So read the dashboard for direction and trend, not as a scoreboard.

### The native analytics dashboard

For Teams and Enterprise plans, Anthropic provides a built-in dashboard at `claude.ai/analytics/claude-code` (Admins and Owners can view). [^9] It surfaces, with exact metric names:

- **Usage:** *lines of code accepted*, *suggestion accept rate*, *daily active users*, *sessions*.
- **Contribution (public beta; requires the GitHub app installed by a GitHub admin and the feature enabled by a Claude Owner):** *PRs with CC*, *Lines of code with CC*, *PRs with Claude Code (%)*, a *PRs per user* chart, and a **Leaderboard** of the top 10 contributors with full CSV export of all users. [^9]

Two caveats decide whether the numbers get read honestly or read flatteringly:

1. **The contribution metrics are conservative by design.** Only lines and PRs with *high confidence* of Claude Code involvement are counted; code a developer substantially rewrote (more than 20% difference) is *not* attributed; lock files, generated code, build directories, and test fixtures are excluded; attribution uses a window from 21 days before to 2 days after the PR merge date. [^9]
2. **Contribution metrics are unavailable under Zero Data Retention**, and not available to API/Console customers (Console shows usage + spend only). [^9]

API/Console customers get a separate dashboard at `platform.claude.com/claude-code` (needs the `UsageView` permission) with *lines accepted*, *suggestion accept rate*, *activity*, *spend*, and a per-user team-insights table (members, spend this month, lines this month). [^9]

### OpenTelemetry for deep observability

For per-user token and cost detail and the behavioral signals underneath, export OTel: set `CLAUDE_CODE_ENABLE_TELEMETRY=1` alongside `OTEL_METRICS_EXPORTER` / `OTEL_LOGS_EXPORTER` and an OTLP endpoint, distributable org-wide via managed settings/MDM where *"environment variables defined in the managed settings file have high precedence and cannot be overridden by users."* [^10] It emits metrics (`claude_code.session.count`, `claude_code.lines_of_code.count`, `claude_code.pull_request.count`, `claude_code.commit.count`, `claude_code.cost.usage`, `claude_code.token.usage`, `claude_code.code_edit_tool.decision`, `claude_code.active_time.total`) and events into your stack, with cardinality controls (`OTEL_METRICS_INCLUDE_SESSION_ID`, `OTEL_METRICS_INCLUDE_ACCOUNT_UUID`) and a beta distributed-tracing mode (`CLAUDE_CODE_ENHANCED_TELEMETRY_BETA=1`) that links each prompt to the API calls and tool executions it triggers. [^10] The `claude_code.code_edit_tool.decision` counter, with its `accept`/`reject` dimension, is the behavioral signal worth watching: a rising reject rate points at a trust or config problem, not at productivity. **Privacy note for the team:** user-prompt and tool content are *off by default* and gated behind explicit `OTEL_LOG_USER_PROMPTS`, `OTEL_LOG_TOOL_DETAILS`, and `OTEL_LOG_TOOL_CONTENT` flags. Leave them off unless you have a compliance reason and consent, because flipping them on exposes conversation content to whoever holds the dashboard. [^10]

### A balanced scorecard

Combine platform metrics with artifact and outcome signals. A workable team scorecard:

| Dimension | Metric | What it tells you |
|-----------|--------|-------------------|
| **Adoption** | Weekly active developers (not seats); session trend | Real uptake; dips flag friction. Native: Adoption chart. [^9] |
| **Artifact maturity** | Skill/agent count, committed hooks, `CLAUDE.md` health | The truest signal of *team* (vs individual) adoption; these accumulate only when the team really adopts |
| **Outcome** | PRs per user over time; PR cycle time; ship-with-vs-without comparison | Ties tool use to delivery; the docs suggest pairing with DORA metrics and sprint velocity. [^9] |
| **Quality / trust** | Suggestion accept rate; tool-rejection rate (`code_edit_tool.decision`) | High rejection = trust or config problem, not productivity. [^10] |
| **Cost** | Spend per *active* developer | Normalizes cost to usage. [^9] |
| **Ramp** | New-hire time-to-first-meaningful-output | The compounding-value payoff of good shared standards. |

Anthropic's own guidance for the leaderboard is to recruit, not rank: use it to *"find team members with high Claude Code adoption who can share prompting techniques and workflows with the team, provide feedback on what's working well, [and] help onboard new users."* [^9]

> **Measurement ethics for the team lead.** Per-user dashboards and leaderboards read as surveillance to the people on them, and surveillance chills the exact experimentation you spent all this effort trying to start. Be transparent about what gets collected. Keep prompt and tool content logging off by default. Frame the leaderboard as "who can help others." And say plainly that "lines of code" is, in Anthropic's own words, a deliberate underestimate and a weak proxy for value. The metric that matters most, *do the artifacts accumulate?*, never shows up on a vendor dashboard at all.

---

## 15.9 Common failure modes (and the fix)

Synthesized from the official failure-pattern list and the configuration docs.

| Failure | Symptom | Fix |
|---------|---------|-----|
| **No named lead** | Implicit ownership collapses within weeks | Name a champion; hand off to a second by week 4 [^1] |
| **Paper adoption** | Seats provisioned, no artifacts accumulate | Measure artifacts + PR cycle time, not seats |
| **`CLAUDE.md` bloat** | Claude ignores rules as the file grows | Prune ruthlessly; push detail to rules/skills; convert a satisfied rule to a hook [^2] |
| **Over-restrictive governance** | Permission friction kills usage | Sensible allowlist baseline built with `/permissions`; audit-driven, not lock-everything [^2] |
| **Review-agent over-engineering** | Team chases every nit a reviewer emits | Scope reviewers to correctness/requirements; treat the rest as optional [^2] |
| **Runaway review spend** | "After every push" multiplied across busy repos | Choose triggers per repo; set a monthly spend cap [^4] |
| **Champion as help desk** | One person answers everything; burnout | Answer in public once, link thereafter; grow a second champion [^1] |

The general-purpose anti-patterns from the official guide deserve a spot on a shared one-pager too: *"the kitchen sink session"* (fix: `/clear` between unrelated tasks), *"correcting over and over"* (fix: after two failed corrections, `/clear` and write a better prompt), *"the over-specified CLAUDE.md,"* *"the trust-then-verify gap"* (fix: always provide verification; if you can't verify it, don't ship it), and *"the infinite exploration"* (fix: scope investigations or push them to subagents). They are the everyday habits that quietly decide whether individuals stay productive enough to keep the team's momentum alive. [^2]

---

## 15.10 A portable rollout checklist

A condensed, company-agnostic sequence you can hand to any team lead.

**Stage 1 -- Seed / Configure**
- [ ] You become the visible champion: post 2-3 techniques with prompts; create `#claude-code`; pin Quickstart.
- [ ] Commit a starter team `CLAUDE.md` (run `/init`, then prune to the include/exclude rubric, target under 200 lines).
- [ ] Commit `.claude/settings.json` with a shared permission allowlist (built via `/permissions`).
- [ ] Add 1-2 high-frequency skills (a `/ship` skill, a test runner) and the auto-format `PostToolUse` hook.
- [ ] Start the weekly show-and-tell thread.

**Stage 2 -- Leverage / Habit**
- [ ] Split `CLAUDE.md` -> `.claude/rules/` (path-scoped) + skills as it grows.
- [ ] Commit a shared review subagent (`.claude/agents/`); standardize on `/code-review` in `CLAUDE.md`.
- [ ] Add `PreToolUse` deny-rules for destructive commands.
- [ ] Make the "correction -> write a rule" ritual explicit.
- [ ] Identify and onboard a second champion.

**Stage 3 -- Govern / Durable**
- [ ] Turn on the analytics dashboard (or OTel pipeline) and stand up a balanced scorecard.
- [ ] Audit permissions/secrets handling; add a `type: http` audit-trail hook if regulated.
- [ ] Install managed Code Review as the team-wide PR gate; choose triggers per repo and set a spend cap.
- [ ] (Enterprise) push managed settings via MDM; lock floors with `allowManaged*Only` keys.
- [ ] Calendar a quarterly review of standards, skills, and metrics.

---

## Sources

- https://code.claude.com/docs/en/champion-kit
- https://code.claude.com/docs/en/best-practices
- https://code.claude.com/docs/en/memory
- https://code.claude.com/docs/en/code-review
- https://code.claude.com/docs/en/analytics
- https://code.claude.com/docs/en/monitoring-usage
- https://code.claude.com/docs/en/settings
- https://code.claude.com/docs/en/permissions
- https://code.claude.com/docs/en/hooks
- https://code.claude.com/docs/en/commands
- https://code.claude.com/docs/en/changelog
- https://claude.com/blog/how-anthropic-teams-use-claude-code

[^1]: web | 2026-06-18 | [code.claude.com/docs/en/champion-kit](https://code.claude.com/docs/en/champion-kit)
[^2]: web | 2026-06-18 | [code.claude.com/docs/en/best-practices](https://code.claude.com/docs/en/best-practices)
[^3]: web | 2026-06-18 | [code.claude.com/docs/en/memory](https://code.claude.com/docs/en/memory)
[^4]: web | 2026-06-18 | [code.claude.com/docs/en/code-review](https://code.claude.com/docs/en/code-review)
[^5]: web | 2026-06-18 | [code.claude.com/docs/en/settings](https://code.claude.com/docs/en/settings)
[^6]: web | 2026-06-18 | [code.claude.com/docs/en/hooks](https://code.claude.com/docs/en/hooks)
[^7]: web | 2026-06-18 | [code.claude.com/docs/en/commands](https://code.claude.com/docs/en/commands)
[^8]: web | 2026-06-18 | [claude.com/blog/how-anthropic-teams-use-claude-code](https://claude.com/blog/how-anthropic-teams-use-claude-code)
[^9]: web | 2026-06-18 | [code.claude.com/docs/en/analytics](https://code.claude.com/docs/en/analytics)
[^10]: web | 2026-06-18 | [code.claude.com/docs/en/monitoring-usage](https://code.claude.com/docs/en/monitoring-usage)
