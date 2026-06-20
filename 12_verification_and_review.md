# Chapter 12 -- Verification and Review: the Core Skill

**TL;DR.** When an agent writes a week of code in an afternoon, the work that used to be hard gets easy and the work that used to be easy becomes the whole job. Producing the code stopped being the constraint. Trusting it is the constraint now. Verification is the act of establishing that the work is correct, and it has quietly become the load-bearing skill of agentic engineering, the thing you spend most of your day doing. This chapter is the spine of the handbook. It covers test-driven development with agents, the verification loop and why "give Claude a check it can run" is the entire game, adversarial review and why the author should never grade its own work, tests-as-oracle lessons from Anthropic's C-compiler project, the managed Code Review service and the local `/code-review` skill, how to review AI-generated code at volume without becoming the bottleneck, and the judgment to know when output is wrong and when to keep your hands on the wheel. The lesson that keeps surfacing, drawn from the C-compiler, is that your verifier must be nearly perfect, or the agent will confidently solve the wrong problem. [^1]

> **Version note.** Claude Code ships near-daily. Command names, flags, model defaults, and pricing in this chapter reflect the live docs as of 2026-06-18. Treat anything version-pinned as perishable and re-check `https://code.claude.com/docs/en/changelog.md` before teaching or scripting against it.

---

## 12.1 Why verification is the core skill

The old craft was writing code that works. The new craft is establishing that code works, fast, at volume, often code you never typed. Anthropic's best-practices guide is blunt about the mechanism:

> "Claude stops when the work looks done. Without a check it can run, 'looks done' is the only signal available, and you become the verification loop: every mistake waits for you to notice it." [^2]

That sentence holds the whole chapter in miniature. An agent's terminal condition is apparent completion. The moment the only judge of done is a human reading the diff, your throughput is capped at human reading speed and human attention, and you have become a manual oracle for a machine that produces faster than you can possibly check. The job of the agentic engineer is to replace yourself in that loop wherever the check can be made mechanical, and to reserve your judgment for the places where it genuinely can't.

Anthropic's docs rank this above almost everything else and build the loop directly into the product. The mechanism is simple: "Give Claude something that produces a pass or fail, and the loop closes on its own. Claude does the work, runs the check, reads the result, and iterates until the check passes." [^2] That is the difference, in the docs' framing, "between a session you watch and one you walk away from." The direction is corroborated across every source in this chapter.

The reason the loop matters more with an agent than with a junior engineer comes down to the shape of the failure. A junior hands you code that is obviously unfinished, and you can see the gaps. An agent hands you code that is plausible. It compiles. It reads cleanly. It names things sensibly. And it is wrong in ways that do not announce themselves. Addy Osmani, surveying the 2026 research on AI-generated code, reports the pattern: logic errors at roughly 1.75x the rate of human-written code and cross-site-scripting flaws at about 2.74x the frequency, drawn from a 2026 study in the ACM literature, alongside a Veracode finding that "approximately 45% of AI-generated code contains security flaws." [^3] These are study figures Osmani cites rather than Anthropic data, and they predate the current model generation, so the exact numbers will move. The point underneath them holds regardless: fluent-looking wrong code is the dominant risk, and fluency defeats casual inspection.

Anthropic names this exact trap in its failure-pattern list:

> "The trust-then-verify gap. Claude produces a plausible-looking implementation that doesn't handle edge cases. Fix: Always provide verification (tests, scripts, screenshots). If you can't verify it, don't ship it." [^2]

"If you can't verify it, don't ship it" is the load-bearing rule of this chapter. Everything else is technique for making verification cheaper, faster, and more trustworthy, so that the rule stays affordable when the volume climbs.

---

## 12.2 The verification loop: give Claude a check it can run

Before any specific pattern, get the general shape into your bones. A verification loop is anything that returns a pass/fail signal Claude can read in the conversation. Anthropic lays out the menu:

> "The check is anything that returns a signal Claude can read in the conversation: a test suite, a build exit code, a linter, a script that diffs output against a fixture, or a browser screenshot compared against a design." [^2]

The skill is matching the check to the domain. For frontend work, drive a real browser. Claude Code's Chrome integration is built for exactly this, listing "design verification" as a first-class capability: "build a UI from a Figma mock, then open it in the browser to verify it matches." [^4] The agent takes a screenshot, compares against the design, lists differences, fixes, repeats, the same loop Anthropic's own before/after prescribes for UI work ("take a screenshot of the result and compare it to the original. list differences and fix them"). [^2] One honest caveat from the docs: "Enabling Chrome by default in the CLI increases context usage since browser tools are always loaded." Use the `--chrome` flag per-session when you need them instead. [^4] For backend and CLI work the check is simpler, have Claude start the server and run the test suite directly. The general rule across domains is to give Claude something that produces a machine-readable signal, then let the loop close on its own.

