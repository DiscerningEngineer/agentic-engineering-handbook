# Chapter 12 -- Evals for Coding Agents

**TL;DR.** A leaderboard number tells you which model is roughly more capable at GitHub-issue-shaped work, measured noisily, and almost nothing about whether your agent on your codebase got better when you changed something. SWE-bench Verified climbed from 49% to the low 80s in about a year and is now saturated near the top, where infrastructure config alone can swing a score by six points. The skill that compounds is building your own eval: twenty to fifty tasks drawn from your real failures, run in isolated environments, graded by code where the answer is verifiable, by a second model where quality is subjective, and by you where it counts. Learn the difference between pass@k (can it ever do this?) and pass^k (can it do this every time?), because they answer opposite questions and diverge fast. Write the eval before the capability exists, then tune the agent until it is green. And read the transcripts, because a green number is a claim, not a fact, until you have watched the runs that produced it. Underneath all of it sits one lesson from the hardest agentic coding project Anthropic has published: the verifier must be near-flawless, or the agent will cheerfully solve the wrong problem.

> **Version-sensitivity warning.** The models churn near-daily and the leaderboards reshuffle weekly. Every score, model name, and benchmark claim here is a snapshot as of 2026-06-18, presented as a trend rather than a fixed fact. Verify the live models page (`https://platform.claude.com/docs/en/about-claude/models/overview`) and the changelog (`https://code.claude.com/docs/en/changelog.md`) before teaching or quoting anything time-sensitive.

---

## 12.1 Why a senior engineer evals at all

The instinct of an experienced engineer is to trust their own judgment. Run the agent on a few real tasks, eyeball the diffs, ship it. That works right up until the day it quietly stops working, and three forces in the agentic era make that day arrive sooner than you would expect.

The first is non-determinism. The same prompt against the same model walks a different path every time. A change that looks like an improvement in one run may have been luck wearing the costume of a fix, and with a single sample you have no way to separate the signal from the noise. The second is the sheer number of things you are now tuning. You are not just picking a model. You are shaping CLAUDE.md, skills, hooks, subagent routing, effort levels, tool surfaces, and permission modes, and each of those interacts with the others. Every one is a hypothesis, and without a measurement harness you are optimizing blind and calling it intuition. The third is the quiet one, regression. Tightening a rule to fix one behavior silently breaks another, and you find out three sessions later when something that used to work doesn't. Anthropic's guidance is direct about the fix: test both the cases where a behavior should occur and the cases where it shouldn't, or you will lopsidedly optimize one at the expense of the other. [^1]

The thesis is that as agents write more of the code, the human's center of gravity shifts toward reviewer and verifier, and verification becomes the load-bearing skill (see Ch. 11). Evals are verification industrialized. Instead of checking one output by hand, you check a distribution of outputs against a known-good standard, repeatably, every time you touch the system.

This is not academia. The most ambitious internal coding experiment Anthropic has written up is a C compiler built almost entirely by agents, and its whole story belongs to Ch. 11. The part that belongs here is the eval lesson, which the author distilled to one operational line: "it's important that the task verifier is nearly perfect, otherwise Claude will solve the wrong problem." [^2] And most of the human's effort went somewhere telling: "Most of my effort went into designing the environment around Claude -- the tests, the environment, the feedback -- so that it could orient itself without me," which in practice meant finding high-quality compiler test suites, writing verifiers and build scripts, watching for the mistakes Claude was making, and designing new tests as each failure surfaced. [^3] The agents wrote the compiler. The human wrote, and obsessively maintained, the evals. That ratio is the lesson.

---

## 12.2 SWE-bench Verified: read the trend, not the number

SWE-bench Verified is the benchmark you will see quoted in every coding-model launch. Knowing what it is, and what it stopped being, is table stakes.

### What it measures

The original SWE-bench is 2,294 task instances crawled from pull requests and issues across 12 popular Python repositories. The agent is handed the issue text and a checkout of the repo from just before the fix, and must edit the codebase to resolve the issue. [^4] Scoring is end-to-end and objective: the patch is applied and the repo's own Fail-to-Pass tests are run, the tests that failed before the human PR and passed after it, which are "the primary signal for evaluation." [^5]

