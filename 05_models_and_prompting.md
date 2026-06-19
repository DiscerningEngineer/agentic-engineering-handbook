# Chapter 05 -- Models, Effort, and Prompting

**TL;DR.** As of mid-2026 the Claude Code rotation is Fable 5 (frontier, long-horizon), Opus 4.8 (the default capable tier), Sonnet 4.6 (daily driver), and Haiku 4.5 (fast, cheap subagent workhorse). You pick a model with `/model` and a handful of aliases. Reasoning depth and token spend ride on a single dial called effort -- `low` through `max` -- which replaced hand-set thinking budgets when adaptive thinking arrived on the 4.6 generation. The old `thinking: {type: "enabled", budget_tokens: N}` is now a 400 error on Opus 4.8 and 4.7. The `ultrathink` keyword still works, but only as a per-turn nudge that injects an in-context instruction, leaving the API effort untouched. The deepest shift is in how you prompt: these models are proactive enough that yesterday's anti-laziness coaching now causes over-triggering, over-engineering, and over-delegation, so modern prompting dials down, gives the why, and grounds claims in tool results. This is the most version-sensitive chapter in the book. Names, defaults, and aliases churn near-daily; verify against the live [models overview](https://platform.claude.com/docs/en/about-claude/models/overview) and the [Claude Code changelog](https://code.claude.com/docs/en/changelog.md) before you lean on any specific value.