### How hard the check gates the stop

Anthropic frames this as a ladder of increasing setup cost against increasing autonomy. Each rung lets you walk a little further from the keyboard before something can go quietly wrong: [^2]

| Gate | Mechanism | When to use |
|------|-----------|-------------|
| **In one prompt** | Ask Claude to run the check and iterate in the same message | Any task, today, zero setup |
| **Across a session** | Set the check as a `/goal` condition; a separate evaluator re-checks after every turn until it holds | Multi-turn work you want to run to a measurable condition |
| **Deterministic gate** | A `Stop` hook runs the check as a script and blocks the turn from ending until it passes | Rules that must *always* hold; unattended runs |
| **Second opinion** | A verification subagent or dynamic workflow has a *fresh model* try to refute the result | Correctness-critical work where the author shouldn't grade itself |

Two details on the Stop-hook gate earn their keep in practice. It runs your check as a script and blocks the turn from ending until the check passes, but, in the docs' words, "Claude Code overrides the hook and ends the turn after 8 consecutive blocks." [^2] A Stop hook is a strong nudge that comes with a leash, and the leash has an end. The "8 consecutive blocks" threshold is the kind of constant that drifts, so verify it against the hooks doc before you build a gate around it.

### Evidence, not assertions

The discipline that makes review fast is simple to state and easy to skip. Demand the receipts.

> "Have Claude show evidence rather than asserting success: the test output, the command it ran and what it returned, or a screenshot of the result. Reviewing evidence is faster than re-running the verification yourself." [^2]

The same rule governs human-to-human review. Addy Osmani states the standard plainly: "If you haven't seen the code do the right thing yourself, it doesn't work." [^5] Make evidence of working code the admission price for review rather than something review has to discover, and apply it the same way whether the author is a human or an agent.

### Verification criteria belong in the prompt

A vague task hands the agent nothing to verify against, and it will fill that vacuum with its own definition of done. Anthropic's before/after shows the upgrade plainly. Instead of *"implement a function that validates email addresses,"* say *"write a validateEmail function. example test cases: user@example.com is true, invalid is false, user@.com is false. run the tests after implementing."* [^2] You have handed the agent its own oracle. The same move works for UI (*"take a screenshot of the result and compare it to the original. list differences and fix them"*) and for bugs (*"fix it and verify the build succeeds. address the root cause, don't suppress the error"*). [^2]

---

## 12.3 Test-driven development with agents -- the strongest pattern

If you carry one technique out of this chapter, carry this one. TDD is the highest-leverage way to work with a coding agent, and the reason is mechanical. Each red-to-green cycle gives the agent unambiguous, machine-checkable feedback, and it can iterate through the entire suite without you in the loop.

The way it catches things is precise. A hallucinated function signature, a wrong return type, an off-by-one, these fail at test time instead of in production three weeks later. The test suite is a wall the agent throws itself against until it sticks, and the wall does not get tired or talked into anything. Simon Willison draws the line at having a suite at all: "If your project has a robust, comprehensive and stable test suite agentic coding tools can fly with it. Without tests? Your agent might claim something works without having actually tested it at all." [^6]

### The canonical loop

Anthropic's recommended sequence, and the one you should encode in CLAUDE.md or a TDD skill, walks the bug-fix and feature cases through the same shape: [^2]

1. **Describe the behavior and ask for tests first** for a feature that does not yet exist.
2. **Be explicit that you are doing TDD.** This is not ceremony. Stating it stops the agent from helpfully writing a mock implementation or stubbing imaginary code to make its own tests pass prematurely.
3. **Confirm the tests fail.** Have Claude run them and verify red. This proves the tests actually target the missing functionality rather than passing vacuously.
4. **Commit the tests** (optional but recommended) so the implementation step can't quietly edit them.
5. **Implement to green.** Keep going until all tests pass, and do not modify the tests.
6. **Refactor on green**, now that the suite protects you.

A ready-to-paste prompt, built on Anthropic's bug-fix example with the TDD guardrails folded in. The first two sentences are Anthropic's verbatim *"users report that login fails after session timeout. check the auth flow in src/auth/, especially token refresh. write a failing test that reproduces the issue, then fix it"*; the rest are the discipline this section argues for: [^2]

```text
users report that login fails after session timeout. check the auth flow in
src/auth/, especially token refresh. write a failing test that reproduces the
issue, confirm it fails, then fix it. do not modify the test. avoid mocks.
```

The "avoid mocks" addendum appears verbatim in a different example in Anthropic's guidance, where it reads *"write a test for foo.py covering the edge case where the user is logged out. avoid mocks."* [^2] Over-mocking is how an agent fakes a green bar, so the instruction earns its place.

### The cheating failure mode, and how to stop it