SWE-bench Verified is a 500-problem subset that OpenAI released, in collaboration with the SWE-bench authors, after a human-annotation campaign with professional software developers screened every sample for well-specified issue descriptions and appropriately scoped tests. They filtered out samples flagged by any one of three annotators for an underspecified problem statement or unfair Fail-to-Pass tests. [^6] The "Verified" qualifier means a human confirmed the task is genuinely solvable and its grader is fair, which is exactly what makes it the cleaner instrument.

This is why it is a good benchmark in principle. It exercises the full loop on real code rather than toy problems: understand the issue, navigate the codebase, write the fix, verify against tests, with a binary, reproducible grader.

### The trend (the part worth teaching)

Anthropic's own model announcements trace the curve cleanly. Claude 3.5 Sonnet reported 49% on SWE-bench Verified in January 2025, beating the prior state of the art of 45%. [^7] By mid-2025 Claude Sonnet 4 reported 72.7%, [^8] Sonnet 4.5 reported 77.2%, [^9] and by early 2026 Opus 4.6 reported 80.8%. [^10] That is the shape: a steep climb across 2025 followed by a cluster near the top.

> **Do not memorize or quote a specific model's SWE-bench score as a fixed fact.** These numbers drift, and sources disagree because different scaffolds, harnesses, and reporting dates produce materially different numbers for the same model. A stale figure quoted in a cohort session ages within a fortnight. Teach the shape of the curve, steep rise then saturation, and point people at the live leaderboard and each vendor's own announcement.

### Why saturation and contamination mean you can't lean on it

Two failure modes come for every benchmark eventually, and both arrived for SWE-bench Verified.

The first is saturation. When the leading models sit within a couple of points of each other near 80%, the benchmark has stopped discriminating between them. The resolution collapses exactly where you most want it. The second is contamination, and the cleanest evidence is the gap to a harder, contamination-resistant cousin. Scale AI's SWE-bench Pro was built specifically to resist training-data leakage, drawing on public codebases "governed by strong copyleft licenses (e.g., GPL), whose 'viral' nature and legal complexities make them highly likely to be excluded from training data" plus "completely private, commercial codebases." [^11] It is 1,865 instances across 41 repositories, partitioned into public, held-out, and commercial sets, spanning Python, Go, JavaScript, and TypeScript, and its tasks are long-horizon, "patches across multiple files and substantial code modifications" that may take a professional hours to days. [^12] [^13]

The tell is the drop. On SWE-bench Pro, "the best-performing models, OpenAI GPT-5 and Claude Opus 4.1, score only 23.3% and 23.1% respectively," against the 70%-plus those same lineages post on Verified. [^13] A forty-to-fifty-point fall on structurally similar work is the contamination and saturation tax made visible. A harder, leak-resistant benchmark restores discrimination, which is precisely the point of building one.

### The senior takeaway

Public benchmarks answer one question, which model is roughly more capable at generic GitHub-issue-shaped tasks, and they answer it noisily. They tell you nothing about whether your agent, on your codebase, with your CLAUDE.md and your skills, got better when you changed something. For that, you build your own.

---

## 12.3 The infrastructure-noise caveat

Before you trust any agentic-coding leaderboard, sit with one fact: the harness the eval ran on can move the score more than the model did.

Anthropic put numbers on it. Running Terminal-Bench 2.0 across six resource configurations, from strict per-task enforcement to fully uncapped, "at uncapped resources, the total lift over 1x is +6 percentage points (p < 0.01)," driven mostly by spurious infrastructure errors, which dropped monotonically "from 5.8% at strict enforcement to 0.5% when uncapped." [^14] The mechanism is mundane and brutal. Container runtimes enforce resources with two parameters, a guaranteed allocation and a hard limit at which the container is killed, and when those are set with no margin, a transient memory spike kills the container outright, a "failure" that has nothing to do with the model's reasoning. The Terminal-Bench leaderboard used a more lenient sandbox that allowed temporary overallocation without terminating, so the same task scored differently on a stricter setup. [^15]

