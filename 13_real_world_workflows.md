# How Real Engineers Actually Work

**TL;DR.** The earlier chapters handed you the machinery. This one is about what people do with it once they live in it all day, and the public record is unusually consistent about the answer. Three things separate the engineers who get compounding value from the ones who get a faster autocomplete. They give the model a way to check its own work, so the loop closes without them watching. They write every recurring correction down as a rule instead of re-prompting, so the fix outlives the session. And they run several agents at once, which turns the human from typist into reviewer and director. None of the three is a feature you toggle. They are habits, and the rest of this chapter is the detail behind them, sourced to the people and the docs that actually said it.

> **A note on what ships and what drifts.** Claude Code ships near-daily, so every version-pinned command in this chapter (`/goal`, `/loop`, auto mode, agent view) carries a docs reference you can re-check, and several carry a minimum-version note straight from the docs (`/goal` needs v2.1.139, `/loop` v2.1.72, auto mode v2.1.83). The model names will age the fastest of all; where a practitioner named a specific model, that name is preserved as a timestamp on their quote, not as current advice. Confirm anything version-sensitive against the changelog (`https://code.claude.com/docs/en/changelog.md`) or `/release-notes` before you teach it. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/changelog.md]

---

## The shape of a real day

Strip the tooling away and the day of a heavy Claude Code user has a shape you can recognize, and it is not "open editor, type code." It is closer to running a small team and reading their output. The same loop turns up across the public writeups.

You start by skimming what finished while you were gone. Then you kick off the day's independent threads, each in its own checkout or worktree, because running several at once is how the tool is built to scale. For each thread you give a crisp brief, let the model explore and plan when the change is non-trivial, let it implement, make it verify against a real check, review the diff, and ship. You cycle between threads, answering whichever one is asking for input, and notifications tell you which one that is. And every time an agent does something wrong, you write the correction into a rule rather than fixing it only in chat.

That is the whole loop. The sections below are the parts of it that reward getting right.

---

## The setup the docs build first

The official best-practices guide reads like a worked example of how the people closest to the tool actually run it, because it was written from how Anthropic's own teams use it. The order it presents things in is itself the lesson: verification first, then a written rule file, then plan-before-code, then parallelism. The sections below take them in that order, because that is the order in which they compound.

### Verification is the most important thing

The guide does not bury this. It is the first practice after the context-window framing, and it is stated as a tip you can read in one breath: "Give Claude a check it can run: tests, a build, a screenshot to compare. It's the difference between a session you watch and one you walk away from." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/best-practices] The reasoning underneath it is the part worth internalizing: "Claude stops when the work looks done. Without a check it can run, 'looks done' is the only signal available, and you become the verification loop: every mistake waits for you to notice it." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/best-practices]

The check is anything that returns a signal the model can read: a test suite, a build exit code, a linter, a script that diffs output against a fixture, a browser screenshot compared against a design. Once the check exists, you decide how hard it gates the stop. In a single prompt you ask Claude to run it and iterate in the same message. Across a session you set it as a `/goal` condition, where a separate evaluator re-checks it after every turn and Claude keeps working until it holds. As a deterministic gate you put it in a `Stop` hook that blocks the turn from ending until the check passes. The full treatment of verification and adversarial review lives in Chapter 11; what belongs here is the practice: it is the first thing the docs put in front of you, and it is the difference between watching and walking away. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/best-practices]

### Write it down instead of re-prompting

This is the habit that compounds. `CLAUDE.md` is the file Claude reads at the start of every conversation, it is meant to be checked into git so a team can contribute to it, and the docs are explicit that "the file compounds in value over time." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/best-practices] The standing move is to fold every recurring correction into it rather than re-prompting: the docs note that "if Claude keeps doing something you don't want despite having a rule against it, the file is probably too long and the rule is getting lost," and the fix is to treat the file like code, reviewing it when things go wrong and pruning it regularly. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/best-practices]

The mechanism is where the leverage hides. A correction in chat fixes one run and leaves the failed attempt sitting in the current context. A written rule fixes every future run and costs nothing in the current window. That asymmetry is the whole game. The memory hierarchy that makes this work, and the discipline that keeps the file from bloating until Claude ignores it, are Chapter 03's territory; the practice is the part that belongs here, and the docs name the litmus: for every line, ask "Would removing this cause Claude to make mistakes?" and if not, cut it, because "bloated CLAUDE.md files cause Claude to ignore your actual instructions." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/best-practices]