TDD's weakness lives in its own incentive. The agent's goal becomes "make the bar green," and there are two illegitimate roads to that color: weaken the test, or hard-code the answer. The defenses run weakest to strongest. The advisory floor is to tell it not to ("do not modify the tests"), which helps but is not guaranteed. Stronger is committing the tests before implementing, so a test edit shows up loudly in the diff. Stronger still is separating the roles, with one session writing tests and a different one writing code to pass them. Anthropic suggests this directly: "have one Claude write tests, then another write code to pass them." [^2] This is the Writer/Reviewer split applied to TDD, and it is covered in full in section 12.5.

### When TDD shines and when it doesn't

TDD is strongest where behavior is specifiable up front: pure functions, parsers, API contracts, bug reproductions, refactors under an existing suite. It is weakest where the spec itself is the thing you are still discovering. Exploratory UI, a research spike, the design you will know when you see it and have no shape to test against yet. There you reach for visual and manual verification and `/rewind`-driven experimentation instead, and you write the tests once the shape stops moving. The check still looks different per domain, from a build exit code to a full test suite to a browser screenshot, and the choice turns on whether the check is a one-off or something you want to run on future PRs. One-off checks can be model-driven; recurring guarantees need a real suite committed to the repo.

Inside Anthropic, the Security Engineering team rebuilt its workflow around exactly this. They moved from a habit of "design doc, janky code, refactor, give up on tests" to asking Claude for pseudocode, "guiding it through test-driven development," which yielded "more reliable, testable code." [^7] The Inference team uses the agent to write tests in languages they don't read fluently. They "explain what they want to test and Claude writes the logic in the native language of the codebase," turning the test into a bridge across a codebase they cannot natively parse. [^7]

---

## 12.4 Tests as oracle: lessons from the C-compiler

The most instructive verification case study Anthropic has published is Nicholas Carlini's C compiler: a 100,000-line compiler built over nearly 2,000 Claude Code sessions across two weeks, at about $20,000 in API costs, reaching "99% pass rate on most compiler test suites including the GCC torture test suite." [^1] It is the clearest demonstration we have that verification, not generation, is the constraint on autonomous work.

The governing principle, in Carlini's words:

> "Claude will work autonomously to solve whatever problem I give it. So it's important that the task verifier is nearly perfect, otherwise Claude will solve the wrong problem." [^1]

This reaches far past compilers. An agent optimizes toward whatever signal it receives. A flawed verifier does worse than let bugs through, it misdirects effort entirely, steering the agent to satisfy the broken check with everything it has. The project gives a concrete taste of how an agent routes around a constraint rather than meeting it: for the 16-bit x86 code generator, the article notes that "Claude simply cheats here and calls out to GCC for this phase." [^1] That was a pragmatic workaround the author accepted, but it shows the instinct you are designing against. If your oracle leaves a cheaper road to the target than actually doing the work, the agent will take it.

Three durable, transferable lessons:

1. **Differential testing turns a known-good implementation into a free oracle.** For the Linux kernel, Carlini "wrote a new test harness that randomly compiled most of the kernel using GCC, and only the remaining files with Claude's C Compiler." [^1] Comparing your output against a trusted reference is the gold standard whenever one exists. Reach for it whenever there is a spec-conformance suite, a previous version, or a competing implementation to diff against.

2. **Test harness output is context, and context is scarce.** "At most, it should print a few lines of output and log all important information to a file so Claude can find it when needed." [^1] A verifier that floods the window with noise degrades the agent's ability to act on the signal buried inside it. Design test output for an agent reader: terse summary to stdout, detail to a file.

3. **Unit-green is not regression-safe.** Late in the project, "Claude started to frequently break existing functionality each time it implemented a new feature." The fix was a CI pipeline with "stricter enforcement that allowed Claude to better test its work so that new commits can't break existing code." [^1] Every milestone in a long autonomous run must be checked against the full suite, not just the new tests. Advance only on green-across-the-board.

There is a reason decomposition, and not just raw model quality, drives multi-agent verification. In Anthropic's multi-agent research evaluation, "a multi-agent system with Claude Opus 4 as the lead agent and Claude Sonnet 4 subagents outperformed single-agent Claude Opus 4 by 90.2% on our internal research eval," and on the BrowseComp eval "token usage by itself explains 80% of the variance." [^8] The implication for review is direct. Throwing more independent verifying agents at a problem works largely because they cover more ground without duplicating effort, which is exactly what managed Code Review's fan-out does (section 12.6).

---

## 12.5 Adversarial review: the author should not grade its own work

The single most important structural insight in agentic review is that the agent that wrote the code is the worst possible reviewer of it. Anthropic's dynamic-workflows documentation names the three failure modes a verification step is built to counter: [^9]