> **Reading note.** Dollar prices and benchmark scores drift faster than a chapter can track, so they are left out on purpose. Consult the [pricing page](https://platform.claude.com/docs/en/about-claude/pricing) and the [models overview](https://platform.claude.com/docs/en/about-claude/models/overview) for live numbers. Version-pinned claims (minimum Claude Code versions, alias resolutions) are accurate as of 2026-06-18 and flagged where they are most likely to move.

## 5.1 The mid-2026 model lineup

Four models earn a place in a senior engineer's daily rotation, with a research-preview tier sitting above them that most readers will never touch. The capability axis that actually matters is how long a task the model can hold in its head without losing the thread: a single prompt, a single sitting, or a stretch of days. The cost and latency you pay per unit of that capability is the second axis, and it bends every routing decision you make.

### The four working models

| Model | API ID / alias | Tier role | Context window | Max output | Adaptive thinking | Notes |
|-------|----------------|-----------|----------------|------------|-------------------|-------|
| Claude Fable 5 | `claude-fable-5` / `fable` | Frontier; hardest, longest-running, most ambiguous work | 1M | 128k | Yes (always on) | GA June 9 2026. Opt-in, never the default. Routes flagged cyber/bio requests to Opus 4.8. ^[source: web \| 2026-06-18 \| platform.claude.com models overview, "Claude Fable 5 and Claude Mythos 5" table (GA date, specs); code.claude.com model-config, "Automatic model fallback" (routing to Opus 4.8)] |
| Claude Opus 4.8 | `claude-opus-4-8` / `opus` | Default capable Opus tier; complex reasoning + agentic coding | 1M | 128k | Yes (only thinking mode) | What `opus` resolves to on the Anthropic API. ^[source: web \| 2026-06-18 \| platform.claude.com models overview, "Latest models comparison" table] |
| Claude Sonnet 4.6 | `claude-sonnet-4-6` / `sonnet` | Balanced daily driver | 1M | 64k | Yes | Best speed/intelligence balance. ^[source: web \| 2026-06-18 \| platform.claude.com models overview, "Latest models comparison" table] |
| Claude Haiku 4.5 | `claude-haiku-4-5` / `haiku` | Fastest; high-throughput subagents | 200k | 64k | No | Near-frontier intelligence at the lowest tier; built for fan-out workers. ^[source: web \| 2026-06-18 \| platform.claude.com models overview, "Latest models comparison" table] |

Three facts in that table catch people over and over, so it is worth slowing down on each.

The first is that Haiku 4.5 carries a 200k window while every other current model carries 1M. Fan heavy work out to a swarm of Haiku subagents and you cannot hand each one a million-token slice; the window simply is not there to fill. ^[source: web | 2026-06-18 | platform.claude.com models overview, "Latest models comparison" table]

The second is that Haiku 4.5 sits out adaptive thinking and the effort parameter entirely. Set effort against it and the control becomes a silent no-op. Nothing errors, nothing warns, the dial just spins free. ^[source: web | 2026-06-18 | code.claude.com model-config, "Adjust effort level" -- "Models not listed here do not support effort"]

The third is that Opus 4.8 has a single thinking mode and that mode is adaptive. The legacy `thinking: {type: "enabled", budget_tokens: N}` comes back as a 400 error. The full deprecation gradient lives in section 5.5. ^[source: web | 2026-06-18 | platform.claude.com adaptive-thinking, "Supported models"]

### Fable 5 and the Mythos research tier

Fable 5 is the frontier model, built for tasks larger than a single sitting: multi-day, goal-directed, ambiguous problems where the model investigates before it acts and verifies its own work more than its smaller siblings do. It is not the Claude Code default on any account type. You reach for it deliberately with `/model fable`. ^[source: web | 2026-06-18 | code.claude.com model-config, "Work with Fable 5"]

Fable 5 runs safety classifiers for offensive-cybersecurity and biology/life-sciences content, plus a third that guards against extracting its own summarized reasoning. When a request trips one, Claude Code falls back to the default Opus model on its own (Opus 4.8 on the Anthropic API and LLM-gateway deployments, Opus 4.7 on Claude Platform on AWS) and drops a notice into the transcript. Here is the part that surprises people: this can fire on the very first request of a session, before you have typed anything that feels unusual, because the first request carries your `CLAUDE.md`, your git status, and your directory names along with it. A security- or biology-adjacent repo can trip the classifier on workspace context alone. To find out whether your own customizations are the trigger, launch with `claude --safe-mode`, which disables `CLAUDE.md`, skills, MCP, and hooks while leaving git status and directory names in place. ^[source: web | 2026-06-18 | code.claude.com model-config, "Automatic model fallback" + "Check what triggered fallback"]

Above Fable sit Mythos 5 (`claude-mythos-5`) and the separate Mythos Preview (`claude-mythos-preview`), invitation-only research-preview models under Project Glasswing, the latter aimed at defensive cybersecurity. Access is not self-serve. Outside the program you will never see them in the picker, and they appear here only so you recognize the names when they surface in someone else's transcript. ^[source: web | 2026-06-18 | platform.claude.com models overview, "Claude Fable 5 and Claude Mythos 5" + Info callout]

### The legacy-but-available shelf

These still run, and you will meet them on locked-down providers (see section 5.3), but migrate off them when the door opens: Opus 4.7 (1M, adaptive-only, `xhigh` default), Opus 4.6 (1M, adaptive, supports `max` but not `xhigh`), Sonnet 4.5 (200k, manual thinking only), Opus 4.5 (200k, manual thinking, but it does accept effort -- `low`/`medium`/`high` only, no `xhigh` or `max`). ^[source: web | 2026-06-18 | platform.claude.com models overview, "Legacy models" table; platform.claude.com effort, "supported by" Note]

Deprecations move fast, so anchor yourself to the live page rather than to memory. One fixed point to orient from: Opus 4.1 (`claude-opus-4-1-20250805`) is deprecated and retires August 5, 2026, with Anthropic pointing migrators toward Opus 4.8. Quoting a deprecation date from memory is how you ship a wrong fact; pull it from [model deprecations](https://platform.claude.com/docs/en/about-claude/model-deprecations) every time. ^[source: web | 2026-06-18 | platform.claude.com models overview, "Legacy models" Warning callout]

### A tokenizer footgun worth knowing

Fable 5, Mythos 5, and Opus 4.7-and-later all run the tokenizer that arrived with Opus 4.7, and the same text produces roughly 30% more tokens on it than it did before. So "1M tokens" on Fable 5 holds a smaller body of actual text than "1M tokens" held on Opus 4.6. If your context budgets were sized from old measurements, they are now optimistic. Re-measure. ^[source: web | 2026-06-18 | platform.claude.com models overview, context-window tooltip: "the same text produces roughly 30% more tokens"]

### Model IDs are pinned snapshots, not evergreen pointers

Starting with the 4.6 generation, model IDs use a dateless format that is still a pinned snapshot. `claude-opus-4-8` is a fixed release, not a moving target. Pre-4.6 models used dated IDs like `claude-sonnet-4-5-20250929`, with the alias column acting as a convenience pointer alongside them. Aliases (`opus`, `sonnet`, and the rest) drift over time toward the recommended version for your provider, which is convenient right up until it is not. When reproducibility is the point -- CI, evals, anything you will re-run weeks from now -- pin the full model ID and sleep easier. ^[source: web | 2026-06-18 | platform.claude.com models overview, Note: "Every Claude model ID is a pinned snapshot"]

## 5.2 `/model`, aliases, and setting your default

### The aliases

Aliases let you pick a model without memorizing version numbers. The canonical set as of 2026-06-18: ^[source: web | 2026-06-18 | code.claude.com model-config, "Model aliases" table]

| Alias | Behavior |
|-------|----------|
| `default` | Clears any override and reverts to the recommended model for your account type. Not itself a model. |
| `best` | Fable 5 where your org has access, otherwise the latest Opus. |
| `fable` | Fable 5 -- hardest, longest-running tasks. |
| `opus` | Latest Opus (Opus 4.8 on the Anthropic API). |
| `sonnet` | Latest Sonnet (Sonnet 4.6) -- daily coding. |
| `haiku` | Fast, efficient Haiku -- simple tasks, subagents. |
| `opus[1m]` / `sonnet[1m]` | Same model, 1M-token window (see section 5.6). |
| `opusplan` | Opus during plan mode, then auto-switches to Sonnet for execution. |

For most senior workflows, `opusplan` is the alias that pays for itself. It gives you Opus-grade reasoning while you are planning and Sonnet-grade economics through the execution phase, which is usually the larger and more expensive stretch of the work. The plan-mode Opus phase runs the same context window as the `opus` setting; on auto-1M tiers it picks up the 1M upgrade in plan mode too, or you force it everywhere with `opusplan[1m]`. If `availableModels` excludes Opus, `opusplan` quietly stays on Sonnet in plan mode rather than erroring out from under you. ^[source: web | 2026-06-18 | code.claude.com model-config, "opusplan model setting"]

> **Beyond plan-boundary switching:** if you want Claude to decide mid-task when to consult a stronger model rather than switching only at the plan/execute boundary, look at the advisor tool (`advisorModel` setting). It is the dynamic-consultation cousin of `opusplan`. ^[source: web | 2026-06-18 | code.claude.com model-config, "opusplan model setting" -> advisor tool]

### Setting and switching, in priority order

1. **In-session:** `/model <alias|name>` switches immediately; `/model` with no arg opens the picker. As of v2.1.153, picking with `Enter` saves it as your default for new sessions, and `s` switches for this session only. (In v2.1.144 through v2.1.152 the behavior was inverted, with `d` saving the default. Version-sensitive; check yours.) ^[source: web | 2026-06-18 | code.claude.com model-config, "Setting your model"]
2. **At startup:** `claude --model <alias|name>` (session-only).
3. **Env var:** `ANTHROPIC_MODEL=<alias|name>` (session-only).
4. **Settings file:** the `model` field, persistent across launches, though project and managed settings override it and reapply on the next launch.

One rule here is subtle and bites the unwary. Both `--model` and `ANTHROPIC_MODEL` apply only to the session you launch with them. To run different models in different terminals at the same time, launch each one with its own `--model` instead of switching with `/model`, which writes a shared default that every new session inherits. Resumed sessions (`--resume`, `--continue`, `/resume`) keep whatever model they had when they were saved, regardless of your current setting, so another terminal's `/model` choice cannot reach across and hijack a resume. ^[source: web | 2026-06-18 | code.claude.com model-config, "Setting your model"]

```bash
# Start on Opus, switch to Sonnet mid-session
claude --model opus
/model sonnet

# Pin a specific snapshot for a reproducible run
claude --model claude-opus-4-8

# Plan-then-build hybrid
/model opusplan
```

Confirm the active model anytime via `/status` (which also shows account info) or a configured status line. The current effort level shows next to the spinner, for example "with low effort". ^[source: web | 2026-06-18 | code.claude.com model-config, "Checking your current model" + "Set the effort level"]

### Fallback chains (availability is not content fallback)

Separate from the Fable-to-Opus content fallback in section 5.1, Claude Code supports availability fallback chains for the moments when your primary model is overloaded or hands back a non-retryable server error. Authentication, billing, rate-limit, request-size, and transport errors are left alone; none of them trigger a switch.

```bash
claude --fallback-model sonnet,haiku        # one session
```
```json
{ "fallbackModel": ["claude-sonnet-4-6", "claude-haiku-4-5"] }   // persistent
```

Chains cap at three models after dedup; the switch holds for the current turn only, then the primary gets another shot on your next message. `--fallback-model` beats the `fallbackModel` setting, and `"default"` expands to your default model inside a chain. ^[source: web | 2026-06-18 | code.claude.com model-config, "Fallback model chains"]

## 5.3 Provider-dependent alias resolution -- read this before you trust `opus`

The single biggest source of "why is my model different from yours" confusion comes down to one fact: aliases resolve to different versions depending on your provider and account type. The word `opus` does not mean the same thing on every desk in the room. ^[source: web | 2026-06-18 | code.claude.com model-config, "Model aliases" resolution paragraph]

| Surface | `opus` resolves to | `sonnet` resolves to |
|---------|--------------------|----------------------|
| Anthropic API | Opus 4.8 | Sonnet 4.6 |
| Claude Platform on AWS | Opus 4.7 | Sonnet 4.6 |
| Bedrock / Vertex / Foundry | Opus 4.6 | Sonnet 4.5 |

And the `default` model by account type: ^[source: web | 2026-06-18 | code.claude.com model-config, "default model setting"]

| Account type | Default model |
|--------------|---------------|
| Max, Team Premium, Enterprise pay-as-you-go, Anthropic API | Opus 4.8 |
| Claude Platform on AWS | Opus 4.7 |
| Pro, Team Standard, Enterprise subscription seats | Sonnet 4.6 |
| Bedrock / Vertex / Foundry | Sonnet 4.5 |

For a regulated-industry or enterprise cohort, this stops being trivia. It is directly in your path the moment your org self-hosts through Bedrock, Vertex, or Foundry for SOC2 or data-residency reasons, and the consequences stack up in a few specific ways.

On those providers, plain `opus` quietly hands you an older model than your teammate on the Anthropic API is running. Pin versions explicitly in your initial rollout through `ANTHROPIC_DEFAULT_OPUS_MODEL`, `ANTHROPIC_DEFAULT_SONNET_MODEL`, `ANTHROPIC_DEFAULT_HAIKU_MODEL`, and `ANTHROPIC_DEFAULT_FABLE_MODEL`, using the full provider-specific IDs. Leave it unpinned and the default can lag the newest release; on Bedrock and Vertex an unavailable default falls back to the previous version for that session, while on Foundry it surfaces as an outright error because Foundry has no equivalent startup check. ^[source: web | 2026-06-18 | code.claude.com model-config, "Pin models for third-party deployments"]

Mind the floor on Claude Code versions too: Opus 4.8 requires v2.1.154 or later, Fable 5 requires v2.1.170 or later, and older builds will not even show Fable 5 in the picker. Run `claude update` to climb the ladder. ^[source: web | 2026-06-18 | code.claude.com model-config, Note: "Opus 4.8 requires Claude Code v2.1.154 or later" + "Work with Fable 5" Note]

The deepest cut is the silent one. On third-party providers, Claude Code figures out whether a model supports effort and thinking by pattern-matching the model ID. Bedrock ARNs and custom deployment names frequently fail to match, which leaves effort and thinking quietly disabled while everything else looks fine. Declare the capabilities explicitly with the `..._SUPPORTED_CAPABILITIES` variables, for instance `effort,xhigh_effort,max_effort,thinking,adaptive_thinking,interleaved_thinking`, so the features you are paying for actually turn on. ^[source: web | 2026-06-18 | code.claude.com model-config, "Customize pinned model display and capabilities"]

The enterprise governance levers (`availableModels`, `enforceAvailableModels`, `modelOverrides`) live in the enterprise/admin chapter. The model-config piece worth carrying here is that an org can restrict the picker, and when a blocked `--model` or `ANTHROPIC_MODEL` gets substituted at startup, Claude Code warns you with both the requested and the substituted model named. ^[source: web | 2026-06-18 | code.claude.com model-config, "Restrict model selection"]

## 5.4 Effort -- the primary reasoning and cost control

Effort took over from fixed thinking budgets as the knob for trading capability against token spend. The reason it earns "primary" billing is that it touches every token in the response, not just the thinking. Lower effort buys fewer tool calls, terser preambles, and more operations folded together; higher effort buys more tool calls, plans laid out before action, and detailed summaries. That breadth is why it stays a real cost lever even when thinking is switched off entirely. ^[source: web | 2026-06-18 | platform.claude.com effort, "How effort works" + "Effort with tool use"]

### The levels

| Level | What it does | Reach for it when |
|-------|--------------|-------------------|
| `low` | Most efficient; significant token savings, some capability loss. May skip thinking entirely. | Short, scoped, latency-sensitive work that is not intelligence-sensitive; subagents; classification and lookup. |
| `medium` | Balanced; moderate savings. | Cost-sensitive agentic work that can trade a little intelligence -- the recommended explicit default for Sonnet 4.6 coding. |
| `high` | High capability. Equivalent to omitting the parameter. Almost always thinks. | Complex reasoning, difficult coding, general agentic tasks. Default on Fable 5, Opus 4.8, Opus 4.6, Sonnet 4.6. |
| `xhigh` | Extended capability for long-horizon work; always thinks deeply. | Long-running (30 min+) agentic and coding tasks; repeated tool calling, deep web and knowledge-base search. Default on Opus 4.7. |
| `max` | Absolute maximum; no constraint on thinking depth. | Genuinely frontier problems only -- diminishing returns and overthinking risk elsewhere. |

^[source: web | 2026-06-18 | platform.claude.com effort, "Effort levels" table + "When to adjust the effort parameter"]

### Which models support which levels

| Model | Supported effort levels | Default |
|-------|-------------------------|---------|
| Fable 5 | `low`, `medium`, `high`, `xhigh`, `max` | `high` |
| Opus 4.8 | `low`, `medium`, `high`, `xhigh`, `max` | `high` |
| Opus 4.7 | `low`, `medium`, `high`, `xhigh`, `max` | `xhigh` |
| Opus 4.6, Sonnet 4.6 | `low`, `medium`, `high`, `max` (no `xhigh`) | `high` |
| Opus 4.5 | `low`, `medium`, `high` (no `xhigh` or `max`; manual-thinking model) | -- |
| Haiku 4.5 | none -- effort unsupported | -- |

Ask for a level the active model does not support and Claude Code falls back to the highest supported level at or below your request, so `xhigh` runs as `high` on Opus 4.6. The scale is calibrated per model, which means the same level name does not map to the same underlying value as you move between models. A `high` on one is not a `high` on another. ^[source: web | 2026-06-18 | code.claude.com model-config, "Adjust effort level" table + "calibrated per model"; platform.claude.com effort, "supported by" Note]

### Per-model effort guidance, from the official docs

On Opus 4.8 and Opus 4.7 the advice is the same: start at `xhigh` for coding and agentic work, hold `high` as the minimum for most intelligence-sensitive workloads, and step down to `medium` or `low` only after your evals confirm quality holds. Reserve `max` for genuinely frontier problems, where on structured-output and less intelligence-sensitive tasks it can tip into overthinking. At `xhigh` or `max`, give the model room by setting a large `max_tokens`; 64k is a reasonable place to start and tune from. ^[source: web | 2026-06-18 | platform.claude.com effort, "Recommended effort levels for Claude Opus 4.7" + "...4.8"]

On Fable 5, effort is the primary intelligence, latency, and cost dial. Start at `high`, the default, push to `xhigh` for the most capability-sensitive work, and drop to `medium` or `low` for routine tasks. The detail that resets your instincts: lower effort on Fable 5 often exceeds `xhigh` on prior models, so you can run it cheaper than its predecessors and still come out ahead. Reduce effort if a task finishes correctly but drags. ^[source: web | 2026-06-18 | platform.claude.com effort, "Recommended effort levels for Claude Fable 5"]

On Sonnet 4.6 the docs nudge against the default. It ships at `high`, but the recommendation is to set `medium` explicitly as the practical default for most apps -- agentic coding, tool-heavy workflows -- to avoid the unexpected latency that `high` invites. Drop to `low` for chat and high-volume work; climb to `high` or `max` only when quality dominates speed and cost. ^[source: web | 2026-06-18 | platform.claude.com effort, "Recommended effort levels for Sonnet 4.6"]

### Setting effort in Claude Code

```text
/effort                 # interactive slider
/effort xhigh           # set directly
/effort auto            # reset to model default
```

You have other doors into the same setting: the arrow keys on the effort slider inside `/model`, the `--effort` flag for a session, the `CLAUDE_CODE_EFFORT_LEVEL` env var (highest precedence), `effortLevel` in settings (which accepts only `low`, `medium`, `high`, `xhigh`), and `effort` in skill or subagent frontmatter to override per-invocation. Precedence runs env var, then configured level, then model default; frontmatter overrides the session level but never the env var. The levels `low`, `medium`, `high`, and `xhigh` persist across sessions, while `max` is session-only unless you set it through the env var. One thing to watch: when you first switch to Fable 5, Opus 4.8, or Opus 4.7, Claude Code applies that model's own default effort even if you had set something else a moment ago, so re-run `/effort` after the switch to get back where you wanted to be. ^[source: web | 2026-06-18 | code.claude.com model-config, "Set the effort level" + "Adjust effort level"]

### `ultracode` -- not an effort level

`ultracode` shows up in the `/effort` menu, but it is not an API effort level. What it does is pair `xhigh` with standing permission for Claude Code to orchestrate dynamic multi-agent workflows on substantive tasks. It is session-only, set through `/effort` or `"ultracode": true` via `--settings`. To replicate the behavior through the API, see Anthropic's "build an orchestration mode" example. The trap is reading it as a sibling of `max`; it is a Claude Code setting, not a model dial. ^[source: web | 2026-06-18 | platform.claude.com effort, "Claude Code's ultracode mode" Note; code.claude.com model-config, "Choose an effort level" -- ultracode row]

## 5.5 Adaptive thinking -- what replaced budgets and what `ultrathink` really does

This is the section the field gets wrong most often, so precision earns its keep here.

### The model

On the 4.6 generation and later, thinking is adaptive. The model decides per step whether to think at all and how much, governed by the `effort` parameter and the complexity of the query in front of it. At `high`, `xhigh`, and `max` it almost always thinks; at lower levels it may skip thinking on a simple problem and answer directly. Adaptive thinking also turns on interleaved thinking automatically, which lets the model think between tool calls, and that single property is most of why it is so strong inside agentic loops. In Anthropic's internal evals, adaptive thinking reliably beats a fixed `budget_tokens` across many workloads, with the gap widest on bimodal and long-horizon agentic work. ^[source: web | 2026-06-18 | platform.claude.com adaptive-thinking, "How adaptive thinking works" + intro Tip]

API config (for SDK-level work; in Claude Code this is handled for you):
```python
client.messages.create(
    model="claude-opus-4-8",
    max_tokens=64000,
    thinking={"type": "adaptive"},
    output_config={"effort": "high"},   # the new home of "how hard to think"
    messages=[...],
)
```
Note `effort` lives under `output_config` and thinking under `thinking`. They are separate parameters that work together, and on Opus 4.8 thinking is off unless you set `thinking: {type: "adaptive"}` explicitly. ^[source: web | 2026-06-18 | platform.claude.com adaptive-thinking, "Adaptive thinking with the effort parameter" code sample + "Supported models"]

### What is deprecated, what is rejected, what survives

| Model | Manual `budget_tokens` status |
|-------|-------------------------------|
| Opus 4.8, Opus 4.7 | Rejected with a 400 error. Adaptive is the only mode. |
| Fable 5, Mythos 5 | Adaptive always on; `thinking: {type: "disabled"}` is rejected; thinking cannot be turned off. |
| Opus 4.6, Sonnet 4.6 | Still functional but deprecated -- slated for removal in a future model release. Migrate. |
| Sonnet 4.5, Opus 4.5, older | Adaptive unsupported; you must use `thinking: {type: "enabled", budget_tokens: N}`. |

^[source: web | 2026-06-18 | platform.claude.com adaptive-thinking, "Adaptive vs manual vs disabled thinking" table + "Supported models" Warning]

So the long-standing question -- is `budget_tokens` deprecated? -- resolves cleanly along a gradient: deprecated on 4.6, outright rejected on 4.7 and 4.8, and mandatory on everything before 4.6. Teach effort, leave budgets behind. There is exactly one escape hatch, and it is narrow: in Claude Code on Opus 4.6 and Sonnet 4.6 only, `CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING=1` reverts to the old fixed budget controlled by `MAX_THINKING_TOKENS`. It does nothing on Opus 4.7-and-later or Fable 5. ^[source: web | 2026-06-18 | code.claude.com model-config, "Adaptive reasoning and fixed thinking budgets"]

### `ultrathink` -- still works, and here is exactly what it does

Settle the long-running ambiguity. `ultrathink` is alive and not deprecated. In Claude Code, dropping `ultrathink` anywhere in your prompt requests deeper reasoning on that turn, because Claude Code recognizes the keyword and adds an in-context instruction. The effort level sent to the API stays exactly where it was. It is a per-turn nudge, a whisper in the model's ear, and it is neither a budget nor an effort override. The neighboring phrases -- "think", "think hard", "think more" -- carry no magic in Claude Code; they pass through as ordinary prompt text. ^[source: web | 2026-06-18 | code.claude.com model-config, "Use ultrathink for one-off deep reasoning"]

Two nuances are worth holding onto. The first is an API-level model wrinkle: when extended thinking is disabled, Opus 4.5 specifically is sensitive to the literal word "think", so prefer "consider", "evaluate", or "reason through". That is an older-model quirk and nothing the 4.6 generation worries about. ^[source: web | 2026-06-18 | platform.claude.com claude-prompting-best-practices, "Leverage thinking" Note: "Claude Opus 4.5 is particularly sensitive to the word think"] The second is that you can steer adaptive thinking per message through the API too: appending "Please think hard before responding." encourages thinking, and "Answer directly without deliberating." suppresses it. The steering is sensitive to wording, so if one phrasing falls flat, reach for a blunter variant. ^[source: web | 2026-06-18 | platform.claude.com adaptive-thinking, "Tuning thinking behavior"]

### Thinking display and cost -- the gotchas

A handful of cost and display behaviors trip teams who assume the obvious, so each one deserves naming.

You are billed for the full thinking tokens no matter what gets displayed. Summarization or omission changes latency and visibility, never cost. Read `usage.output_tokens_details.thinking_tokens` to see how much of the output went to reasoning; it is a read-only breakdown, while `output_tokens` stays the authoritative billing total. ^[source: web | 2026-06-18 | platform.claude.com adaptive-thinking, "Pricing" -- output_tokens_details.thinking_tokens]

The default display changed quietly on the frontier, and that quiet is the danger. Fable 5, Mythos 5, Opus 4.8, and Opus 4.7 now default `thinking.display` to `"omitted"`, leaving an empty `thinking` field and a faster time-to-first-text. Opus 4.6 defaulted to `"summarized"`. If your tooling counted on reading those summaries, set `display: "summarized"` explicitly or watch it find nothing there. In Claude Code, interactive Anthropic-API sessions get redacted thinking by default; set `showThinkingSummaries: true` to expand the full summaries, and `Ctrl+O` toggles the verbose view. ^[source: web | 2026-06-18 | platform.claude.com adaptive-thinking, "Controlling thinking display" + display-default Note; code.claude.com model-config, "Extended thinking"]

Cost control is two instruments played together: `max_tokens` is the hard ceiling and `effort` is the soft guidance. `max_tokens` caps total output across both thinking and text, and at `high` or `max` the model can run the whole tank dry. When you see `stop_reason: "max_tokens"`, your move is to raise `max_tokens` or lower effort. ^[source: web | 2026-06-18 | platform.claude.com adaptive-thinking, "Cost control"]

Fable 5 refuses reasoning extraction. Prompt it to echo or transcribe its internal reasoning into the response and you can trip the `reasoning_extraction` refusal category, which shows up as elevated fallbacks to Opus 4.8. Read the structured `thinking` blocks instead and let it keep its inner monologue inner. ^[source: web | 2026-06-18 | platform.claude.com adaptive-thinking, "Thinking output on Claude Fable 5 and Claude Mythos 5"]

## 5.6 200K vs 1M context -- when to reach for the big window

Fable 5, Opus 4.6-and-later, and Sonnet 4.6 support a 1M-token window; Haiku 4.5 stays capped at 200k. Select the big window with the `[1m]` suffix: ^[source: web | 2026-06-18 | code.claude.com model-config, "Extended context"]

```text
/model opus[1m]
/model sonnet[1m]
/model claude-opus-4-8[1m]
```

Availability and cost vary by plan, so point to the live page rather than quoting dollars. On Max, Team, and Enterprise, Opus auto-upgrades to 1M with no config at all, covering both Team Standard and Premium seats. Sonnet 1M never rides that auto-upgrade and requires usage credits on every subscription plan, Max included. On the Anthropic API and pay-as-you-go, Fable 5, Opus 4.8, and Opus 4.7 always run 1M. The 1M window bills at standard model pricing with no premium for the tokens beyond 200K. Turn it off entirely with `CLAUDE_CODE_DISABLE_1M_CONTEXT=1`. ^[source: web | 2026-06-18 | code.claude.com model-config, "Extended context" table + pricing paragraph]

The decision rule is quieter than the temptation. Default to the standard window for most tasks, because a curated 200k beats an uncurated 1M, and context rot still applies inside the big window just as surely as the small one (Ch. 02 walks through why). The 1M window earns its keep on the genuinely large jobs: 100-plus-file refactors, onboarding into a strange codebase, monorepo cross-package work. Even there, keep compacting proactively. And keep one eye on the tokenizer, because on Fable 5 and Opus 4.7-and-later that roughly 30% inflation means "1M" holds less text than your instincts from older models will tell you (section 5.1).

## 5.7 Prompt patterns for agentic coding

Something flipped on the 4.5 and 4.6 generations, and most of modern prompting follows from it. These models are proactive, literal, and capable enough that the failure mode reversed direction. The old prompts were written to fight under-doing, full of "CRITICAL: you MUST use this tool", "be thorough", "go above and beyond". Run those same prompts on current models and you get over-triggering, over-engineering, and over-delegation instead. The work now is mostly subtraction: remove the coaching, and add the why. ^[source: web | 2026-06-18 | platform.claude.com claude-prompting-best-practices, "Migration considerations" #6 + "Tool usage" overtrigger paragraph]

### Core principles, which carry across every model

The first principle is to be explicit about action versus suggestion. "Can you suggest changes..." often yields suggestions; "Change this function to improve performance" yields edits. The models follow instructions literally, so say what you mean. ^[source: web | 2026-06-18 | platform.claude.com claude-prompting-best-practices, "Tool usage" example]

The second, and the single highest-ROI habit, is to explain the why and not just the what. "NEVER use ellipses" is weaker than "Your response is read aloud by a text-to-speech engine, so never use ellipses since the engine will not know how to pronounce them." The model generalizes from the rationale. ^[source: web | 2026-06-18 | platform.claude.com claude-prompting-best-practices, "Add context to improve performance"]

The third is to tell it what to do, not what to avoid. "Your response should be composed of smoothly flowing prose paragraphs" beats "do not use markdown". ^[source: web | 2026-06-18 | platform.claude.com claude-prompting-best-practices, "Control the format of responses" #1]

The fourth is to dial back the anti-laziness language. Where you said "CRITICAL: You MUST use this tool when...", now say "Use this tool when...". Replace "Default to using [tool]" with "Use [tool] when it would enhance your understanding of the problem." Remove "if in doubt, use [tool]" entirely, because it over-triggers now. ^[source: web | 2026-06-18 | platform.claude.com claude-prompting-best-practices, "Overthinking and excessive thoroughness" + "Tool usage"]

The fifth is to ground every claim in an opened file. This is the anti-hallucination workhorse:
```text
<investigate_before_answering>
Never speculate about code you have not opened. If the user references a specific file,
you MUST read the file before answering. Make sure to investigate and read relevant
files BEFORE answering questions about the codebase. Never make any claims about code
before investigating unless you are certain of the correct answer - give grounded and
hallucination-free answers.
</investigate_before_answering>
```
^[source: web | 2026-06-18 | platform.claude.com claude-prompting-best-practices, "Minimizing hallucinations in agentic coding"]

### Curb over-engineering, the dominant code-quality failure

Opus 4.5 and 4.6, and Fable 5 at high effort, lean toward building too much: extra files nobody asked for, abstractions that wrap a single call site, speculative flexibility, defensive code guarding against scenarios that physically cannot happen. Anthropic's counter-prompt names every flavor of it directly (condensed here; the full version lives in the prompting guide):
```text
Avoid over-engineering. Only make changes that are directly requested or clearly
necessary. Keep solutions simple and focused:
- Scope: Don't add features, refactor code, or make "improvements" beyond what was
  asked. A bug fix doesn't need surrounding code cleaned up.
- Documentation: Don't add docstrings, comments, or type annotations to code you
  didn't change.
- Defensive coding: Don't add error handling, fallbacks, or validation for scenarios
  that can't happen. Trust internal code; only validate at system boundaries (user
  input, external APIs).
- Abstractions: Don't create helpers or utilities for one-time operations or
  hypothetical future requirements.
```
^[source: web | 2026-06-18 | platform.claude.com claude-prompting-best-practices, "Overeagerness" sample prompt]

### Don't let it game the tests

Pair the over-engineering guard with a "solve the general problem, not the test cases" instruction, and you close the door on a model that hard-codes values or writes a helper-script workaround just to turn the suite green:
```text
Write a high-quality, general-purpose solution using the standard tools available. Do
not create helper scripts or workarounds. Implement a solution that works for all valid
inputs, not just the test cases. Don't hard-code values. Tests verify correctness, they
do not define the solution. If the task is infeasible or a test is wrong, tell me rather
than working around it.
```
^[source: web | 2026-06-18 | platform.claude.com claude-prompting-best-practices, "Avoid focusing on passing tests and hard-coding"]

For the verification side of this -- TDD with agents, adversarial review, and the C-compiler case study -- see Ch. 11.

### Parallel tool calls

These models already parallelize tool calls well, and a `use_parallel_tool_calls` block pushes the hit rate toward ~100%. The shape of it (condensed from the official version):
```text
<use_parallel_tool_calls>
If you intend to call multiple tools and there are no dependencies between them, make all
the independent calls in parallel (e.g. reading 3 files = 3 parallel calls). If a call
depends on a previous call's output, call sequentially. Never use placeholders or guess
missing parameters.
</use_parallel_tool_calls>
```
When parallel bash starts bottlenecking your own machine, dial the other way with the inverse one-liner the docs supply: "Execute operations sequentially with brief pauses between each step to ensure stability." ^[source: web | 2026-06-18 | platform.claude.com claude-prompting-best-practices, "Optimize parallel tool calling" -- both sample prompts]

### Curb over-delegation

Opus 4.6 carries a strong predilection for subagents and will sometimes spawn one to grep when a direct grep would have finished already. When the delegation runs hot, rein it in:
```text
Use subagents when tasks can run in parallel, require isolated context, or involve
independent workstreams that don't need to share state. For simple tasks, sequential
operations, single-file edits, or tasks where you need to maintain context across steps,
work directly rather than delegating.
```
^[source: web | 2026-06-18 | platform.claude.com claude-prompting-best-practices, "Subagent orchestration" sample prompt]

Subagent mechanics themselves -- the built-ins, the 5-level nesting limit, forks, and the config knobs -- live in Ch. 07.

### Long-horizon and multi-context-window prompting

Tasks that outrun a single context window are the bread and butter of agentic engineering, and they reward a few specific prompt moves.

Tell the model that the harness compacts, so it does not politely wrap up early on a budget it imagines it is running out of:
```text
Your context window will be automatically compacted as it approaches its limit, allowing
you to continue working indefinitely from where you left off. Do not stop tasks early due
to token budget concerns. As you approach your token budget limit, save your current
progress and state to memory before the context window refreshes. Never artificially stop
a task early regardless of the context remaining.
```

Persist state in files: a structured format like `tests.json` for test and task status, a freeform `progress.txt` for notes, and git for the checkpoints. Remind it that "It is unacceptable to remove or edit tests because this could lead to missing or buggy functionality", because a model under pressure to make things pass will reach for the test file if you let it.

First-window and later-window prompts want to read differently. Spend the first window scaffolding: write the tests, lay down an `init.sh` setup script, build the runway. Later windows just iterate against the todo list. And when you can, prefer starting fresh over compaction, because these models are genuinely good at rediscovering their own state from the filesystem -- running `pwd`, reading `progress.txt`, `tests.json`, and the git log to find their place again. Hand it verification tools, Playwright MCP or computer use, so a long autonomous run can check its own work without you sitting there to confirm it. ^[source: web | 2026-06-18 | platform.claude.com claude-prompting-best-practices, "Long-horizon reasoning and state tracking" + "Multi-context window workflows"]

The mechanics of compaction itself, and the CLAUDE.md memory hierarchy these prompts lean on, live in Ch. 02 and Ch. 03 respectively.

### Reversibility guardrails at the prompt level

Left without guidance, the models will occasionally take an action you cannot take back: delete files, force-push, post something to the outside world. For judgment calls, prompt it. For a hard guarantee, reach for a hook (Ch. 09 covers this), because deterministic beats persuadable every time the stakes are real. The prompt form:
```text
Consider the reversibility and potential impact of your actions. Take local, reversible
actions like editing files or running tests freely. For actions that are hard to reverse,
affect shared systems, or could be destructive, ask before proceeding:
- Destructive: deleting files or branches, dropping database tables, rm -rf
- Hard to reverse: git push --force, git reset --hard, amending published commits
- Visible to others: pushing code, commenting on PRs/issues, sending messages
Do not use destructive actions as a shortcut, and don't bypass safety checks (e.g.
--no-verify).
```
^[source: web | 2026-06-18 | platform.claude.com claude-prompting-best-practices, "Balancing autonomy and safety" sample prompt]

The interactive backstop for this -- checkpoints, rewind, and the truth that Bash side effects are permanent -- is in Ch. 04.

### Prompting Fable 5 specifically -- less scaffolding, not more

Fable 5 follows instructions well enough that a brief instruction does the work an enumerated list used to do, and the over-prescriptive skills you wrote for older models can actively drag its output down. When you adopt it, re-audit your skills and prune the scaffolding you no longer need; Fable 5 will even update skills on the fly based on what it learns from the task at hand. ^[source: web | 2026-06-18 | platform.claude.com prompting-claude-fable-5, "Strong instruction following" + "Recommended scaffolding changes"]

Describe the outcome, not the steps. Hand Fable 5 the result you want and let it plan the path; pair it with `/goal` to keep it working until the outcome holds. Skip the verification reminders, since it self-verifies with less prompting. Size up: give it work you would normally break into pieces, and start at the top of your difficulty range rather than testing it on easy tasks that undersell its range. ^[source: web | 2026-06-18 | code.claude.com model-config, "Work with Fable 5"; platform.claude.com prompting-claude-fable-5, "Recommended scaffolding changes"]

Hand it ambiguity. Root-cause investigations, outage debugging, and architecture decisions are where the extra investigation pays off. ^[source: web | 2026-06-18 | platform.claude.com prompting-claude-fable-5, "Capability improvements" + "Recommended scaffolding changes"]

Expect longer turns by default. Hard tasks can run many minutes per request, and autonomous runs can stretch for hours. Adjust client timeouts, streaming, and progress indicators before you migrate, and consider async or scheduled check-ins rather than blocking until each run returns. ^[source: web | 2026-06-18 | platform.claude.com prompting-claude-fable-5, "Longer turns by default"]

Ground its progress claims. This one move nearly eliminated fabricated status in Anthropic's testing, even on tasks designed to elicit it:
```text
Before reporting progress, audit each claim against a tool result from this session.
Only report work you can point to evidence for; if something is not yet verified, say so
explicitly. If tests fail, say so with the output; if a step was skipped, say that; when
something is done and verified, state it plainly without hedging.
```
^[source: web | 2026-06-18 | platform.claude.com prompting-claude-fable-5, "Ground progress claims during long runs"]

Curb the elaboration and arrow-chain shorthand in user-facing summaries. A single brevity-plus-readability instruction -- lead with the outcome, write the final summary as a re-grounding for a reader who saw none of the work, drop the working vocabulary you built up along the way -- does as much as listing every pattern by name. ^[source: web | 2026-06-18 | platform.claude.com prompting-claude-fable-5, "Strong instruction following" + "Readability when communicating with the user"]

Watch for rare early-stopping. Deep in long runs, Fable 5 can end a turn on a statement of intent ("I'll now run X") without issuing the tool call. A "continue" or "go ahead end to end" suffices; for autonomous pipelines, add a system reminder that the model is operating autonomously and the user cannot answer mid-task. ^[source: web | 2026-06-18 | platform.claude.com prompting-claude-fable-5, "Rare cases of early stopping"]

### Frontend and design defaults

Left to their own devices, these models converge on what everyone now recognizes as AI slop: Inter and Space Grotesk fonts, purple gradients on white, the same cookie-cutter layout you have seen a hundred times. When frontend quality matters, include a `<frontend_aesthetics>` block steering distinctive typography, cohesive theming, purposeful motion, and atmospheric backgrounds. The full snippet lives in the official prompting guide and in the bundled `frontend-design` skill. ^[source: web | 2026-06-18 | platform.claude.com claude-prompting-best-practices, "Frontend design" -- frontend_aesthetics sample prompt]

### Prefills are gone on 4.6 and later

Here is a migration trap that bites silently: prefilled assistant responses on the last turn are rejected with a 400 on 4.6-and-later models and on Mythos Preview. If you leaned on prefill to force JSON, kill preambles, or steer around refusals, move that work over to Structured Outputs (a schema constraint), an explicit "respond directly without preamble" instruction, tool calling, or XML-tag output. Earlier models still accept prefill, which is exactly why the failure sneaks up on you when you upgrade. ^[source: web | 2026-06-18 | platform.claude.com claude-prompting-best-practices, "Migrating away from prefilled responses"]

## 5.8 Decision tables (slide-ready)

**Which model?**
| Situation | Model | Effort |
|-----------|-------|--------|
| Hardest, multi-day, ambiguous; outage RCA; large net-new systems | Fable 5 | `high` (-> `xhigh` for the most capability-sensitive) |
| Default complex reasoning + agentic coding | Opus 4.8 | `xhigh` for coding/agentic; `high` minimum |
| Daily feature work, tool-heavy flows | Sonnet 4.6 | `medium` (explicit) |
| Cheap parallel subagents, search, format checks | Haiku 4.5 | (no effort support) |
| Plan-then-build | `opusplan` | `xhigh` / `high` |
| 100-plus-file refactor, monorepo onboarding | `opus[1m]` / `sonnet[1m]` | `high` / `xhigh` |

**Which reasoning control?**
| You want... | Use |
|-----------|-----|
| Standing depth/cost for the session | `effort` (`/effort`, `effortLevel`, `CLAUDE_CODE_EFFORT_LEVEL`) |
| A one-off deeper-thinking nudge this turn | `ultrathink` in the prompt (does not change API effort) |
| `xhigh` plus auto multi-agent orchestration | `ultracode` (Claude Code only, session) |
| A hard cap on total token output | `max_tokens` |
| A per-skill or per-subagent override | `effort` in frontmatter |
| The old fixed budget on Opus 4.6 / Sonnet 4.6 only | `CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING=1` + `MAX_THINKING_TOKENS` |

**Prompting reflex on 4.6 and later (the inversion):**
| Old instinct (older models) | New move |
|------------------------------|------------------|
| "CRITICAL: you MUST use X" | "Use X when..." |
| "Be thorough / go above and beyond" | Say what specifically; trust defaults; lower effort if it overthinks |
| "Default to subagents" | Delegate only for parallel, isolated, or independent work |
| Prefill to force format | Structured Outputs / "respond directly" / XML tags |
| Hand-set `budget_tokens` | `effort` + `thinking: {type: "adaptive"}` |
| Remind it to verify (Fable 5) | Skip it -- it self-verifies; ground its progress claims instead |

## 5.9 The fastest-moving facts in this chapter

This is the most version-sensitive chapter in the book, and a few facts churn faster than the rest. None of this is uncertainty about what is true today; it is a heads-up about what to re-check before you script against it. The highest-churn item is alias resolution and account defaults (sections 5.2 and 5.3), which shift with every model release. Right behind it sit the minimum Claude Code versions (Opus 4.8 needs v2.1.154+, Fable 5 needs v2.1.170+, `/model`-saves-default landed in v2.1.153), the deprecation and retirement dates, and the effort defaults and per-model supported-levels matrix. When any of these are load-bearing, confirm them live: account defaults and version floors against the [Claude Code changelog](https://code.claude.com/docs/en/changelog.md), retirement dates against [model deprecations](https://platform.claude.com/docs/en/about-claude/model-deprecations), and aliases against the [models overview](https://platform.claude.com/docs/en/about-claude/models/overview).

## Sources

- Claude platform docs -- *Models overview* (lineup, specs, context windows, max output, tokenizer note, legacy table, deprecation warning). ^[source: web | 2026-06-18 | https://platform.claude.com/docs/en/about-claude/models/overview]
- Claude platform docs -- *Effort* (levels, per-model support, recommended effort by model, ultracode note). ^[source: web | 2026-06-18 | https://platform.claude.com/docs/en/build-with-claude/effort]
- Claude platform docs -- *Adaptive thinking* (adaptive vs manual vs disabled, supported models, display defaults, pricing, Fable 5 reasoning extraction). ^[source: web | 2026-06-18 | https://platform.claude.com/docs/en/build-with-claude/adaptive-thinking]
- Claude platform docs -- *Prompting best practices* (the proactivity inversion, the sample prompts, prefill migration). ^[source: web | 2026-06-18 | https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices]
- Claude platform docs -- *Prompting Claude Fable 5* (scaffolding changes, longer turns, grounding progress claims, early stopping). ^[source: web | 2026-06-18 | https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-fable-5]
- Claude platform docs -- *Pricing* (live per-MTok and 1M-window numbers). ^[source: web | 2026-06-18 | https://platform.claude.com/docs/en/about-claude/pricing]
- Claude platform docs -- *Model deprecations* (retirement dates and migration targets). ^[source: web | 2026-06-18 | https://platform.claude.com/docs/en/about-claude/model-deprecations]
- Claude Code docs -- *Model configuration* (`/model`, aliases, `opusplan`, fallback chains, provider resolution, effort, ultracode, ultrathink, extended thinking, extended context, third-party pinning). ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/model-config]
- Claude Code docs -- *Changelog* (version floors and per-feature minimums; re-verify version-pinned claims here). ^[source: web | 2026-06-18 | https://code.claude.com/docs/en/changelog.md]