### Plan mode, then auto-accept

There is a quiet correction to make here, because the secondhand version of this workflow has it backwards. The posture to copy is plan first, then let Claude code. The docs lay it out as a four-phase workflow: explore in plan mode where Claude "reads files and answers questions without making changes," then "ask Claude to create a detailed implementation plan," then "switch out of plan mode and let Claude code, verifying against its plan," then commit. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/best-practices] The plan-then-accept loop has a name in the permission system too: when a plan is ready you can "approve and start in auto mode" or "approve and accept edits" directly from the plan prompt. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/permission-modes]

The docs are explicit that planning has a cost and a place. For a typo or a renamed variable you skip it; "if you could describe the diff in one sentence, skip the plan." Planning earns its keep "when you're uncertain about the approach, when the change modifies multiple files, or when you're unfamiliar with the code being modified." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/best-practices] Editing the plan is a senior move worth its own muscle memory: press `Ctrl+G` to open the proposed plan in your text editor and change it directly before Claude proceeds, rather than arguing with it in chat. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/permission-modes]

### Parallelism, and the trust that makes it work

Running several sessions at once is the practice the docs frame as the way Claude Code "scales horizontally." The recommended approach for isolated parallel work is worktrees: "run separate CLI sessions in isolated git checkouts so edits don't collide," with the desktop app taking it a step further and creating "a worktree for every new session automatically." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/best-practices] ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/worktrees] The point of the isolation is concrete: edits in one session never touch files in another, "so you can have Claude building a feature in one terminal while fixing a bug in a second." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/worktrees]

What makes several concurrent agents supervisable by one person is auto mode. A separate classifier model reviews each action before it runs and blocks anything that "escalates beyond your request, targets unrecognized infrastructure, or appears driven by hostile content Claude read," so routine work proceeds without a prompt and you are only pulled in for the genuinely risky calls. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/permission-modes] When the approvals stop interrupting, one human can hold several threads. The mechanics of the parallel session itself are below.

### The smaller habits worth stealing

A handful of documented habits are durable and worth lifting wholesale.

Keep a reusable workflow for every inner-loop task you repeat. The docs show this as a skill with a `disable-model-invocation: true` workflow you trigger by hand, the canonical example being a fix-issue routine that views the issue, implements the fix, writes and runs tests, and opens a PR. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/best-practices] Reduce permission friction the safe way, with auto mode or `/permissions` allowlists rather than reaching for the dangerous-skip flag: "after the tenth approval you're not really reviewing anymore, you're just clicking through." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/best-practices] And give Claude real CLI surface area, because a CLI is "the most context-efficient way to interact with external services," so install `gh`, `aws`, `gcloud`, `sentry-cli` and tell Claude to use them. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/best-practices]

Two habits worth building are now simply native features, so reach for the built-in. Dictation is one: hold or tap `Space` to dictate a prompt, which is faster than typing and tends to produce more detailed instructions as a side effect, and the docs are blunt that "the more precise your instructions, the fewer corrections you'll need." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/interactive-mode] ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/best-practices] A custom status line is the other: track context usage continuously so context pressure is always in view, which the best-practices doc recommends directly. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/best-practices]

---

## How Anthropic's own teams use it