The effect is benchmark-dependent. On the lighter SWE-bench, a crossover test varying RAM up to 5x baseline across 227 problems with 10 samples each found scores "only 1.54 percentage points higher at 5x than 1x." [^15] Heavier tasks are more sensitive: on jobs that install big data-science libraries, "the pod runs out of memory during installation, before the agent writes a single line of solution code," when pandas, networkx, and scikit-learn exhaust the cap. [^15] And there is a confounder you cannot configure away: "pass rates fluctuate with time of day, likely because API latency varies with traffic patterns and incidents." A task that times out at peak load fails for reasons orthogonal to capability. [^15]

### The operational rules this gives you

- **Treat sub-three-point leaderboard gaps as noise** unless the configs are documented and matched. Anthropic's framing: "leaderboard differences below 3 percentage points deserve skepticism until the eval configuration is documented and matched." [^15]
- **Specify resources as two numbers, not one.** Set a guaranteed allocation and a separate, higher hard-kill threshold per task, calibrated so floor and ceiling scores fall within noise of each other. At a 3x ceiling, Anthropic "cut infra error rates by roughly two-thirds (5.8% to 2.1%, p < 0.001) while keeping the score lift modest and well within noise (p = 0.40)," meaning the headroom fixed reliability without inflating capability. [^15]
- **Document your harness config as a first-class variable**, with the rigor you would apply to prompt format, temperature, or effort. Two people running "the same eval" with different container limits, concurrency, or timeouts are running different experiments.
- **Run comparison runs at a consistent time, with consistent concurrency.** If you A/B two prompts and run version A at 2pm under load and version B at 2am, you have confounded the result with API latency.

This caveat is why the rest of the chapter exists. If a six-point swing can come from a VM config, then the only evals you can fully trust are the ones whose harness you control end to end.

---

## 12.4 Build your own: a 20-50 task eval

Anthropic's guidance is refreshingly concrete: "20-50 simple tasks drawn from real failures is a great start." [^16] Early-stage agents show clear performance changes at small sample sizes, so you do not need a thousand-task dataset to catch a real regression. Start small, start real, grow the suite as new failure modes surface.

### Where the tasks come from

Don't invent tasks. Harvest them. Reality is a better task designer than you are, and the highest-fidelity eval tasks come from things that already broke.

- **Your manual pre-release checks.** The behaviors you already verify by hand before you trust a change are each a latent eval task. [^16]
- **The bug tracker and support queue.** If you are in production, real user-reported failures are eval gold. They are representative by construction.
- **Transcripts where the agent went wrong.** Every time you watch Claude do something dumb in a real session, capture the starting state, the task, and the correct outcome.

For a personal or team agentic-coding setup this stays concrete. Keep a directory of small frozen repos or fixtures, each paired with a task spec and an expected result. When the agent fails a real task on your actual repo, minimize it into a reproducible fixture and add it to the suite. Over time the suite becomes the accumulated institutional memory of every way this agent has gotten something wrong, which is exactly the memory worth keeping.

### What makes a task good

Four properties separate a useful eval task from a misleading one. [^16]

1. **An unambiguous spec.** The task must be passable by an agent that simply follows instructions correctly. If the spec is vague, a fail tells you nothing about which one broke down, the agent or the ambiguity. Anthropic's red flag: "a 0% pass rate across many trials (i.e. 0% pass@100) is most often a signal of a broken task, not an incapable agent." [^16] If nothing ever passes, suspect your task or grader before you blame the model.

2. **A reference solution.** "For each task, it's useful to create a reference solution: a known working output that passes all graders." [^16] Writing it yourself does double duty: it proves the task is solvable, and it verifies your graders are configured correctly. A grader that fails your own reference solution is broken.

3. **Balanced coverage.** Include the negative cases. If you test "the agent should run the suite before declaring done," also test "the agent should not run the suite when only docs changed." Optimizing the positive case alone produces an agent that does the behavior even when it shouldn't. [^16]

4. **An isolated environment.** Each trial starts from a clean state. "Each trial should be 'isolated' by starting from a clean environment," because unnecessary shared state between runs can cause correlated failures that inflate or deflate your results. [^16] In practice: fresh git checkout (or a `git worktree`/`git stash` reset), fresh temp dir, and a `--bare` Claude Code invocation so local config and memory don't leak in. Bare mode "reduce[s] startup time by skipping auto-discovery of hooks, skills, plugins, MCP servers, auto memory, and CLAUDE.md," which is the property you want: the environment becomes the experimental variable instead of an accident of your dotfiles, and the docs call it "the recommended mode for scripted and SDK calls." [^17]