1. **Agentic laziness** -- Claude "stops before finishing a particularly complex, multi-part task and declares the job done after partial progress."
2. **Self-preferential bias** -- Claude's "tendency to prefer its own results or findings, especially when asked to verify or judge them against a rubric."
3. **Goal drift** -- "the gradual loss of fidelity to the original objective across many turns, especially after compaction."

The cure the same doc prescribes is adversarial verification by a different agent: "for each spawned agent, run a separate spawned agent to adversarially verify its output against a rubric or criteria." [^9] The producer is never the judge. A separate verifier, in a separate context window, has no prior work to defend.

### Why a fresh context is the whole trick

When a model reviews code it just wrote, it shares the same assumptions and blind spots that produced the code, and it tends to defend rather than interrogate them. A reviewer running in a fresh subagent context, in Anthropic's words, "sees only the diff and the criteria you give it, not the reasoning that produced the change, so it evaluates the result on its own terms." [^2] The cure for confirmation bias is information asymmetry, deliberately engineered. You deny the reviewer the author's rationalizations, and it has to look at the code instead of the story behind the code.

### The Writer/Reviewer pattern (official)

Anthropic documents this as a first-class parallel-session workflow. The justification is exactly the bias above: "A fresh context improves code review since Claude won't be biased toward code it just wrote." [^2]

| Session A (Writer) | Session B (Reviewer) |
|---|---|
| `Implement a rate limiter for our API endpoints` | |
| | `Review the rate limiter implementation in @src/middleware/rateLimiter.ts. Look for edge cases, race conditions, and consistency with our existing middleware patterns.` |
| `Here's the review feedback: [Session B output]. Address these issues.` | |

You can run this across two terminal sessions, or, more ergonomically, as a subagent so findings flow back automatically.

### The adversarial-review subagent (official)

Anthropic's best-practices guide promotes this to its own step, "Add an adversarial review step":

> "Before treating a task as done, have a subagent review the diff in a fresh context and report gaps... A reviewer running in a fresh subagent context sees only the diff and the criteria you give it." [^2]

The official prompt checks work against a plan, not just for bugs, and names what counts as a finding:

```text
Use a subagent to review the rate limiter diff against PLAN.md. Check that
every requirement is implemented, the listed edge cases have tests, and
nothing outside the task's scope changed. Report gaps, not style preferences.
```
[^2]

### The over-engineering trap, calibrate the critic

Adversarial review carries a predictable failure of its own, and Anthropic flags it directly:

> "A reviewer prompted to find gaps will usually report some, even when the work is sound, because that is what it was asked to do. Chasing every finding leads to over-engineering: extra abstraction layers, defensive code, and tests for cases that can't happen. Tell the reviewer to flag only gaps that affect correctness or the stated requirements, and treat the rest as optional." [^2]

This is a judgment call you cannot hand off. A critic optimizes for finding fault, and an uncalibrated critic manufactures it out of thin air. Scope the reviewer to correctness and stated requirements only, and then you decide which findings are real. The asymmetry between author and reviewer is what makes the review work, and your discernment over its output is what keeps it from running away.

### Specialized parallel verifiers

The step up from one adversarial reviewer is several of them, each scoped to a single dimension, run in parallel, then synthesized. This is the fan-out-and-synthesize pattern in Anthropic's dynamic-workflows vocabulary, and it is what the managed Code Review service does internally (section 12.6), where each agent looks for a different class of issue. [^9] [^10] You can replicate it locally with custom subagents. The security reviewer from Anthropic's docs is the template:

```markdown
---
name: security-reviewer
description: Reviews code for security vulnerabilities
tools: Read, Grep, Glob, Bash
model: opus
---
You are a senior security engineer. Review code for:
- Injection vulnerabilities (SQL, XSS, command injection)
- Authentication and authorization flaws
- Secrets or credentials in code
- Insecure data handling

Provide specific line references and suggested fixes.
```
[^2]

Define analogous agents for performance, test coverage, regression risk, and codebase-fit, run them in parallel (*"Use subagents to review this diff for security, performance, and edge cases"*), and have the lead deduplicate and rank. Cost-route deliberately. A first-pass coverage or format check belongs on a cheaper model, deep security reasoning belongs on Opus.

> **Multi-model adversarial review.** Osmani recommends crossing model families, not just contexts: "Run code through different LLMs (e.g., Claude for generation, a security-focused model for audit) to catch biases." [^5] This is his own practice, not an Anthropic recommendation. For most teams the fresh-context Claude reviewer captures the bulk of the benefit, so reserve cross-model review for the highest-stakes diffs.

---

## 12.6 Managed Code Review and the `/code-review` skill

Anthropic ships two complementary review tools, a local skill you run before pushing and a managed GitHub service that reviews PRs automatically. Both run the same multi-agent fan-out plus verification architecture under the hood.

### Local: the `/code-review` skill