Anthropic published an internal-usage roundup that is one of the better windows into real workflows precisely because it reaches past engineering into design, data science, marketing, and legal. The throughline is that the strongest teams treat Claude Code as a first stop and a thought partner. ^[source: web | 2026-06-18 | https://claude.com/blog/how-anthropic-teams-use-claude-code]

### Engineering

Product engineers describe Claude Code as their first stop for any programming task, asking it which files to examine before they build, which removes the manual context-gathering that used to precede every change. The same posture gives them the confidence to fix bugs in codebases they don't own and review the proposed fix instead of escalating to the owning team. ^[source: web | 2026-06-18 | https://claude.com/blog/how-anthropic-teams-use-claude-code]

The security team's pivot is the cleanest case study in the post. Their old loop was "design doc, janky code, refactor, give up on tests." Now they ask for pseudocode, guide Claude through test-driven development, and check in periodically, which yields more reliable, testable code. The field-tested TDD-with-agents loop is Chapter 11's home; what this adds is the before-and-after of a real team that changed how it works. ^[source: web | 2026-06-18 | https://claude.com/blog/how-anthropic-teams-use-claude-code]

Two debugging stories are worth keeping. Feeding stack traces and docs to trace control flow turned problems that used to take "10-15 minutes of manual scanning" into ones resolved roughly three times as fast. And during a live production incident, the data infrastructure team fed Claude dashboard screenshots and it walked them through Google Cloud's UI until they found pod IP-address exhaustion, then handed over the exact commands to create a new IP pool and add it to the cluster. ^[source: web | 2026-06-18 | https://claude.com/blog/how-anthropic-teams-use-claude-code] The lesson is to paste the screenshot. An image is first-class context that the model reasons over directly, and the same input drives the design-to-code work below.

The rest of the engineering pattern rhymes. The inference team writes logic in the codebase's native language by explaining what to test and letting Claude write the Rust they don't know well. New data scientists feed entire codebases, `CLAUDE.md` files included, to map pipeline dependencies and trace what feeds a dashboard, replacing what a data-catalog tool used to do. ^[source: web | 2026-06-18 | https://claude.com/blog/how-anthropic-teams-use-claude-code]

### Outside engineering

Designers feed Figma files in and set up autonomous loops where Claude writes the feature, runs tests, and iterates, reviewing before refinement; one had Claude build itself Vim key bindings with minimal human review. Data scientists with no TypeScript fluency build whole React apps to visualize model performance with one-shot prompting and small tweaks. Growth marketing processes CSVs of hundreds of ads to flag underperformers and generate compliant variations, with a Figma plugin that swaps copy to produce up to a hundred variations. Legal built a prototype phone tree to route people to the right lawyer, a non-technical team shipping a custom tool with no dev resources. ^[source: web | 2026-06-18 | https://claude.com/blog/how-anthropic-teams-use-claude-code]

Every one of these reduces to the same shape the post names directly: "they give Claude abstract problems, let it work autonomously, then review solutions before final refinements," and "the most successful teams treat Claude Code as a thought partner rather than a code generator." ^[source: web | 2026-06-18 | https://claude.com/blog/how-anthropic-teams-use-claude-code] The senior reading is that autonomy is earned per task-type. You hand off more as your verification harness for that kind of work gets stronger, never as a blanket setting.

---

## What independent practitioners converge on

A workflow that shows up in the official guidance, in Anthropic's internal roundup, and in unaffiliated engineers' writeups is the one most likely to generalize. Two independent senior accounts are worth reading in full, and both are cited from the authors' own writing.

### A disciplined daily loop

Arpan Patel's writeup is a representative senior practice whose structure mirrors the official advice almost point for point. His feature loop is explore, then plan, then code: "Hit `Shift+Tab` twice to drop into plan mode," edit the plan with `Ctrl+G` until it matches your head, implement, then either invoke a `/pr-review` subagent or spawn a fresh session for an independent critique. ^[source: web | 2026-06-18 | https://arps18.github.io/posts/claude-code-mastery/] His bug loop is stricter: "Reproduce it before you touch anything," pipe the error in with `cat error.log | claude`, and ask for a failing test that demonstrates the bug before any fix. The rule under it is the one to memorize: "Never let Claude claim success without evidence, whether that's tests, screenshots, or real command output." ^[source: web | 2026-06-18 | https://arps18.github.io/posts/claude-code-mastery/]

His memory discipline is the same litmus the docs use, arrived at independently. For every line in `CLAUDE.md` he asks "Would removing this cause Claude to make a mistake? If the answer is no, cut it," and when Claude errs he tells it "Update CLAUDE.md so you don't repeat this." ^[source: web | 2026-06-18 | https://arps18.github.io/posts/claude-code-mastery/] He keeps a gitignored `CLAUDE.local.md` for personal rules that never leave his machine. His `/pr-review` subagent runs `git diff main...HEAD`, reads full files rather than diff hunks, groups findings Critical through Low, and deliberately does not flag style preferences that aren't in the project rules. ^[source: web | 2026-06-18 | https://arps18.github.io/posts/claude-code-mastery/] He sets `CLAUDE_CODE_AUTO_COMPACT_WINDOW=400000` to force earlier compaction because, in his words, "context rot kicks in around 300-400k tokens on the 1M model." ^[source: web | 2026-06-18 | https://arps18.github.io/posts/claude-code-mastery/] And he resists installing every MCP, keeping a lean starter set of GitHub, Context7, and one or two domain-specific servers, because "bloated tool lists hurt decision quality." ^[source: web | 2026-06-18 | https://arps18.github.io/posts/claude-code-mastery/]

His best single line is the plan-as-design-review: "Have one Claude write the plan. Spin up a second Claude in a fresh session and ask it to review the plan as a staff engineer, no context bias." ^[source: web | 2026-06-18 | https://arps18.github.io/posts/claude-code-mastery/] That pattern is below.

### The parallel-agent lifestyle

Simon Willison documents the multi-agent extreme, and his account is valuable because he runs Claude Code alongside other agents, which makes the practice portable rather than tool-specific. (His writeup is dated October 2025 and names the then-current model; the workflow generalizes, the model name does not.) "I frequently have multiple terminal windows open running different coding agents in different directories." ^[source: web | 2026-06-18 | https://simonwillison.net/2025/Oct/5/parallel-coding-agents/]

The part to steal is his taxonomy of what to hand a parallel agent. Research and proof-of-concept tasks that "answer questions or provide recommendations without making modifications to a project that you plan to keep." System understanding, where a reasoning model traces how a portion of your existing system works in a minute or two. Low-stakes maintenance, "problems that really just require a little bit of extra cognitive overhead which can be outsourced to a bot." And specified implementation, because "code that started from your own specification is a lot less effort to review." ^[source: web | 2026-06-18 | https://simonwillison.net/2025/Oct/5/parallel-coding-agents/]

His safety posture is the transferable insight. He runs no-approvals mode "for tasks where I'm confident malicious instructions can't sneak into the context." Riskier work goes to asynchronous cloud agents, where "if anything goes wrong the worst that can happen is my source code getting leaked." And he notes he should "start habitually running my local agents in Docker containers to further limit the blast radius." For isolation against the same repo he does "a fresh checkout, often into /tmp." ^[source: web | 2026-06-18 | https://simonwillison.net/2025/Oct/5/parallel-coding-agents/] Match the autonomy to the blast radius: high-trust, no-approvals work is reserved for tasks where the radius is bounded and untrusted content cannot reach the context, and everything else gets sandboxing or review.

---

## Parallelism in practice

Parallelism is the single most-cited unlock, so the mechanics deserve their own treatment. Three approaches, in increasing order of how much coordination the tool does for you. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/best-practices]

| Approach | Isolation | Coordination | Best for |
|---|---|---|---|
| Separate git checkouts | A full second clone | You, manually | Maximum isolation; no shared history to reason about ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/best-practices] |
| Git worktrees (`claude --worktree <name>`) | Separate working dir + branch, shared history | You, manually | Isolated parallel sessions; native CLI and automatic desktop support ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/worktrees] |
| Agent teams | Each session in its own context | Automated (shared tasks, messaging, a lead) | Sustained parallelism, large migrations ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/best-practices] |

### The worktree workflow

A worktree is a separate working directory with its own files and branch that shares the repository history, so edits in one session never touch files in another. Pass `--worktree` (or `-w`) with a name to create one and start Claude in it, then run the same command with a different name in a second terminal for an isolated parallel session. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/worktrees]

```bash
# terminal 1
claude --worktree feature-auth
# terminal 2 -- different name, isolated parallel session, same repo
claude --worktree fix-rate-limit
```

The desktop app creates a worktree for every new session automatically, which is why the team prefers worktrees over manual checkouts. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/worktrees] To watch many sessions from one screen instead of separate terminals, open agent view with `claude agents`. It is "one screen for all your background sessions," grouping them under Needs input, Working, and Completed so you step in only when one needs you. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/agent-view]

### The Writer/Reviewer pattern: parallelism for quality

The most valuable second session has nothing to do with throughput. It is a fresh context for review, because a clean session "won't be biased toward code it just wrote." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/best-practices] The docs give the exact shape:

- **Session A (Writer):** "Implement a rate limiter for our API endpoints."
- **Session B (Reviewer):** "Review the rate limiter implementation in @src/middleware/rateLimiter.ts. Look for edge cases, race conditions, and consistency with our existing middleware patterns."
- **Session A:** "Here's the review feedback: [Session B output]. Address these issues."

The same shape works for tests, where one Claude writes them and another writes the code to pass them, and for plans, where one drafts and a fresh staff-engineer session critiques, which is Patel's habit above. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/best-practices] ^[source: web | 2026-06-18 | https://arps18.github.io/posts/claude-code-mastery/] One caution the docs are explicit about: a reviewer asked to find gaps will report some even when the work is sound, so tell it to flag only what affects correctness or the stated requirements, and treat the rest as optional. Chasing every finding leads to over-engineering. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/best-practices]

### When parallelism hurts

Parallelism is not free, and the practitioners are blunt about it. It helps for truly independent tasks, for wall-clock pressure, and for exploration that adds signal. It hurts the moment tasks have sequential dependencies, edit the same files, or shrink to the point where coordination costs more than the work itself. Master the single loop first; scale to three or five only once your briefs are crisp and your verification runs on its own.

---

## The keyboard-level layer

Underneath the lived practice is a muscle-memory layer, the keystrokes real users hit dozens of times a day. Defaults below were current at access date and are customizable; confirm yours in `/keybindings`. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/interactive-mode]

**Steering and recovery:**
- `Esc` stops Claude mid-action and preserves the work done so far, so you can redirect. The single most-used key. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/interactive-mode]
- `Esc Esc` on an empty prompt opens the rewind menu (also `/rewind`), restoring conversation, code, or both to a prior checkpoint. Rewinding a wrong turn beats typing a correction, because a correction leaves the failure in the window while a rewind drops it. Checkpoints and "Bash side effects are permanent" are Chapter 04's territory. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/interactive-mode]
- `Shift+Tab` cycles permission modes (default, accept-edits, plan, and auto where enabled). ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/permission-modes]
- `Ctrl+G` opens the proposed plan, or your prompt, in your external editor. Edit the plan, don't argue with it. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/interactive-mode]