```bash
# One isolated eval case, headless, JSON output, reproducible config.
# --bare skips auto-discovery of CLAUDE.md / skills / hooks / MCP / auto memory,
# so the environment is the experimental variable, not your local setup.
claude -p "$(cat tasks/fix-null-deref/prompt.md)" \
  --bare \
  --permission-mode acceptEdits \
  --allowedTools "Read,Edit,Bash" \
  --output-format json \
  > runs/fix-null-deref/$(date +%s).json
```

The flags here are documented on the headless page: `-p`/`--print` for non-interactive use, `--bare` for reproducible startup, `--permission-mode acceptEdits` to let Claude write files (and auto-approve `mkdir`/`touch`/`mv`/`cp`) without prompting, `--allowedTools` to pre-authorize the rest, and `--output-format json` to get a structured payload that includes `total_cost_usd` so you can track spend per run. [^18]

> Flags evolve. The headless page notes `--bare` "will become the default for `-p` in a future release," and the changelog moves fast. Confirm against `claude --help` and the changelog before scripting a harness. The concept, headless plus reproducible config plus clean checkout per case, is stable; the exact flag names may drift.

### The harness shape

A minimal personal eval harness is a loop, not a framework, and the whole thing fits in your head.

1. For each task, reset to a clean environment (fresh checkout or temp dir).
2. Invoke the agent headless on the task prompt, capturing the full transcript (`--output-format json` or `stream-json`).
3. Run the grader(s) on the result.
4. Record pass/fail plus the transcript path.
5. Repeat each task k times, because you need multiple samples (see pass@k / pass^k below).
6. Aggregate and diff against the baseline run.

You do not need a vendor framework to start. A bash or python loop over a `tasks/` directory, writing JSONL results, is enough to detect regressions, and the docs show the linter-style pattern directly: `git diff main | claude -p "..."` wrapped in a `package.json` script. [^18] Reach for the Claude Agent SDK (CLI, Python, or TypeScript) when you want programmatic control over the agent loop, sessions, and grading in one process rather than shelling out. The SDK "gives you the same tools, agent loop, and context management that power Claude Code." [^17] It supports subagents by default, connects to tools over MCP, and "automatically summarizes previous messages when the context limit approaches." [^19]

---

## 12.5 Grader types: code, model, human

Every eval comes down to a grader, a function that maps an agent's output to pass/fail or a score. There are three families, and mature suites compose all three. [^16]

| Grader | What it is | Strengths | Weaknesses | Use when |
|--------|-----------|-----------|------------|----------|
| **Code-based** | String/regex match, run the test suite, static analysis, exit-code or outcome check | Fast, cheap, objective, reproducible, easy to debug | Brittle to valid variations the designer didn't anticipate | The answer is verifiable: tests pass, file exists, output parses, exit code 0 |
| **Model-based (LLM-as-judge)** | A second model scores the output against a structured rubric | Flexible, scalable, captures nuance and open-ended quality | Non-deterministic, more expensive, needs calibration | Quality is subjective: tone, clarity, "is this a reasonable approach," does the explanation hold up |
| **Human** | Expert or crowdsourced review | Gold-standard quality, matches expert judgment | Expensive, slow, needs expert access | Calibrating the other two; high-stakes final judgment; novel domains |

[^16]

### Code-based: the default for coding agents

For coding evals, code-based grading is your workhorse, because code has a natural oracle: tests. The single most reliable grader for a "fix this bug" task is "the hidden test suite passes," which is exactly how SWE-bench works, and it is why coding is one of the most eval-friendly domains there is. The verification function is more objective than for, say, prose, where two readers can disagree about whether something is good.

Concrete code-based graders for an agentic-coding suite:

- **Tests as oracle.** Run the suite; pass if and only if green. The gold standard.
- **Static checks.** Lint clean, typecheck clean, no `TODO`/`FIXME` left behind, no stray debug `print`.
- **Outcome assertions.** The expected file was created or modified; a specific function signature exists; the build succeeds; a CLI produces the expected output.
- **Negative assertions.** The agent did not touch files outside scope, did not introduce a banned dependency, and did not weaken a test to make it pass, the critical reward-hacking guard.

> **The reward-hacking trap.** Code-based graders are gameable. If "tests pass" is the whole grader, a determined agent can make tests pass by editing the tests or hard-coding the expected output. This is the C-compiler lesson in miniature: an imperfect verifier invites the agent to solve the wrong problem. [^3] Defenses: keep the grading tests out of the agent's workspace, assert the agent did not modify the test files, and hold out a test set the agent never sees. Anthropic's published reward-hacking evaluations for Claude follow this shape, scoring models against held-out tests and a classifier that flags when a model gamed the visible ones, and report both numbers in the model system cards. [^20]

### Model-based: making the subjective binary

When the thing you care about resists a code check, "is this a clean refactor," "is the PR description accurate," "did the agent explain its reasoning sensibly," you reach for a second model as judge against a rubric. The Agent SDK names this directly as a verification strategy alongside rules-based and visual feedback: "LLM as a judge." [^21] The discipline that keeps it honest is forcing a binary verdict. Structure the judge prompt so it returns only `pass`/`fail`, or a numeric score against weighted criteria, rather than free-form opinion.

A rubric is a structured set of criteria, each with a weight and a checker, and the checker can itself be a regex, a function call, a structural assertion, or an LLM-judge sub-prompt. The tradeoff is real, and worth stating plainly to a cohort: an LLM judge is a second model call, so it is slower, costlier, and variable between runs. The mitigations are mechanical. Pin the judge model and effort. Constrain the verdict to a schema with `--output-format json` and `--json-schema` so you parse a field rather than prose. [^18] And run the judge a few times per case if its variance is material.

The pattern generalizes into a runtime loop, which is where evals and self-correcting agents meet. The SDK's core agent loop is "gather context -> take action -> verify work -> repeat," and the verify step is where the rubric lives. [^21] Generate output, score it against a rubric you wrote, return if it clears the threshold and otherwise feed the rubric's verdict back as feedback and retry. That is the eval loop turned inward, and it is the bridge between "I have an eval suite" and "my agent grades itself before it hands me anything."

You must calibrate model-based graders against humans. "LLM-as-judge graders should be closely calibrated with human experts to gain confidence that there is little divergence between the human grading and model grading." [^16] A judge that disagrees with your expert judgment is worse than no judge, because it hands you a confident number pointing the wrong way. Spot-check a sample of its verdicts by hand and tune the rubric until it agrees with you on the cases you care about.

### Human: the calibration anchor

Human grading is the gold standard you can't afford at scale, so spend it where it earns its cost: calibrating your code-based and model-based graders, and adjudicating high-stakes or novel cases. The point of automated graders is to amortize human judgment, not replace it. Pull a sample periodically and grade it yourself, and when your automated graders diverge from your own verdict, trust the verdict and fix the graders.

---

## 12.6 pass@k vs pass^k: capability vs consistency

A single run tells you almost nothing about a non-deterministic agent. You run each task k times, and how you aggregate those runs determines which question you are answering. The two standard aggregations mean opposite things and diverge fast.

### Definitions and formulas

**pass@k** is the probability that at least one of k attempts succeeds. It is the original metric from the Codex/HumanEval paper, where "k code samples are generated per problem, a problem is considered solved if any sample passes the unit tests, and the total fraction of problems solved is reported." [^22] It measures the capability ceiling: given enough chances, can the agent ever do this? Because computing it from a single estimated success rate "can have high variance," the paper estimates it by generating n samples (it uses n=200, k<=100), counting the c that pass, and applying an unbiased estimator:

```
pass@k := E_problems [ 1 - C(n - c, k) / C(n, k) ]
```

where n is the number of samples generated per task (n >= k), and c is the number of those samples that pass the unit tests (c <= n). [^23] The paper notes that estimating pass@k instead as `1 - (1 - p)^k` from an empirical per-sample rate p "results in a consistent underestimate," which is why the combinatorial estimator exists. [^23]