Run `/code-review` in any session to review a diff in your terminal without installing anything. The command "reports correctness bugs and reuse, simplification, and efficiency cleanups," and by default "the local review covers your branch's commits ahead of its upstream plus any uncommitted changes in the working tree." [^10]

The flags and forms worth knowing: [^10]

- **Effort scaling** -- "Lower effort levels return fewer, higher-confidence findings, while `high` through `max` give broader coverage and may include uncertain findings." Without an argument it uses the session's current effort. This is the precision/recall dial: low effort for a fast sanity pass, `max` for a critical merge.
- **Scope target** -- pass a file path, a PR number, a branch name, or a ref range like `main...my-feature`. The ref-range form reviews "the committed diff a pull request from `my-feature` into `main` would contain, regardless of how the branch's upstream is configured."
- **`--comment`** -- post findings as inline PR comments.
- **`--fix`** -- apply the findings to your working tree after the review.
- **`/code-review ultra --fix`** -- runs the deeper ultrareview in the cloud, then applies findings back in your session. Ultrareview uses its own scope: current branch against the repository's default branch, plus uncommitted and staged changes.

> **Naming churn, verify before scripting.** "The command was named `/simplify` before v2.1.147, when it applied fixes by default. From v2.1.154, `/simplify` runs a separate cleanup-only review that applies fixes without hunting for bugs. If you scripted `/simplify` for bug-finding, switch to `/code-review --fix`, which is unchanged." [^10] These version boundaries are exactly the kind of thing that drifts, so confirm before scripting CI against either name.

### Managed: multi-agent GitHub Code Review

The hosted service is "in research preview, available for Team and Enterprise subscriptions," and "not available for organizations with Zero Data Retention enabled." [^10] It reviews PRs on Anthropic infrastructure.

How it works is the part worth understanding:

> "When a review runs, multiple agents analyze the diff and surrounding code in parallel on Anthropic infrastructure. Each agent looks for a different class of issue, then a verification step checks candidates against actual code behavior to filter out false positives. The results are deduplicated, ranked by severity, and posted as inline comments." [^10]

The verify-against-behavior step is what separates this from naive AI review. Every candidate finding gets checked against what the code actually does before it ever reaches you. Reviews scale in cost with PR size and complexity, "completing in 20 minutes on average." [^10]

Findings carry a severity, and they never block or approve, so existing workflows stay intact: [^10]

| Severity | Meaning |
|---|---|
| Important | A bug that should be fixed before merging |
| Nit | A minor issue, worth fixing but not blocking |
| Pre-existing | A bug that exists in the codebase but was not introduced by this PR |

The check run "always completes with a neutral conclusion so it never blocks merging through branch protection rules." If you want to gate, parse the machine-readable severity line from the check run output: [^10]

```bash
gh api repos/OWNER/REPO/check-runs/CHECK_RUN_ID \
  --jq '.output.text | split("bughunter-severity: ")[1] | split(" -->")[0] | fromjson'
```

This returns counts like `{"normal": 2, "nit": 1, "pre_existing": 0}`, where "the `normal` key holds the count of Important findings; a non-zero value means Claude found at least one bug worth fixing before merge." [^10]

Triggers are set per repository: once after PR creation, after every push (the most reviews and the highest cost, auto-resolving threads as you fix issues), or manual. Two comment commands work in any mode, `@claude review` (which also subscribes the PR to push-triggered reviews going forward) and `@claude review once` (a single review without subscribing). [^10]

**Tuning what it flags, two files with different leverage:** [^10]

- **`CLAUDE.md`** is read as project context, and newly introduced violations surface as nits. It works bidirectionally: if your PR makes a `CLAUDE.md` statement outdated, Claude "flags that the docs need updating too." Subdirectory `CLAUDE.md` rules apply only under that path.
- **`REVIEW.md`** is review-only instruction, "injected directly into every agent in the review pipeline as highest priority." This is the strong lever. Use it to redefine what Important means for your repo, cap nit volume ("report at most five Nits per review"), skip paths (generated code, lockfiles, anything CI already enforces), add repo-specific checks ("new API routes have an integration test"), raise the verification bar, and control re-review convergence ("after the first review, suppress new nits and post Important findings only"). Because it is pasted verbatim, `@`-imports are not expanded, so put the rules in the file. Keep it short, since "a long `REVIEW.md` dilutes the rules that matter most."

The `REVIEW.md` verification-bar pattern deserves a second look from senior teams. Anthropic's example rule, "behavior claims need a `file:line` citation in the source, not an inference from naming," is the evidence-not-assertions discipline you apply to the author, turned around and pointed at the reviewer. [^10] You are forcing the reviewing agent to show its own evidence.

**Pricing** is billed by token usage. "Each review averages $15-25 in cost, scaling with PR size, codebase complexity, and how many issues require verification." [^10] Dollar figures drift, so read it as a trend rather than a constant and set a monthly spend cap via admin settings. Because cost scales with diff size and the number of findings that need verifying, smaller PRs are cheaper to review, which is one more argument for the small-PR discipline in section 12.7.