**Feeding context fast:**
- `@path` references a file or directory so Claude reads it before responding, and pulls in nearby `CLAUDE.md` files. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/common-workflows]
- `Ctrl+V` pastes a clipboard image (use `Ctrl+V`, not `Cmd+V`, in most terminals; iTerm2 also accepts `Cmd+V`), or drag and drop. Screenshots of errors, designs, and diagrams are first-class context. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/common-workflows] ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/interactive-mode]
- Pipe stdin: `git log --oneline -20 | claude -p "summarize these recent commits"`, or `cat error.log | claude`. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/common-workflows] ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/best-practices]
- `!` runs a shell command directly and lands its output in context, like `!git status`. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/interactive-mode]
- `Cmd+Click` (Mac) or `Ctrl+Click` (Windows/Linux) an `[Image #N]` reference to open it in your viewer. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/common-workflows] The `[Image #N]` chip itself is inserted at the cursor by the paste shortcut above. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/interactive-mode]

**Context hygiene:**
- `/clear` between unrelated tasks, the single best fix for the kitchen-sink session. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/best-practices]
- `/compact <hint>` summarizes with focus, e.g. `/compact Focus on the API changes`. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/best-practices]
- `/btw` asks a side question that answers from current context, appears in a dismissible overlay, and never enters conversation history. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/interactive-mode]