**pass^k** ("pass-power-k") is a practitioner inversion of the same idea: the probability that all k attempts succeed. With a per-trial success rate p, it is simply `p^k`. It measures consistency, can the agent do this every time? At k=1 the two metrics are identical. As k grows they pull apart violently, pass@k climbing toward 100% while pass^k collapses toward 0%. [^24]

### The worked example that makes it click

Take an agent with a 70% per-attempt success rate and k=3:

- **pass@3 is about 97%.** Across three tries it almost certainly succeeds at least once.
- **pass^3 is 0.70^3 = 34.3%.** Across three consecutive tries it succeeds on all three only about a third of the time.

Same agent, same 70%. One number says 97%, the other says 34%. The number you report is a choice about which question you are answering, and reporting the wrong one is how teams ship agents that dazzle in the demo and fall over in production.

### Which one you actually want

- **pass@k for human-in-the-loop work.** If a human reviews the output and one good answer out of k is enough, interactive coding where you regenerate when the first attempt misses, or a research question where any correct path suffices, pass@k is the honest metric.
- **pass^k for autonomous pipelines.** If the output is consumed directly with no human gate, or the agent must chain N steps that all have to succeed, consistency is everything. A 97% pass@3 agent that is only 34% pass^3 will blow roughly two of every three multi-step autonomous runs.

For production agents, consistency rather than best-case capability is the optimization target, because it reflects the user-facing reality of an automated system. When you are deciding whether to let an agent run unsupervised (auto mode, run-to-condition goals, headless CI pipelines), pass^k over your real-failure suite is the number that should gate the decision.

### A practical note on k

You cannot measure pass^k honestly from one sample per task. Take enough samples to estimate the per-trial rate, then compute both metrics from it. Five to ten samples per task is a reasonable start for a 20-50 task suite; more tightens the estimate at the cost of API spend and wall-clock. And recall section 12.3: run those samples under matched conditions, same time window, same concurrency, same resource config, or infrastructure noise will masquerade as inconsistency and send you chasing a flaky agent that was never flaky.

---

## 12.7 Eval-driven development for agent workflows

The highest-leverage practice in this chapter is also the most counterintuitive: write the eval before the capability exists. Anthropic's framing is direct: "Build evals to define planned capabilities before agents can fulfill them, then iterate until the agent performs well." [^16]

This is TDD's discipline (Ch. 11) lifted from the level of the code to the level of the agent system. Instead of "write a failing test, make it pass," it is "write a failing eval, then tune the agent, CLAUDE.md, skills, hooks, model, effort, subagent routing, until the eval is green." The eval is the executable specification of what you want your agent to be able to do.

### The EDD loop for an agentic setup

1. **Define the target capability as an eval.** "When asked to fix a failing test, the agent should locate the cause, fix the source not the test, and leave the suite green." Encode it: a fixture repo with a known bug, a code-based grader asserting the tests pass and the test file was not modified, plus a negative case where the right move is to fix the test.
2. **Run the baseline.** Establish where the current config lands on pass@k and pass^k.
3. **Change one thing.** Add a CLAUDE.md rule, write a skill, install a hook, bump effort, reroute the model. One hypothesis at a time, or you cannot attribute the result.
4. **Re-run and diff against baseline.** Did pass^k rise on the target case without regressing the negative case or anything else in the suite?
5. **Read the transcripts** (below). Confirm the reason it passed is the reason you intended, not luck or reward hacking.
6. **Keep it or revert.** Then repeat.

This is how you make the high-dimensional tuning of an agentic setup tractable. Memory, rules, skills, hooks, permission modes, model routing, effort, each lever stops being a vibe you argue about and becomes a hypothesis you can falsify against your suite.

### The non-negotiable: read the transcripts

You can have a green dashboard and a broken eval at the same time, and you will not know it until you look. Anthropic is blunt: "You won't know if your graders are working well unless you read the transcripts and grades from many trials." [^16] Reading transcripts is how you catch the three things a number alone hides.

