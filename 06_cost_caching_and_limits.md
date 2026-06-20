# Chapter 06 -- Cost, Caching and Limits

**TL;DR.** Running Claude Code well at the senior level is mostly an economics problem wearing an engineering costume. There are two billing worlds: a flat subscription (Pro, Max) where you trade dollars for a rolling usage allowance, and metered API billing through the Console where you pay per token and your spend climbs with your ambition. The subscription world bites through two rate-limit windows, a five-hour rolling session and a seven-day week, and both reset on schedules you do not get to pick. The metered world bites through tier-gated spend caps and per-minute token limits. Prompt caching is the single largest lever you have over the metered bill: a cache read costs a tenth of a fresh input token, Claude Code turns it on for you, and the things that quietly break it are the things you do without thinking, like switching models mid-session or clearing context. The multipliers that actually move your spend are the 1M context window, effort and thinking, and the fan-out math of subagents and agent teams, which run roughly N times the tokens of a single session. You watch all of it through `/usage`, `/cost`, the status line, and the Console, and you cap it through workspace spend limits and `/usage-credits`. Dollar figures drift; this chapter teaches you the structure and points you at the live numbers.

> **Reading note.** Exact dollar prices and rate-limit message counts change faster than a book can track, so they are described structurally and left to the live pages. Consult the [pricing page](https://platform.claude.com/docs/en/about-claude/pricing) for per-token rates and the [costs page](https://code.claude.com/docs/en/costs) for Claude Code spend guidance. Claude Code ships near-daily; version-pinned and tier-specific claims are accurate as of 2026-06-18, so re-verify them against the official docs before you script against them. Model routing and effort are the home of Chapter 05; context engineering is the home of Chapter 02; this chapter owns the money.

## 6.1 Two billing worlds and when each makes sense

Every Claude Code session bills against one of two meters, and which one you are on changes how you think about cost from the first keystroke.

A subscription, Pro or Max, is a flat monthly fee that buys a rolling usage allowance. You do not pay per token; you pay for the month and then spend against a budget that refills on its own schedule. Claude Code is included in both the Pro and the Max plan. [^1] [^2] On a subscription the dollar figure that `/usage` shows you for a session is an estimate computed locally and is not what you are billed; your subscription already covered it. [^3]

API billing through the Console is the metered world. You pay for input tokens, output tokens, and cache operations at the rates on the pricing page, and your monthly bill is the honest sum of everything you ran. When you first authenticate Claude Code against a Console account, a workspace called "Claude Code" is created automatically to centralize cost tracking; you cannot mint API keys in it, because it exists only for Claude Code's own authentication and usage. [^4]

The choice between them is not about which is cheaper in the abstract, because it depends entirely on how hard you drive. The honest way to decide is to look at the shape of your usage.

| Dimension | Subscription (Pro / Max) | API / Console (metered) |
|-----------|--------------------------|-------------------------|
| Cost shape | Flat monthly fee, predictable | Variable, scales with tokens consumed |
| What bites you | The five-hour and seven-day rate-limit windows | Spend caps and per-minute token limits |
| Marginal cost of one more turn | Zero dollars until you hit a window | Real dollars, every turn |
| Best fit | Steady solo or small-team daily driving | Bursty heavy automation, agent teams, CI |
| Cost visibility | Plan usage bars in `/usage`, the Console Usage page | Per-token Console reporting, `/cost`, `/usage` session estimate |
| Spend control | `/usage-credits` monthly limit on overage | Workspace spend limits in the Console |

The pragmatic read for a working senior is this. If your usage is steady and you are mostly one person driving one session at a time, a subscription is the right default because it converts a jittery per-token bill into a fixed cost and removes the small tax of watching the meter on every prompt. The moment your usage turns bursty and heavy, automation, scheduled routines, agent teams fanning out across a codebase, the metered world stops being the expensive option and starts being the controllable one, because spend caps and per-workspace limits give an admin a real lever that a flat subscription does not. Anthropic publishes a useful baseline for the metered case: across enterprise deployments, the average is around 13 dollars per developer per active day and 150 to 250 dollars per developer per month, with 90 percent of users staying under 30 dollars per active day. [^5] Treat those as planning anchors, not promises; a single careless agent-team run can clear a day's average in an afternoon.

A team that is rolling Claude Code out should not guess. Start a small pilot group, turn on the tracking tools, and establish a real baseline before a wider rollout. [^6]

## 6.2 The rate-limit windows and when they bite

Subscription usage is governed by two rolling windows, and understanding their shape is the difference between a smooth week and a Thursday where the model goes dark in the middle of a refactor.

The first window is a five-hour rolling session. Your session-based usage limit resets every five hours, and the segments are called sessions. [^7] The second is a weekly window that resets at a fixed time each week assigned to your account, the same time regardless of when you started using Claude or when your subscription began. [^8] Pro carries a single weekly limit that applies across all models. [^9] Max adds a second weekly limit for Sonnet models specifically, on top of the all-models one. [^10] When you run a window dry, Claude Code names which limit you hit and when it resets: the messages are "You've hit your session limit," "You've hit your weekly limit," and a model-specific "You've hit your Opus limit," each shown with its own reset time. [^11]

The tiers stack as multiples of Pro. Max 5x gives you five times the per-session usage of Pro; Max 20x gives you twenty times. [^12] How many messages that actually buys you is deliberately not a fixed number, because consumption depends on message length, file attachment size, conversation length, tool usage, model choice, effort level, and artifact creation. [^13]

What matters for an engineer is the failure mode, more than the headline multiple. When a window empties you do not get a gentle warning, you get a wall: Claude Code shows a message like "You've hit your session limit - resets 3:45pm", "You've hit your weekly limit - resets Mon 12:00am", or "You've hit your Opus limit - resets 3:45pm", and blocks further requests until the reset time shown. [^14] The five-hour wall is annoying but self-healing; wait it out and you are back. The weekly wall is the one that ruins a sprint, because it can leave you locked out for days; the in-CLI escape is to buy usage credits with `/usage-credits` (Pro and Max), which is covered in 6.7. The two windows compose: a heavy five-hour burst can be fine, but four of them in a week can drain the weekly bucket while every individual session looked healthy.

You do not have to fly blind into either wall. Claude Code exposes both windows to your status line. The `rate_limits` object carries `five_hour` and `seven_day`, each with a `used_percentage` from 0 to 100 and a `resets_at` in Unix epoch seconds. [^15] The object appears only for Claude.ai subscribers (Pro and Max) and only after the first API response of a session, and each window can be independently absent, so a robust status line handles the missing case. [^16]

```bash
#!/bin/bash
input=$(cat)
MODEL=$(echo "$input" | jq -r '.model.display_name')
# "// empty" produces no output when rate_limits is absent
FIVE_H=$(echo "$input" | jq -r '.rate_limits.five_hour.used_percentage // empty')
WEEK=$(echo "$input"  | jq -r '.rate_limits.seven_day.used_percentage // empty')

LIMITS=""
[ -n "$FIVE_H" ] && LIMITS="5h: $(printf '%.0f' "$FIVE_H")%"
[ -n "$WEEK" ]   && LIMITS="${LIMITS:+$LIMITS }7d: $(printf '%.0f' "$WEEK")%"

[ -n "$LIMITS" ] && echo "[$MODEL] | $LIMITS" || echo "[$MODEL]"
```
[^17]

The senior move is to wire the weekly percentage into your status line and treat it like a fuel gauge on a long drive. The five-hour number tells you whether to take a coffee break; the seven-day number tells you whether to switch to a cheaper model, defer the agent-team run to next week, or shift heavy work to a metered key. The Console Usage page is the authoritative record for subscribers and shows plan usage bars, activity stats, and a usage breakdown that the local status line cannot fully replicate. [^18]

### The metered world has its own limits

API users on the Console live under a different limit regime entirely, and it is worth knowing even if you mostly subscribe, because the moment you hand Claude Code an API key you inherit it. Metered usage is governed by spend limits, a monthly dollar ceiling, and rate limits measured in requests per minute (RPM), input tokens per minute (ITPM), and output tokens per minute (OTPM) per model class. [^19] Your organization climbs through usage tiers automatically as cumulative credit purchases cross thresholds, and each tier raises both your monthly spend ceiling and your per-minute limits. [^20]

| Usage tier | Monthly spend ceiling | Opus 4.x ITPM | Sonnet 4.x ITPM |
|------------|-----------------------|---------------|-----------------|
| Tier 1 | $500 | 500,000 | 30,000 |
| Tier 2 | $500 | 2,000,000 | 450,000 |
| Tier 3 | $1,000 | 5,000,000 | 800,000 |
| Tier 4 | $200,000 | 10,000,000 | 2,000,000 |
| Monthly Invoicing | No limit | Custom | Custom |

[^21]

Exceed any per-minute limit and you get a 429 with a `retry-after` header telling you how long to wait; Claude Code's own retry logic handles transient 429s for you, but sustained ones mean you have outgrown your tier. [^22] The single most important fact buried in that page, and the bridge to the next section, is that for most current models only uncached input tokens count toward ITPM. Cache reads do not. [^23] That turns prompt caching from a cost optimization into a throughput multiplier: with an 80 percent cache hit rate against a 2,000,000 ITPM limit you can effectively push 10,000,000 total input tokens per minute, because the 8M cached tokens are free of the limit. [^24]

## 6.3 Prompt caching: the largest lever you have

A coding session is the most cache-friendly workload there is. The same system prompt, the same tool definitions, the same CLAUDE.md, the same files you read ten turns ago, all of it sits at the front of the context and almost never changes. Prompt caching lets the model resume from a prefix it has already processed instead of re-reading it, and on a long session that prefix is most of your input. [^25]

### What it costs and why it pays for itself

Caching is priced as multipliers on the base input rate, and the asymmetry is the whole point.

| Cache operation | Multiplier on base input | What it means |
|-----------------|--------------------------|---------------|
| Base input token | 1.0x | A fresh, uncached token |
| 5-minute cache write | 1.25x | The premium to store a prefix for five minutes |
| 1-hour cache write | 2.0x | The premium to store a prefix for an hour |
| Cache read (hit) | 0.1x | Reusing a stored prefix |

[^26]

The write costs a little more than a normal token once; every read after that costs a tenth of one. A cache hit costs 10 percent of the standard input price, which means the five-minute cache pays for itself after a single read (the 1.25x write is recovered by avoiding one full-price re-read), and the one-hour cache pays for itself after two reads. [^27] In an interactive session where you send turn after turn against the same large prefix, you cross that break-even instantly and stay in the black for the rest of the conversation. The default cache lifetime is five minutes, refreshed at no cost each time the cached content is read, with a one-hour option for an additional cost. [^28]

### How Claude Code uses it

You do not configure any of this by hand. Claude Code applies prompt caching automatically to optimize performance and reduce cost, caching the repeated content, system prompts, tool definitions, and the like, that makes up the stable front of a session. [^29] You can turn it off, globally or per model tier, with environment variables, though there is almost never a reason to: `DISABLE_PROMPT_CACHING=1` kills it everywhere, and `DISABLE_PROMPT_CACHING_HAIKU`, `_SONNET`, `_OPUS`, and `_FABLE` kill it per tier. [^30]

The mechanics underneath are worth knowing because they explain why some habits silently raise your bill. The cache follows a strict prefix hierarchy, `tools` then `system` then `messages`, and a change at any level invalidates that level and everything after it. [^31] So a change to your tool set blows away the entire cache; a change to the system prompt blows away system and messages but spares the tool prefix; a new user turn at the very end costs you nothing because everything before it is still a valid prefix.

### What invalidates the cache, and the habits that cost you

Most cache misses in Claude Code are self-inflicted. The list below is the one to internalize.

| What you do | What it invalidates | The cost |
|-------------|---------------------|----------|
| Switch models mid-session (`/model`) | The whole cache; the next call re-reads full history uncached | A full-price re-read of your entire context |
| Add or remove a tool / MCP server | Tools, system, and messages -- everything | The whole prefix is rebuilt |
| Edit CLAUDE.md mid-session (changes the system prompt) | System and messages, sparing the tool prefix | System and message prefixes rebuilt |
| Toggle thinking / change the thinking parameter | The messages cache only | Re-read of message history; tool and system prefixes survive |
| Add or remove an image anywhere in context | The messages cache only | Re-read of message history; tool and system prefixes survive |
| Clear tool results (context editing) | The cached prefix at the clear point | Cache write costs on the next call |

[^32]

The model-switch line deserves emphasis because Claude Code warns you about it directly. When you open the `/model` picker on a conversation that already has output, it asks for confirmation, because the next response re-reads the full history without cached context. [^33] On a long session that single switch can be one of the most expensive keystrokes you make all day. The discipline is simple: decide your model and effort at the top of a session and leave them alone. If you genuinely need a second model's judgment mid-task, that is what subagents and the advisor tool are for (see Ch. 05) -- they spend tokens in a side context instead of detonating your main cache.

### Reading where your cache is landing

The status line breaks current usage into its components, and the two that tell the caching story are `cache_creation_input_tokens` (what you wrote to cache this turn) and `cache_read_input_tokens` (what you reused). [^34] A healthy long session shows reads dwarfing fresh input; if you see large creation numbers turn after turn, something in your prefix is churning and you are paying the write premium repeatedly without ever cashing in the read discount. The same `usage` fields are reported by the API directly, so the same diagnosis works whether you are subscribed or metered. [^35]

## 6.4 Context editing as a cost lever, and its cache tax

Caching rewards a long, stable prefix. Context editing does the opposite, it actively removes content from the conversation, and the tension between the two is the most subtle cost decision in this chapter. (The mechanics of context management and compaction are the home of Ch. 02; here we treat only the money.)

Context editing selectively clears specific content from conversation history as it grows, both to stay within limits and to keep the model focused, because context is a finite resource with diminishing returns and stale content degrades attention. [^36] Two strategies matter for cost. The `clear_tool_uses` strategy drops old tool results once Claude has processed them, which is ideal for agentic workflows where file contents and search output pile up. [^37] The `clear_thinking` strategy drops thinking blocks under context pressure. [^38]

Here is the trap. Clearing tool results invalidates the cached prefix at the point where the clear happens. [^39] You free context space, but you also pay a fresh cache write on the next call. Clearing thinking blocks does the same: keep them and the cache is preserved, clear them and the cache invalidates at the clear point. [^40] So a too-eager clearing policy can cost you more than it saves, because you are repeatedly demolishing and rebuilding a prefix you were getting at a tenth price.

The fix is to clear in worthwhile chunks. The `clear_at_least` parameter sets a minimum number of tokens to clear per activation, so you only break the cache when you will free enough context to justify the write. [^41] The decision reduces to a single sentence: clear when the context savings outweigh one cache write, and not before.

| Choice | Cache impact | When it is the right call |
|--------|--------------|---------------------------|
| Keep the long context | Cache hits preserved, low input cost | Short or medium sessions, high reasoning value in the history |
| Clear tool results | Cache invalidated at clear point | Long agentic runs drowning in stale tool output; gate with `clear_at_least` |
| Keep thinking blocks | Cache preserved | Extended-thinking work where reasoning continuity pays |
| Clear thinking blocks | Cache invalidated at clear point | Only when you are critically short on context |

[^42]

In day-to-day Claude Code use the same tension shows up in plainer clothes. `/clear` between unrelated tasks starts a fresh session and stops you from dragging stale context, and its tokens, into every subsequent message. [^43] That is the cheap, blunt version of the same idea: a new task does not deserve the old task's prefix.

## 6.5 The multipliers that actually move your spend

A handful of choices dominate your token bill, and they compound. Knowing their rough magnitudes lets you reason about cost before you press enter instead of after you read the invoice.

### The 1M context window

The headline first, because it surprises people: the 1M-token context window uses standard model pricing with no premium for tokens beyond 200K. A 900K-token request bills at the same per-token rate as a 9K one. [^44] [^45]

The multiplier is therefore not in the rate, it is in the volume. There is no rate cliff, but a million-token context is a million tokens you pay for on every single turn until you compact or clear, because the whole context is re-sent each call (caching softens this, but cache reads are still 0.1x, not zero). The cost of a 1M session is the cost of carrying that weight turn after turn. On Max, Team, and Enterprise plans Opus is automatically upgraded to 1M context at no extra configuration; Sonnet with 1M is not part of that upgrade and requires usage credits on every plan including Max. [^46] So the senior instinct, reach for the biggest window available, is exactly backwards as a default. Use the window you need, let compaction and clearing keep the carried weight down, and treat 1M as a deliberate tool for genuinely large codebases rather than a free upgrade.

### Effort and thinking

Extended thinking is on by default because it sharply improves complex reasoning, and thinking tokens are billed as output tokens, the most expensive kind. The default budget can run to tens of thousands of tokens per request depending on the model. [^47] On adaptive-reasoning models the effort level is the dial, `low` through `max`, and lower effort means fewer thinking tokens and a smaller bill. [^48] The cost framing, as opposed to the capability framing that lives in Ch. 05, is blunt: every step up the effort ladder buys reasoning with output tokens, and `max` removes the constraint on token spending entirely. [^49] For latency-sensitive or mechanically simple work, dropping effort is the cheapest large saving you can make without touching anything else. Note that thinking cannot be turned off on Fable 5, which always reasons; the toggle and `MAX_THINKING_TOKENS=0` have no effect there. [^50]

### Subagents and agent teams: the Nx multiplier

This is where bills get surprising, because the cost is not linear in your effort, it is linear in the number of agents you spawn. A subagent runs verbose work, test runs, log processing, doc fetching, in its own context so only a summary returns to your main conversation; that keeps your main context lean, but the subagent still spends real tokens on its own context. [^51] (Subagent mechanics, nesting, and config knobs are the home of Ch. 08; orchestration economics and the research-system figures are the home of Ch. 11.)

Agent teams take the multiplier further. They spawn multiple full Claude Code instances, each carrying its own context window, so token usage scales roughly with team size and how long each teammate runs. [^52] Anthropic puts a concrete number on it: agent teams use approximately 7x more tokens than a standard session when teammates run in plan mode, because each teammate is a separate instance with its own context. [^53] Agent teams are disabled by default and gated behind `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`, which is the right posture for a feature with that cost profile. [^54]

The cost discipline for teams is concrete: run Sonnet for teammates rather than Opus, since coordination work does not need the top tier; keep teams small, because cost is roughly proportional to headcount; keep spawn prompts focused, since everything in the prompt loads into each teammate's context from the start; and shut teammates down when their work is done, because an idle-but-alive teammate keeps consuming tokens until it exits. [^55] You can pin every subagent and teammate to a cheap model in one move with `CLAUDE_CODE_SUBAGENT_MODEL`. [^56]

### Background token use

There is a small floor under every session. Claude Code spends a trickle of tokens on background work even when idle, summarizing prior conversations for `--resume` and processing some status commands, typically under four cents per session. [^57] That background work runs on Haiku by default, controllable through `ANTHROPIC_DEFAULT_HAIKU_MODEL`. [^58] It is noise next to an agent-team run, but it is real, and it is why a session left open all day is never quite free.

| Multiplier | Rough magnitude | The cost lever |
|------------|-----------------|----------------|
| 1M context window | No rate premium; volume carried every turn | Use the window you need; compact and clear to drop carried weight |
| Effort / thinking | Default thinking can be tens of thousands of output tokens | Lower effort for simple or latency-sensitive work |
| Subagent | One extra context per agent | Isolates verbose work; pin to a cheap model |
| Agent team | ~7x a standard session in plan mode | Sonnet teammates, small teams, shut them down |
| Background | Under ~$0.04 per session | Negligible; runs on Haiku |

## 6.6 Reading where the tokens go: /usage, /cost, and the Console

You cannot manage what you cannot see, and Claude Code gives you four lenses, each with a different fidelity and a different audience.

`/usage` is the primary in-CLI lens. Its Session block shows token statistics for the current session and a locally computed dollar estimate, with the explicit caveat that the figure may differ from your actual bill. [^59] For subscribers it goes further and shows plan usage bars, activity stats, and a breakdown that attributes recent usage to skills, subagents, plugins, and individual MCP servers, each as a percentage of the total, with `d` and `w` toggling the last 24 hours versus the last 7 days. [^60] That attribution breakdown is the diagnostic that matters: it tells you which MCP server or which subagent is quietly eating your allowance, and that is almost never the thing you would have guessed.

`/cost` and the Session block surface the same locally computed session estimate; the headline numbers look like this:

```text
Total cost:            $0.55
Total duration (API):  6m 19.7s
Total duration (wall): 6h 33m 10.2s
Total code changes:    0 lines added, 0 lines removed
```
[^61]

The status line is the always-on lens. `cost.total_cost_usd` accumulates the estimated cost of every API call in the session, and the context fields, `used_percentage`, `current_usage` with its cache split, let you watch the meter without typing a command. [^62] `/context` shows what is currently consuming the window, which is the right tool when `/usage` tells you the bill is high and you need to find the bloat. [^63]

The Console is the lens of record. For authoritative billing, the Usage page in the Claude Console is the source of truth; the in-CLI figures are estimates computed from local session history on this machine, which means usage from other devices or from claude.ai does not appear in them. [^64] The hierarchy to remember: status line and `/usage` for fast, local, approximate situational awareness; the Console for the number you actually owe.

One gap worth flagging for enterprise readers. On Bedrock, Vertex, and Foundry, Claude Code does not send cost metrics from your cloud, so several large enterprises route through LiteLLM, an open-source proxy that tracks spend by key, to recover visibility. [^65] That project is unaffiliated with Anthropic and unaudited, so treat it as a community workaround rather than a blessed path.

## 6.7 Spend caps and managed cost controls

Visibility is half the job; the other half is putting a hard ceiling under the spend so a runaway automation cannot bankrupt a budget overnight. The controls differ by billing world.

On the metered side, an admin sets workspace spend limits on the total Claude Code workspace spend, and can view cost and usage reporting in the Console. [^66] Because the auto-created "Claude Code" workspace counts toward your organization's overall API rate limits, you can also set a workspace rate limit on it to cap Claude Code's share and protect other production workloads from being starved. [^67] Workspace limits are the cleanest lever a team lead has: they are enforced server-side, they do not depend on every developer behaving, and they let one noisy workspace's overflow flow back to the others rather than blocking the whole org. [^68]

On the subscription side, Pro and Max users can set a monthly spend limit on usage credits with the `/usage-credits` command, and if you hit that limit while credits remain, Claude Code prompts you to raise or remove it without leaving the CLI; changing it requires billing access on the account. [^69] This is the safety valve for the overage that sits on top of a subscription, the 1M-Sonnet credits, the fast-mode credits, and it is worth setting deliberately rather than discovering its absence the hard way.

Enterprise admins also have a softer control worth knowing in a cost chapter: model restriction. `availableModels` in managed or policy settings restricts which models users can select, and combined with `enforceAvailableModels` it can stop the Default option from resolving to an expensive tier. [^70] An organization that wants its fleet on Sonnet by default, with Opus reserved for the people who genuinely need it, enforces that here rather than hoping for discipline. The cost lever in a managed setting is rarely a single dollar cap; it is the combination of a spend ceiling, a model allowlist, and a default that points at the cheap tier.

| Control | Billing world | Where you set it |
|---------|---------------|------------------|
| Workspace spend limit | Metered (API) | Console workspace Limits page |
| Workspace rate limit | Metered (API) | Console workspace Limits page |
| `/usage-credits` monthly cap | Subscription (Pro/Max) | In-CLI, requires billing access |
| `availableModels` allowlist | Either | Managed / policy settings |
| Tier spend ceiling | Metered (API) | Console Limits, advances with credit purchases |

## 6.8 The cost-versus-capability calculus

Everything in this chapter resolves to a single recurring decision: how much capability does this particular task actually need, and what is the cheapest way to buy exactly that much. The mistake that costs seniors the most is reaching for the top of every dial by reflex, the biggest model, the highest effort, the largest window, a team of agents, when the task in front of them wanted a fraction of it.

The routing logic lives in Ch. 05 and the context logic in Ch. 02, so here is only the money summary. Sonnet handles most coding work well and costs less than Opus; reserve Opus for complex architecture and multi-step reasoning, and switch with `/model` rather than running everything on the expensive tier out of habit. [^71] Specific prompts beat vague ones on cost as much as on quality, because "improve this codebase" triggers broad scanning while "add input validation to the login function in auth.ts" lets the model work with minimal file reads. [^72] Plan mode is a cost tool as much as a quality one: it surfaces a wrong direction before you have paid to execute it, and `/rewind` lets you abandon an expensive dead end without re-running the whole session. [^73] (Checkpoints and rewind are the home of Ch. 04; verification, which is itself a cost decision because catching a bad direction early is cheaper than fixing it late, is the home of Ch. 12.)

The deepest cost lever is the cheapest to pull and the easiest to forget: keep the context small. Token cost scales with context size, so every habit that keeps the window lean, clearing between tasks, deferring MCP tool definitions, preferring CLI tools over MCP servers, offloading log-grepping to hooks, moving rarely-used CLAUDE.md content into on-demand skills, pays out on every message for the rest of the session. [^74] Caching makes a stable prefix cheap; context discipline keeps the prefix small. Hold both at once and the bill mostly takes care of itself.

If there is one sentence to carry out of this chapter, it is that cost in Claude Code is not a number you discover at month-end, it is a series of small decisions you make all day, and a senior who watches the gauges and matches the dial to the task spends a fraction of what the same work costs the engineer who leaves everything maxed and finds out later.

## Quick reference

| You want to... | Reach for |
|----------------|-----------|
| Pick a billing model | Steady solo: subscription. Bursty / team automation: metered Console |
| See current session cost | `/cost` or `/usage` Session block (local estimate) |
| See what is eating your plan allowance | `/usage` breakdown, `d`/`w` toggle (subscribers) |
| See what is filling the context window | `/context` |
| Watch limits and cost continuously | Status line: `rate_limits.*`, `cost.total_cost_usd`, `context_window.used_percentage` |
| Get the authoritative bill | Console Usage page |
| Cap subscription overage | `/usage-credits` monthly limit |
| Cap team / API spend | Workspace spend limit + workspace rate limit (Console) |
| Cut thinking cost | Lower `/effort`; `MAX_THINKING_TOKENS=0` (not on Fable 5) |
| Cut team cost | Sonnet teammates, small teams, `CLAUDE_CODE_SUBAGENT_MODEL`, shut them down |
| Keep cache hits high | Fix model + effort at session start; avoid mid-session `/model`, tool, and CLAUDE.md churn |
| Free context without wrecking cache | Context editing with `clear_at_least`; `/clear` between unrelated tasks |
| Restrict the fleet's models | `availableModels` + `enforceAvailableModels` (managed settings) |

For your own subscription windows, the authoritative live view is your account's Settings > Usage page (or `/usage` in the CLI); the per-token rates and tier limits live on the official docs in Sources below.

## Sources

- https://platform.claude.com/docs/en/about-claude/pricing
- https://platform.claude.com/docs/en/build-with-claude/prompt-caching
- https://platform.claude.com/docs/en/build-with-claude/context-editing
- https://platform.claude.com/docs/en/api/rate-limits
- https://code.claude.com/docs/en/costs
- https://code.claude.com/docs/en/model-config
- https://code.claude.com/docs/en/statusline
- https://code.claude.com/docs/en/errors
- https://support.claude.com/en/articles/8325606-what-is-the-pro-plan
- https://support.claude.com/en/articles/11049741-what-is-the-max-plan
- https://support.claude.com/en/articles/9797557-usage-limit-best-practices
- https://support.claude.com/en/articles/11145838-use-claude-code-with-your-pro-or-max-plan

[^1]: web | 2026-06-18 | [support.claude.com](https://support.claude.com) "What is the Pro plan?" -- "Claude Code access"
[^2]: web | 2026-06-18 | [support.claude.com](https://support.claude.com) "What is the Max plan?" -- "Access to Claude Code"
[^3]: web | 2026-06-18 | [code.claude.com](https://code.claude.com) costs, "Using the /usage command" note
[^4]: web | 2026-06-18 | [code.claude.com](https://code.claude.com) costs, "Managing costs for teams" note
[^5]: web | 2026-06-18 | [code.claude.com](https://code.claude.com) costs -- "the average cost is around \$13 per developer per active day and \$150-250 per developer per month"
[^6]: web | 2026-06-18 | [code.claude.com](https://code.claude.com) costs -- "start with a small pilot group and use the tracking tools below to establish a baseline"
[^7]: web | 2026-06-18 | [support.claude.com](https://support.claude.com) "What is the Pro plan?" -- "Your session-based usage limit will reset every five hours"
[^8]: web | 2026-06-18 | [support.claude.com](https://support.claude.com) "What is the Max plan?" -- "Weekly limits reset at a fixed time each week that is assigned to your account"
[^9]: web | 2026-06-18 | [support.claude.com](https://support.claude.com) "What is the Pro plan?" -- "Pro plans also have a weekly usage limit that applies across all models"
[^10]: web | 2026-06-18 | [support.claude.com](https://support.claude.com) "What is the Max plan?" -- "Max plans also have two weekly usage limits: one that applies across all models and another for Sonnet models only."
[^11]: web | 2026-06-18 | [https://code.claude.com/docs/en/errors](https://code.claude.com/docs/en/errors) -- "You've hit your session limit / You've hit your weekly limit / You've hit your Opus limit"
[^12]: web | 2026-06-18 | [support.claude.com](https://support.claude.com) "What is the Max plan?" -- "Max 5x provides 5 times more usage per session than the Pro plan" / "Max 20x provides 20 times more usage per session than the Pro plan"
[^13]: web | 2026-06-18 | [support.claude.com](https://support.claude.com) "Usage limit best practices" -- "Additional factors that affect your usage limits include: Message length, File attachment size, Current conversation length, Tool usage (e.g., Research, web search), Model choice, Effort level, Artifact creation and usage"
[^14]: web | 2026-06-18 | [https://code.claude.com/docs/en/errors](https://code.claude.com/docs/en/errors)
[^15]: web | 2026-06-18 | [code.claude.com](https://code.claude.com) statusline, "Available data" -- rate_limits fields
[^16]: web | 2026-06-18 | [code.claude.com](https://code.claude.com) statusline, "Rate limit usage" -- "only present for Claude.ai subscribers (Pro/Max) after the first API response"
[^17]: web | 2026-06-18 | [code.claude.com](https://code.claude.com) statusline, "Rate limit usage" Bash example
[^18]: web | 2026-06-18 | [code.claude.com](https://code.claude.com) costs, "Using the /usage command" -- subscriber view
[^19]: web | 2026-06-18 | [platform.claude.com](https://platform.claude.com) api/rate-limits -- "measured in requests per minute (RPM), input tokens per minute (ITPM), and output tokens per minute (OTPM)"
[^20]: web | 2026-06-18 | [platform.claude.com](https://platform.claude.com) api/rate-limits -- "Requirements to advance tier" table
[^21]: web | 2026-06-18 | [platform.claude.com](https://platform.claude.com) api/rate-limits -- spend limit and Tier 1-4 ITPM tables, Opus 4.x and Sonnet 4.x rows
[^22]: web | 2026-06-18 | [platform.claude.com](https://platform.claude.com) api/rate-limits -- "you will get a 429 error ... along with a retry-after header"
[^23]: web | 2026-06-18 | [platform.claude.com](https://platform.claude.com) api/rate-limits -- "only uncached input tokens count towards your ITPM rate limits"
[^24]: web | 2026-06-18 | [platform.claude.com](https://platform.claude.com) api/rate-limits -- "With a 2,000,000 ITPM limit and an 80% cache hit rate ... 10,000,000 total input tokens per minute"
[^25]: web | 2026-06-18 | [platform.claude.com](https://platform.claude.com) build-with-claude/prompt-caching -- "allowing you to resume from specific prefixes in your prompts"
[^26]: web | 2026-06-18 | [platform.claude.com](https://platform.claude.com) about-claude/pricing -- "Prompt caching" multiplier table
[^27]: web | 2026-06-18 | [platform.claude.com](https://platform.claude.com) about-claude/pricing -- "caching pays off after just one cache read for the 5-minute duration ... or after two cache reads for the 1-hour duration"
[^28]: web | 2026-06-18 | [platform.claude.com](https://platform.claude.com) build-with-claude/prompt-caching -- "Default Cache Lifetime: 5 minutes" / "Extended Cache Option: 1-hour"
[^29]: web | 2026-06-18 | [code.claude.com](https://code.claude.com) costs -- "Claude Code automatically optimizes costs through prompt caching"
[^30]: web | 2026-06-18 | [code.claude.com](https://code.claude.com) model-config, "Prompt caching configuration" -- env var table
[^31]: web | 2026-06-18 | [platform.claude.com](https://platform.claude.com) build-with-claude/prompt-caching -- "Cache follows a hierarchy: tools -> system -> messages"
[^32]: web | 2026-06-18 | [platform.claude.com](https://platform.claude.com) build-with-claude/prompt-caching -- "What Invalidates the Cache" table: tool-definition row invalidates tools/system/messages; system-prompt-content row invalidates system/messages; thinking-parameter and image rows invalidate the messages cache only
[^33]: web | 2026-06-18 | [code.claude.com](https://code.claude.com) model-config, "Setting your model" -- "The picker asks for confirmation when the conversation has prior output, since the next response re-reads the full history without cached context"
[^34]: web | 2026-06-18 | [code.claude.com](https://code.claude.com) statusline, "Context window fields" -- current_usage breakdown
[^35]: web | 2026-06-18 | [platform.claude.com](https://platform.claude.com) build-with-claude/prompt-caching -- "Tracking Cache Performance" usage fields
[^36]: web | 2026-06-18 | [platform.claude.com](https://platform.claude.com) context-editing -- "selectively clear specific content from conversation history as it grows"
[^37]: web | 2026-06-18 | [platform.claude.com](https://platform.claude.com) context-editing -- "clear_tool_uses_20250919 strategy clears tool results"
[^38]: web | 2026-06-18 | [platform.claude.com](https://platform.claude.com) context-editing -- "clear_thinking_20251015 strategy manages thinking blocks"
[^39]: web | 2026-06-18 | [platform.claude.com](https://platform.claude.com) context-editing -- "Tool result clearing: Invalidates cached prompt prefixes when content is cleared"
[^40]: web | 2026-06-18 | [platform.claude.com](https://platform.claude.com) context-editing -- "When thinking blocks are cleared, the cache is invalidated at the point where clearing occurs"
[^41]: web | 2026-06-18 | [platform.claude.com](https://platform.claude.com) context-editing -- "Use the clear_at_least parameter to ensure a minimum number of tokens is cleared each time"
[^42]: web | 2026-06-18 | [platform.claude.com](https://platform.claude.com) context-editing -- "Context editing and prompt caching": tool-result clearing invalidates cached prefixes; thinking blocks kept preserve the cache, cleared invalidates at the clear point
[^43]: web | 2026-06-18 | [code.claude.com](https://code.claude.com) costs, "Manage context proactively" -- "Use /clear to start fresh when switching to unrelated work"
[^44]: web | 2026-06-18 | [code.claude.com](https://code.claude.com) model-config, "Extended context" -- "uses standard model pricing with no premium for tokens beyond 200K"
[^45]: web | 2026-06-18 | [platform.claude.com](https://platform.claude.com) about-claude/pricing -- "Long context pricing" -- "A 900k-token request is billed at the same per-token rate as a 9k-token request"
[^46]: web | 2026-06-18 | [code.claude.com](https://code.claude.com) model-config, "Extended context" -- plan table
[^47]: web | 2026-06-18 | [code.claude.com](https://code.claude.com) costs, "Adjust extended thinking" -- "Thinking tokens are billed as output tokens, and the default budget can be tens of thousands of tokens per request"
[^48]: web | 2026-06-18 | [code.claude.com](https://code.claude.com) model-config, "Adjust effort level"
[^49]: web | 2026-06-18 | [code.claude.com](https://code.claude.com) model-config, "Choose an effort level" -- max "with no constraint on token spending"
[^50]: web | 2026-06-18 | [code.claude.com](https://code.claude.com) model-config, "Extended thinking" -- "Thinking cannot be turned off on Fable 5"
[^51]: web | 2026-06-18 | [code.claude.com](https://code.claude.com) costs, "Delegate verbose operations to subagents"
[^52]: web | 2026-06-18 | [code.claude.com](https://code.claude.com) costs, "Agent team token costs" -- "Token usage scales with the number of active teammates"
[^53]: web | 2026-06-18 | [code.claude.com](https://code.claude.com) costs, "Manage agent team costs" -- "approximately 7x more tokens than standard sessions when teammates run in plan mode"
[^54]: web | 2026-06-18 | [code.claude.com](https://code.claude.com) costs, "Agent team token costs" -- "disabled by default. Set CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1"
[^55]: web | 2026-06-18 | [code.claude.com](https://code.claude.com) costs, "Agent team token costs" -- cost management bullets
[^56]: web | 2026-06-18 | [code.claude.com](https://code.claude.com) model-config, env vars -- CLAUDE_CODE_SUBAGENT_MODEL
[^57]: web | 2026-06-18 | [code.claude.com](https://code.claude.com) costs, "Background token usage" -- "typically under \$0.04 per session"
[^58]: web | 2026-06-18 | [code.claude.com](https://code.claude.com) model-config, env vars -- ANTHROPIC_DEFAULT_HAIKU_MODEL "or background functionality"
[^59]: web | 2026-06-18 | [code.claude.com](https://code.claude.com) costs, "Using the /usage command" -- session block, local estimate caveat
[^60]: web | 2026-06-18 | [code.claude.com](https://code.claude.com) costs, "Using the /usage command" -- subscriber breakdown by skills/subagents/plugins/MCP
[^61]: web | 2026-06-18 | [code.claude.com](https://code.claude.com) costs, "Using the /usage command" session output sample
[^62]: web | 2026-06-18 | [code.claude.com](https://code.claude.com) statusline, "Available data" -- cost and context_window fields
[^63]: web | 2026-06-18 | [code.claude.com](https://code.claude.com) costs, "Reduce MCP server overhead" -- "Run /context to see what's consuming space"
[^64]: web | 2026-06-18 | [code.claude.com](https://code.claude.com) costs, "Using the /usage command" -- "computed from local session history on this machine"
[^65]: web | 2026-06-18 | [code.claude.com](https://code.claude.com) costs, "Managing costs for teams" -- "Claude Code does not send metrics from your cloud ... using LiteLLM"
[^66]: web | 2026-06-18 | [code.claude.com](https://code.claude.com) costs, "Managing costs for teams" -- "set workspace spend limits ... view cost and usage reporting"
[^67]: web | 2026-06-18 | [code.claude.com](https://code.claude.com) costs -- "set a workspace rate limit on this workspace's Limits page ... to cap Claude Code's share"
[^68]: web | 2026-06-18 | [platform.claude.com](https://platform.claude.com) api/rate-limits, "Setting lower limits for Workspaces"
[^69]: web | 2026-06-18 | [code.claude.com](https://code.claude.com) costs, "Managing costs for teams" -- "/usage-credits ... set a monthly spend limit on usage credits"
[^70]: web | 2026-06-18 | [code.claude.com](https://code.claude.com) model-config, "Restrict model selection" -- availableModels / enforceAvailableModels
[^71]: web | 2026-06-18 | [code.claude.com](https://code.claude.com) costs, "Choose the right model" -- "Sonnet handles most coding tasks well and costs less than Opus"
[^72]: web | 2026-06-18 | [code.claude.com](https://code.claude.com) costs, "Write specific prompts"
[^73]: web | 2026-06-18 | [code.claude.com](https://code.claude.com) costs, "Work efficiently on complex tasks" -- plan mode and /rewind
[^74]: web | 2026-06-18 | [code.claude.com](https://code.claude.com) costs, "Reduce token usage" -- context-size strategies