---

## Commands that keep a session running

Three documented commands automate the loops practitioners run by hand, and they compose rather than compete. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/goal]

| Command | Next turn starts when | Stops when | Use for |
|---|---|---|---|
| `/goal <condition>` | The previous turn finishes | A separate model confirms the condition holds | Run-to-condition work: "all tests in test/auth pass and the lint step is clean" ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/goal] |
| `/loop` | A time interval elapses | You stop it, or Claude decides it's done | Polling while a session is open: a babysitting loop that shepherds PRs ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/scheduled-tasks] |
| `Stop` hook | The previous turn finishes | Your own script or prompt decides | A deterministic gate that must hold every time ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/best-practices] |

`/goal` is the one to reach for first. After each turn a small fast model checks whether your condition holds against what Claude has surfaced in the conversation, and if not, Claude starts another turn instead of returning control to you. Write the condition as something Claude's own output can demonstrate, name the check ("`npm test` exits 0"), and bound it with a clause like "or stop after 20 turns." It pairs with auto mode: auto mode removes the per-tool prompts, `/goal` removes the per-turn prompts. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/goal] These are exactly the version-pinned features to confirm in the changelog before you depend on them.

---

## Common failure patterns

The best-practices doc names five failure modes that every practitioner eventually hits, and memorizing the fix is the fast path past them. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/best-practices]