- **False passes.** The grader said pass, but the agent cheated, edited the test, hard-coded the answer, modified out-of-scope files. The reward-hacking failure mode. [^3]
- **False fails.** The agent found a valid alternative approach your rigid grader did not anticipate. Anthropic warns specifically against scoring the path instead of the result: "There is a common instinct to check that agents followed very specific steps like a sequence of tool calls in the right order. We've found this approach too rigid and results in overly brittle tests... it's often better to grade what the agent produced, not the path it took." [^16]
- **Broken tasks.** The 0% pass@100 signal that means your spec or environment is wrong, not the agent. [^16]

Headless JSON or stream-json output is what makes this scalable. Every run produces a machine-readable transcript you can store, diff, and sample for manual review. The grader hands you the number; the transcript tells you whether to believe it.

---

## 12.8 A senior's eval checklist

| Decision | Guidance |
|----------|----------|
| Should I trust this leaderboard number? | Only as a trend. Ignore gaps under ~3 points unless configs are documented and matched. [^15] |
| How many tasks to start? | 20-50 from real failures: bug tracker, support queue, your manual checks. [^16] |
| Where do tasks come from? | Things that already broke. Minimize a real failure into a reproducible fixture. [^16] |
| Which grader? | Code-based by default (tests as oracle); model-based for subjective quality; human to calibrate both. [^16] |
| 0% pass@100? | Suspect a broken task or grader before an incapable agent. [^16] |
| pass@k or pass^k? | pass@k for human-in-the-loop; pass^k for autonomous pipelines and multi-step runs. [^25] |
| Isolation? | Clean environment per trial: fresh checkout, `--bare`, no shared state. [^26] |
| Grading philosophy? | Grade the outcome, not the path. Rigid step-checking is brittle. [^16] |
| Guard against reward hacking? | Held-out tests, assert no test-file edits, a classifier for gaming. [^27] |
| The one habit? | Read the transcripts. The number is not the truth until you have watched the runs. [^16] |

Here is the insight that compounds. In the agentic era your eval suite is the asset and the agent is the commodity. The models keep getting better on their own, on someone else's roadmap, on someone else's schedule. What is durable is the verification harness that tells you whether your system, on your problems, actually improved. That is the senior engineer's edge. Not knowing which model tops the leaderboard this week, but holding the apparatus to find out what works for the codebase sitting in front of you.

---

## Sources

- Anthropic -- Demystifying evals for AI agents: `https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents`
- Anthropic -- Quantifying infrastructure noise in agentic coding evaluations: `https://www.anthropic.com/engineering/infrastructure-noise`
- Anthropic -- Building a C compiler with a team of parallel Claudes: `https://www.anthropic.com/engineering/building-c-compiler`
- Anthropic -- Building agents with the Claude Agent SDK: `https://claude.com/blog/building-agents-with-the-claude-agent-sdk`
- Anthropic -- Run Claude Code programmatically (headless): `https://code.claude.com/docs/en/headless`
- Anthropic -- Raising the bar on SWE-bench Verified with Claude 3.5 Sonnet: `https://www.anthropic.com/research/swe-bench-sonnet`
- Anthropic -- Introducing Claude 4 (SWE-bench Verified figures): `https://www.anthropic.com/news/claude-4`
- Anthropic -- Introducing Claude Sonnet 4.5: `https://www.anthropic.com/news/claude-sonnet-4-5`
- Anthropic -- Claude Opus 4.6: `https://www.anthropic.com/news/claude-opus-4-6`
- Anthropic -- Claude Code changelog: `https://code.claude.com/docs/en/changelog.md`
- Anthropic -- Claude models overview: `https://platform.claude.com/docs/en/about-claude/models/overview`
- OpenAI -- Introducing SWE-bench Verified: `https://openai.com/index/introducing-swe-bench-verified/`
- Jimenez et al. -- SWE-bench: Can Language Models Resolve Real-World GitHub Issues? (arXiv:2310.06770): `https://arxiv.org/abs/2310.06770`
- SWE-bench project site (scoring / Fail-to-Pass): `https://www.swebench.com/`
- Xu et al. (Scale AI) -- SWE-Bench Pro: Can AI Agents Solve Long-Horizon Software Engineering Tasks? (arXiv:2509.16941): `https://arxiv.org/abs/2509.16941`
- Scale AI -- SWE-Bench Pro: Raising the Bar for Agentic Coding: `https://scale.com/blog/swe-bench-pro`
- Chen et al. -- Evaluating Large Language Models Trained on Code (Codex/HumanEval, pass@k definition; arXiv:2107.03374): `https://arxiv.org/abs/2107.03374`
- Philipp Schmid -- pass@k vs pass^k: understanding agent reliability (pass^k framing): `https://www.philschmid.de/agents-pass-at-k-pass-power-k`