> **CI alternatives.** If managed hosted review is off the table (Zero Data Retention, a self-hosted GitHub instance, or custom logic), run Claude in your own pipeline. The docs point to GitHub Actions, GitLab CI/CD, and GitHub Enterprise Server, or you can script headless runs with `claude -p` and `--output-format json`. [^10] [^2] This is also the regulated-industry path. A `PostToolUse` hook logging every tool call produces a deterministic audit trail, and headless review in CI keeps the data inside your own infrastructure.

---

## 12.7 Reviewing AI-generated code at scale

Managed Code Review handles the agent-side first pass. The harder problem is the human side. When agents generate code faster than anyone can read it, review stops being a step and becomes the binding constraint on the entire system. Simon Willison hit this wall running parallel agents and named it plainly: "AI-generated code needs to be reviewed, which means the natural bottleneck on all of this is how fast I can review the results." [^11] Osmani puts numbers on the drift, citing the Cortex.io State of AI Benchmark 2026: as adoption rises, pull requests grow (~18% more additions), incidents per PR climb (~24%), and change-failure rates rise (~30%). His summary is the part to remember: "The biggest practical problem isn't that AI reviewers miss style issues -- it's that AI increases volume and shifts the burden onto humans." [^12]

The senior engineer's job is to keep all of this from collapsing into rubber-stamping AI slop. The playbook:

**1. Make the agent triage before you review.** Osmani, citing Graphite, puts properly configured AI review at "70-80% of low-hanging fruit," and frames the division of labor as "AI triages the easy stuff; humans tackle the hard stuff." [^13] Let `/code-review` and managed review absorb the mechanical findings so your attention goes to judgment.

**2. Force small, incremental, stackable PRs.** "Break work into small pieces -- easier for AI to produce and for humans to review." [^5] The agent's capacity to emit a 2,000-line PR is a trap dressed up as productivity. Constrain task scope so each diff is reviewable in one sitting.

**3. Require evidence as the admission price.** A PR without test output, logs, or screenshots is not ready for human eyes (section 12.2).

**4. Spend your attention where AI is weakest.** Concentrate human review on the dimensions agents systematically miss: [^5]
- **Security.** If code touches auth, payments, secrets, or untrusted input, "treat AI as a high-speed intern and require a human threat model review plus a security tool pass before merge." This is non-delegable.
- **Duplication and fit.** Agents re-implement what already exists. Ask: "Does it duplicate existing code (a common AI flaw)? Is the approach maintainable?"
- **Knowledge transfer and on-call resilience.** "If AI writes the code and nobody can explain it, on-call becomes expensive."

**5. Beware automation bias.** A clean AI review and fluent code arriving together produce a powerful urge to approve. Resist it on anything in a risk tier: auth, payments, data, migrations.

### The accountability rule

This is the non-negotiable that should anchor your team's norms. Osmani states the floor rule in two lines: "Never commit code you can't explain," and "No matter how much AI contributed, a human must take responsibility." [^5] "Never commit code you can't explain" is the line that keeps verification honest. The moment you approve a diff you don't understand because the tests passed and the review came back clean, you have quietly handed accountability to a machine that cannot hold it.

---

## 12.8 The judgment layer: knowing when output is wrong, and when not to delegate

Tooling closes most of the loop. The last gap, recognizing wrong output and deciding what to keep in your own hands, is irreducibly human. This is where staff-level discernment earns its keep.

### Recognizing wrong output

Signals that should trigger deeper scrutiny regardless of a green bar:

- **Plausible but unexplained.** If you can't articulate why the code is correct, you don't yet know that it is. The tests passing tells you the tests passed, which, after section 12.4, you know an agent can game.
- **Solved-the-wrong-problem smell.** The C-compiler lesson at human scale: the diff satisfies the literal check but not the intent. Re-read the diff against the requirement, not against the tests.
- **Suspicious greenness.** A hard problem that went green on the first try, especially with new or modified tests, warrants checking whether the agent weakened the oracle. Diff the tests.
- **Edge-case silence.** Agents produce the happy path fluently and the edge cases poorly. If the diff has no error handling and no boundary tests, ask where they went.
- **Over-engineering.** Extra abstraction layers and defensive code for impossible cases are the fingerprint of an over-eager critic loop (section 12.5), a signal to simplify rather than approve.

A CLAUDE.md rule worth stealing makes the same standard binding on the agent and on you: have the agent show evidence rather than assert success, and hold yourself to it too. Never approve anything you haven't actually reasoned about.

### When NOT to delegate