| Failure | Symptom | Fix |
|---|---|---|
| Kitchen-sink session | One task, then an unrelated one, then back; context full of noise | `/clear` between unrelated tasks |
| Correcting forever | Same mistake, correction, still wrong, correct again | After two failed corrections, `/clear` and write a better initial prompt |
| Over-specified `CLAUDE.md` | Claude ignores half your rules | Prune ruthlessly; convert must-always rules to hooks |
| Trust-then-verify gap | Plausible code that misses edge cases | Always provide a check; if you can't verify it, don't ship it |
| Infinite exploration | An unscoped "investigate X" reads hundreds of files | Scope it narrowly or delegate to a subagent so it doesn't eat main context |

The doc's own closing note is worth keeping in mind: these are starting points, not laws. Sometimes you should let context accumulate because you are deep in one problem and the history is valuable; sometimes you should skip planning because the task is exploratory. Pay attention to what works and the intuition follows. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/best-practices]

---

## The senior operating model

Across the official guidance, Anthropic's teams, and independent engineers, the same model keeps surfacing, and it is a model of management: you run a small team of agents and read their output.

You set direction and taste; the agent provides throughput. The bottleneck moved from whether you can write the code to whether you can specify and verify it. ^[source: web | 2026-06-18 | https://claude.com/blog/how-anthropic-teams-use-claude-code] The feedback loop is the product: whoever wires the tightest verify-loop gets the most autonomous, highest-quality runs. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/best-practices] The system compounds, because corrections become rules and rules become a memory and skill library that makes every future session sharper; the docs say it plainly, that a `CLAUDE.md` "compounds in value over time." ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/best-practices] Autonomy is earned per task-type and calibrated to blast radius: plan-and-review for the unfamiliar and irreversible, high-trust auto mode for the bounded and reversible. ^[source: web | 2026-06-18 | https://simonwillison.net/2025/Oct/5/parallel-coding-agents/] And parallelism multiplies a good single-agent practice rather than fixing a bad one.

> **A last word for a regulated context.** The flashiest items here are the most version-volatile and, in health-tech and other regulated settings, the ones most worth scrutinizing for data handling and auditability before adoption. The durable practices are the unglamorous ones: verify, write rules instead of corrections, clear context aggressively, scope your investigations, and review in a fresh context. Build the boring habits first. The features will keep changing under you, and the changelog is the only thing that stays current. ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/changelog.md]

---

## Sources

- Anthropic, "How Anthropic teams use Claude Code": https://claude.com/blog/how-anthropic-teams-use-claude-code
- Claude Code docs, Best practices: https://code.claude.com/docs/en/best-practices
- Claude Code docs, Common workflows: https://code.claude.com/docs/en/common-workflows
- Claude Code docs, Interactive mode (keyboard shortcuts, `/btw`, voice): https://code.claude.com/docs/en/interactive-mode
- Claude Code docs, Permission modes (auto mode, plan mode): https://code.claude.com/docs/en/permission-modes
- Claude Code docs, Worktrees: https://code.claude.com/docs/en/worktrees
- Claude Code docs, Agent view: https://code.claude.com/docs/en/agent-view
- Claude Code docs, `/goal`: https://code.claude.com/docs/en/goal
- Claude Code docs, Scheduled tasks (`/loop`): https://code.claude.com/docs/en/scheduled-tasks
- Claude Code docs, Changelog (verify version-pinned claims): https://code.claude.com/docs/en/changelog.md
- Arpan Patel, "Beyond the Prompt: Claude Code" (his own blog): https://arps18.github.io/posts/claude-code-mastery/
- Simon Willison, "Embracing the parallel coding agent lifestyle" (his own blog, Oct 2025): https://simonwillison.net/2025/Oct/5/parallel-coding-agents/