[^1]: web | 2026-06-18 | Anthropic, "Demystifying evals for AI agents," anthropic.com/engineering/demystifying-evals-for-ai-agents
[^2]: web | 2026-06-18 | Anthropic, "Building a C compiler with a team of parallel Claudes," anthropic.com/engineering/building-c-compiler
[^3]: web | 2026-06-18 | Anthropic, building-c-compiler
[^4]: web | 2026-06-18 | Jimenez et al., "SWE-bench: Can Language Models Resolve Real-World GitHub Issues?", arXiv:2310.06770, abstract
[^5]: web | 2026-06-18 | [swebench.com/original.html](https://swebench.com/original.html), "SWE-bench" overview
[^6]: web | 2026-06-18 | OpenAI, "Introducing SWE-bench Verified," openai.com/index/introducing-swe-bench-verified
[^7]: web | 2026-06-18 | Anthropic, "Raising the bar on SWE-bench Verified with Claude 3.5 Sonnet," anthropic.com/research/swe-bench-sonnet, 2025-01-06
[^8]: web | 2026-06-18 | Anthropic, "Introducing Claude 4," anthropic.com/news/claude-4
[^9]: web | 2026-06-18 | Anthropic, "Introducing Claude Sonnet 4.5," anthropic.com/news/claude-sonnet-4-5
[^10]: web | 2026-06-18 | Anthropic, "Claude Opus 4.6," anthropic.com/news/claude-opus-4-6
[^11]: web | 2026-06-18 | Scale AI, "SWE-Bench Pro: Raising the Bar for Agentic Coding," scale.com/blog/swe-bench-pro
[^12]: web | 2026-06-18 | Xu et al., "SWE-Bench Pro: Can AI Agents Solve Long-Horizon Software Engineering Tasks?", arXiv:2509.16941, abstract
[^13]: web | 2026-06-18 | Scale AI, scale.com/blog/swe-bench-pro
[^14]: web | 2026-06-18 | Anthropic, "Quantifying infrastructure noise in agentic coding evaluations," anthropic.com/engineering/infrastructure-noise
[^15]: web | 2026-06-18 | Anthropic, infrastructure-noise
[^16]: web | 2026-06-18 | Anthropic, demystifying-evals-for-ai-agents
[^17]: web | 2026-06-18 | Anthropic, "Run Claude Code programmatically," code.claude.com/docs/en/headless
[^18]: web | 2026-06-18 | Anthropic, code.claude.com/docs/en/headless
[^19]: web | 2026-06-18 | Anthropic, "Building agents with the Claude Agent SDK," claude.com/blog/building-agents-with-the-claude-agent-sdk
[^20]: web | 2026-06-18 | Anthropic, Claude system cards (reward-hacking evaluations), anthropic.com; verify the latest system card before quoting figures
[^21]: web | 2026-06-18 | Anthropic, claude.com/blog/building-agents-with-the-claude-agent-sdk
[^22]: web | 2026-06-18 | Chen et al., "Evaluating Large Language Models Trained on Code," arXiv:2107.03374, sec. 2.1
[^23]: web | 2026-06-18 | Chen et al., arXiv:2107.03374, sec. 2.1
[^24]: web | 2026-06-18 | practitioner framing: Schmid, "pass@k vs pass^k: understanding agent reliability," philschmid.de/agents-pass-at-k-pass-power-k
[^25]: web | 2026-06-18 | Chen et al., arXiv:2107.03374
[^26]: web | 2026-06-18 | Anthropic, demystifying-evals-for-ai-agents; code.claude.com/docs/en/headless
[^27]: web | 2026-06-18 | Anthropic, building-c-compiler; Claude system cards