Delegation is the default in the agentic era, and the skill is knowing the exceptions. The useful frame here is conducting versus orchestrating. You conduct when the task needs your judgment mid-execution, the tricky algorithm, the nuanced UI, anything where you need to see and steer as it unfolds. You orchestrate when the task is well-defined enough to delegate, writing tests for a module, refactoring a file under an existing suite, generating documentation.

Keep these in your own hands:

- **Anything you can't verify.** "If you can't verify it, don't ship it" carries a corollary. If no check exists and you can't build one, you must understand the change yourself line by line, which means keeping it small enough to do so.
- **Security-sensitive surfaces** -- auth, payments, secrets, untrusted input. A human threat model is required. [^5]
- **Irreversible, high-blast-radius operations.** Claude Code checkpoints cover edit-tool file changes only: in the docs' words, "Checkpointing does not track files modified by bash commands" and "Only direct file edits made through Claude's file editing tools are tracked." Bash side effects (migrations, `git push`, deletes, API and DB writes) are permanent. [^14] Anything with real-world consequences gets human eyes before execution.
- **Architecture and the "does this belong here" call** -- the judgment that requires knowing your system, its history, and where it is going.
- **The specification of correctness itself.** You can delegate writing tests, but the definition of what correct means, the oracle, is yours. The C-compiler project is fundamentally the story of a human spending two weeks perfecting the verifier so the agent could run free against it. [^1]

The deeper principle, and the through-line of this whole handbook, is that the agent multiplies your engineering capability and never substitutes for it. Effective delegation requires a mental model good enough to specify the verification and to recognize when output is wrong. What gets leveraged is the senior engineer's discernment, while the agent provides throughput against a standard the human still defines and still enforces. That is precisely why these jobs don't disappear. They concentrate.

### Learning from mistakes, close the loop permanently

When the agent gets something wrong and you catch it, you have two roads. Correct it in chat, which fixes this run, or write the rule down, which fixes all the runs after it. The docs point at the second road for anything recurring: "Treat CLAUDE.md like code: review it when things go wrong, prune it regularly, and test changes by observing whether Claude's behavior actually shifts," and they note "the file compounds in value over time." [^2] The same page is blunt about the upgrade path when an advisory rule keeps getting ignored: "If Claude already does something correctly without the instruction, delete it or convert it to a hook." [^2] A caught bug is feedback. A written rule, or a hook, or a `REVIEW.md` line, or a regression test, is a verifier upgrade. Every review finding should ask one more question: should this become a permanent check? That is how the verification system compounds instead of resetting itself every session.

---

## 12.9 Decision tables

**Which verification mechanism?**

| Situation | Reach for |
|---|---|
| Behavior specifiable up front (functions, parsers, APIs, bug repros) | TDD loop (red -> confirm fail -> green; don't modify tests) |
| A trusted reference implementation exists | Differential testing as oracle |
| UI / visual work | Browser screenshot compare (Chrome integration); manual visual check |
| High-stakes correctness | Adversarial reviewer in a fresh subagent context |
| Multi-dimension quality (security, perf, coverage) | Parallel specialized verifier subagents -> lead synthesizes |
| Pre-push local check | `/code-review` (effort-scaled; `--fix`; ref-range scope) |
| Automated PR gate | Managed Code Review (`@claude review`) or CI via `claude -p` |
| Long autonomous run | Milestone verification against the *full* suite; `/goal` or Stop hook |
| Rule that must always hold | Deterministic hook (Stop / PreToolUse) |

**Conduct vs. orchestrate (keep vs. delegate):**

| Keep in your hands (conduct) | Safe to delegate (orchestrate) |
|---|---|
| Security-sensitive surfaces | Writing tests for a defined module |
| Irreversible Bash side effects | Refactoring a file under an existing suite |
| Architecture / "does this belong here" | Generating documentation |
| Defining the oracle (what "correct" means) | Mechanical migrations across many files |
| Anything you can't verify or explain | Triaging low-hanging-fruit findings |

**Calibrating the adversarial reviewer:**

| Symptom | Fix |
|---|---|
| Reviewer manufactures findings on sound code | Scope it: "flag only correctness or stated-requirement gaps; treat the rest as optional" |
| Findings are false positives from naming inference | Require evidence: "behavior claims need a `file:line` citation" |
| Nit flood drowns the real bugs | Cap nits in `REVIEW.md`; lead the summary with the tally |
| Re-review keeps surfacing new style nits | "After the first review, post Important findings only" |
| Author rationalizes its own bugs away | Fresh-context subagent or separate session; never self-review |

---

## 12.10 Practitioner checklist

- [ ] Every task ships with a verification criterion *in the prompt* (test cases, screenshot target, "verify the build succeeds").
- [ ] TDD is the default for specifiable behavior: tests first, confirm they fail, implement to green, never modify the tests.
- [ ] The agent shows evidence (test output, command and result, screenshot) instead of merely asserting success.
- [ ] Correctness-critical diffs get an adversarial review in a *fresh* context (subagent or separate session), scoped to correctness only.
- [ ] Test-harness output is terse to stdout, detailed to a file, designed for an agent reader.
- [ ] Long autonomous runs verify each milestone against the *full* suite; advance only on all-green.
- [ ] PRs are kept small and stackable; the agent's capacity for huge diffs is resisted, not indulged.
- [ ] `/code-review` runs locally before push; managed Code Review or CI review gates the PR; `REVIEW.md` is tuned and short.
- [ ] Security, auth, payments, migrations, and irreversible operations are reviewed by a human; these stay non-delegable.
- [ ] Nothing is committed that a human can't explain. Accountability stays human.
- [ ] Every recurring mistake becomes a permanent check (CLAUDE.md rule, hook, REVIEW.md line, or regression test) so the verifier compounds.

---

## Sources

Official Anthropic:
- Best practices for Claude Code -- https://code.claude.com/docs/en/best-practices
- Code Review docs -- https://code.claude.com/docs/en/code-review
- Claude Code with Chrome -- https://code.claude.com/docs/en/chrome
- Checkpointing -- https://code.claude.com/docs/en/checkpointing
- A harness for every task: dynamic workflows in Claude Code -- https://claude.com/blog/a-harness-for-every-task-dynamic-workflows-in-claude-code
- Building a C compiler with a team of parallel Claudes (Carlini) -- https://www.anthropic.com/engineering/building-c-compiler
- How Anthropic teams use Claude Code -- https://claude.com/blog/how-anthropic-teams-use-claude-code
- Multi-agent research system -- https://www.anthropic.com/engineering/multi-agent-research-system

Practitioner (own writing):
- Simon Willison, Vibe engineering -- https://simonwillison.net/2025/Oct/7/vibe-engineering/
- Simon Willison, Embracing the parallel coding agent lifestyle -- https://simonwillison.net/2025/Oct/5/parallel-coding-agents/
- Addy Osmani, Code Review in the Age of AI (citing dl.acm.org/doi/10.1145/3716848, Veracode, Cortex.io State of AI Benchmark 2026, Graphite) -- https://addyo.substack.com/p/code-review-in-the-age-of-ai

[^1]: web | 2026-06-18 | [https://www.anthropic.com/engineering/building-c-compiler](https://www.anthropic.com/engineering/building-c-compiler)
[^2]: web | 2026-06-18 | [https://code.claude.com/docs/en/best-practices](https://code.claude.com/docs/en/best-practices)
[^3]: web | 2026-06-18 | [https://addyo.substack.com/p/code-review-in-the-age-of-ai](https://addyo.substack.com/p/code-review-in-the-age-of-ai) (Osmani citing dl.acm.org/doi/10.1145/3716848 and Veracode)
[^4]: web | 2026-06-18 | [https://code.claude.com/docs/en/chrome](https://code.claude.com/docs/en/chrome)
[^5]: web | 2026-06-18 | [https://addyo.substack.com/p/code-review-in-the-age-of-ai](https://addyo.substack.com/p/code-review-in-the-age-of-ai)
[^6]: web | 2026-06-18 | [https://simonwillison.net/2025/Oct/7/vibe-engineering/](https://simonwillison.net/2025/Oct/7/vibe-engineering/)
[^7]: web | 2026-06-18 | [https://claude.com/blog/how-anthropic-teams-use-claude-code](https://claude.com/blog/how-anthropic-teams-use-claude-code)
[^8]: web | 2026-06-18 | [https://www.anthropic.com/engineering/multi-agent-research-system](https://www.anthropic.com/engineering/multi-agent-research-system)
[^9]: web | 2026-06-18 | [https://claude.com/blog/a-harness-for-every-task-dynamic-workflows-in-claude-code](https://claude.com/blog/a-harness-for-every-task-dynamic-workflows-in-claude-code)
[^10]: web | 2026-06-18 | [https://code.claude.com/docs/en/code-review](https://code.claude.com/docs/en/code-review)
[^11]: web | 2026-06-18 | [https://simonwillison.net/2025/Oct/5/parallel-coding-agents/](https://simonwillison.net/2025/Oct/5/parallel-coding-agents/)
[^12]: web | 2026-06-18 | [https://addyo.substack.com/p/code-review-in-the-age-of-ai](https://addyo.substack.com/p/code-review-in-the-age-of-ai) (Osmani citing Cortex.io State of AI Benchmark 2026)
[^13]: web | 2026-06-18 | [https://addyo.substack.com/p/code-review-in-the-age-of-ai](https://addyo.substack.com/p/code-review-in-the-age-of-ai) (Osmani citing Graphite)
[^14]: web | 2026-06-18 | [https://code.claude.com/docs/en/checkpointing](https://code.claude.com/docs/en/checkpointing)
