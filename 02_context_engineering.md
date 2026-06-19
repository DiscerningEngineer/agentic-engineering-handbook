# Chapter 02 -- Context Engineering

**TL;DR.** The context window is the scarcest resource you have, and learning to manage it well is most of what separates the engineers who get clean results from the ones who feel like they are wrestling the tool. Anthropic's governing principle reads backwards at first: more context is not better. As the window fills, the model's recall and reasoning quietly decline, a slide they call context rot, so the job becomes assembling the smallest set of high-signal tokens that maximizes the odds of the outcome you want. [^1] [^2] This chapter walks the diagnostics (`/context`, `/usage`), the levers (`/clear`, `/compact`, `/rewind`), the architectural calls (just-in-time vs. front-loading, 200K vs. 1M), how auto-compaction behaves and what survives it, the API-level equivalents for SDK builders (context editing and server-side compaction), and the senior patterns that keep a long session sharp instead of watching it dull turn over turn: rewind over correct, structured note-taking, offload to a subagent.

> **Version sensitivity.** Claude Code ships near-daily. Command names, default thresholds, and model lineups here are accurate as of 2026-06-18 but drift. Verify version-pinned claims against the changelog (`https://code.claude.com/docs/en/changelog`) and run `claude --version` to confirm what you are on.

---

## 2.1 Why context engineering is *the* discipline

Prompt engineering is the craft of writing one good instruction, particularly the system prompt. Context engineering is the wider craft around it, what Anthropic calls "the set of strategies for curating and maintaining the optimal set of tokens (information) during LLM inference, including all the other information that may land there outside of the prompts." [^1] The distinction earns its keep because an agentic session was never a single prompt. It is hundreds of turns, and each turn re-sends the entire accumulated history: the system prompt, your `CLAUDE.md`, auto memory, every loaded skill, the tool definitions, the MCP tool listing, and the full conversation transcript down to each prior tool call and its raw output. [^3]

You pay for those tokens every turn, and the larger cost is attention, diluted a little further across all of it with each pass. Anthropic frames context engineering as managing "the entire context state (system instructions, tools, Model Context Protocol (MCP), external data, message history, etc)." [^1] The way to hold this is to treat the window as a budget you spend continuously, not a bucket you fill once.

---

## 2.2 Context rot -- the mechanism, not the myth

Context rot is the quality decline that creeps in as the window fills. It is a structural property of how the transformer pays attention, the same in every long-context model, and it cannot be waited out or trained away.

Anthropic's explanation is that language models carry an "attention budget" that they "draw on when parsing large volumes of context. Every new token introduced depletes this budget by some amount." [^1] The reason sits in the architecture: a transformer maintains "n^2 pairwise relationships for n tokens," and as context grows, "a model's ability to capture these pairwise relationships gets stretched thin." [^1] What this produces is gradual rather than sudden, "a performance gradient rather than a hard cliff." [^1] The official context-windows doc states the consequence plainly: "more context isn't automatically better. As token count grows, accuracy and recall degrade, a phenomenon known as *context rot*. This makes curating what's in context just as important as how much space is available." [^2]

Two things follow that are worth carrying with you. The first is that the model cannot tell signal from noise on your behalf. A stale tool output from eighty turns back, a file you read once and never touched again, an MCP server you connected just in case, all of it competes for the same finite attention as the instruction you care about this second. The model weights helpful context and clutter the same way, which makes you the filter. The second is that the slide is silent. You will not get an error at sixty percent fill. You will get subtly worse work, a missed edge case, a forgotten constraint, a hallucinated function signature, and the temptation will be to blame the model when the context was the culprit. Catching that slide early is the whole skill, and the signal to act is qualitative, the work getting sloppier, as much as it is any number on a gauge.

Even the strongest long-context model rewards curation. Anthropic notes that Claude reaches state-of-the-art results on long-context retrieval benchmarks like MRCR and GraphWalks, "but these gains depend on what's in context, not just how much fits." [^2] A clean, curated 200K window beats a cluttered 1M window.

### Context awareness -- the model watches its own budget

The newer models manage some of this themselves. Claude Sonnet 4.6, Sonnet 4.5, and Haiku 4.5 carry "context awareness," which lets them "track their remaining context window (that is, 'token budget') throughout a conversation" and pace accordingly. [^2] The mechanism is out in the open. At the start of a conversation the model receives a budget signal, and after each tool call it receives an update:

```xml
<budget:token_budget>1000000</budget:token_budget>
<system_warning>Token usage: 35000/1000000; 965000 remaining</system_warning>
```
[^2]

Anthropic's image for it is that "for a model, lacking context awareness is like competing in a cooking show without a clock." [^2] It earns its keep most in "long-running agent sessions that require sustained focus" and "multi-context-window workflows where state transitions matter." [^2]

The practical upshot is narrow. Context awareness keeps the model from stopping early and from over-spending, and it leaves your curation work exactly where it was. A model that knows it has 100K tokens of junk left still has to look at the junk.

---

## 2.3 The diagnostic commands: `/context` and `/usage`

You cannot manage what you cannot see. Two commands give you eyes.

### `/context` -- what's eating the window right now

`/context` gives "a live breakdown by category with optimization suggestions," covering everything currently resident in the window. [^3] A surprising amount loads before you type a single word. The startup context alone holds the system prompt, the first 200 lines (or 25KB) of auto memory, the environment block, the MCP tool names, the one-line skill descriptions, your global `~/.claude/CLAUDE.md`, and the project `CLAUDE.md`. [^3] Run `/context` before a long task to see your baseline, and run it again the moment the work starts feeling off.

What you are hunting for:

- **MCP servers you forgot you connected.** MCP tool definitions are deferred by default, so "only tool names enter context until Claude uses a specific tool," but a heavily-used or poorly-scoped server can still dominate. [^4] Run `/mcp` to see per-server costs and disable anything you are not actively using. [^3]
- **A bloated `CLAUDE.md`.** It loads at session start and stays resident the whole session, even during unrelated work. Anthropic's guidance is to keep it under 200 lines, include only essentials, and move specialized workflow instructions into skills that load on demand. [^4] (Full treatment in Ch. 03.)
- **Stale conversation bulk**, the cumulative weight of tool outputs and file reads from earlier in the session.

You can keep usage in view the whole time by configuring the status line to display it, so nothing sneaks up on you. [^4]

### `/usage` -- token spend and attribution

The Session block in `/usage` shows detailed token-usage statistics for the current session, including a cost estimate computed locally from token counts. [^4]

```text
Total cost:            $0.55
Total duration (API):  6m 19.7s
Total duration (wall): 6h 33m 10.2s
Total code changes:    0 lines added, 0 lines removed
```
[^4]

On a Pro, Max, Team, or Enterprise plan, `/usage` also "attributes recent usage to skills, subagents, plugins, and individual MCP servers, with each shown as a percentage of the total." [^4] Press `d` or `w` to switch between the last 24 hours and the last 7 days. That attribution is the fast way to catch a heavy subagent or a misconfigured MCP server. The figures are "approximate and computed from local session history on this machine, so usage from other devices or claude.ai is not included." [^4]

> **On the dollar figure.** For API users it is an estimate and "may differ from your actual bill"; authoritative billing lives in the Claude Console Usage page. For Max and Pro subscribers the session cost figure is not relevant for billing, since usage is included in the subscription; subscribers see plan-usage bars and a usage breakdown instead. [^4] Treat the local estimate as a gauge, not a number to quote.

> **Version note.** The VS Code "Account & usage" dialog with a Day and Week toggle requires Claude Code v2.1.174 or later. [^4]

---

## 2.4 The control commands: `/clear`, `/compact`

### `/clear` -- the hard reset

`/clear` wipes the conversation history and starts fresh, with no recovery of the cleared conversation. Use it whenever you switch to unrelated work, because "stale context wastes tokens on every subsequent message." [^4]

The senior habit makes `/clear` the default move between tasks rather than the last resort. Most engineers lean on it too rarely because they are afraid of losing context they might still want. The fix is to make the session findable before you wipe it: run `/rename` first so you can locate it later, then `/resume` to come back if it turns out you need it. [^4] You get the clean window and a recoverable record in the same move.

The project-root `CLAUDE.md` survives `/clear`, because it is re-read from disk at session start. That is the structural reason persistent rules belong in files. Anything you say only in chat is gone after `/clear` or after compaction, while a file-backed rule comes right back. (More in Ch. 03.)

### `/compact` -- summarize on demand

`/compact` summarizes the conversation history into a condensed form, freeing window space while preserving what matters. It accepts a focus argument, which is the part most people skip:

```text
/compact Focus on code samples and API usage
```

The focus argument "tells Claude what to preserve during summarization." [^4] You can also set standing compaction instructions in `CLAUDE.md`:

```markdown
# Compact instructions

When you are using compact, please focus on test output and code changes
```
[^4]

An un-focused compaction summarizes generically and tends to drop exactly the detail you needed. A focused one, preserve the schema migration decisions and the failing test output, is the difference between a usable post-compaction session and one you have to re-orient from scratch.

**Which lever for which moment:**

- `/clear` -- you are done with this task entirely and nothing forward depends on the history. Cheapest, cleanest.
- `/compact` -- you need to keep going on the same task but the window is filling, and you want the thread of the work preserved in summary.
- `/rewind` (next section) -- a specific recent decision went wrong and you want to undo to a point rather than summarize everything.

---

## 2.5 `/rewind` and checkpoints -- rewind over correct

This earns its own section because it overturns a habit nearly everyone carries over from chat interfaces.

### How checkpointing works

Claude Code "automatically captures the state of your code before each edit." [^5] Specifically:

- Every user prompt creates a new checkpoint.
- Checkpoints persist across sessions, so you can reach them in resumed conversations.
- They are auto-cleaned up along with sessions after 30 days, which is configurable. [^5]

Open the rewind menu with `/rewind`, or press `Esc` twice when the prompt input is empty. If the input contains text, double-`Esc` clears it instead, and the cleared text is saved to your input history so `Up` recalls it. [^5] The menu lists every prompt you sent during the session. Pick a point, then choose:

- **Restore code and conversation** -- revert both to that point.
- **Restore conversation** -- rewind the conversation while keeping current code.
- **Restore code** -- revert file changes while keeping the conversation.
- **Summarize from here** -- compress the selected message and everything after it into a summary, useful to discard a side discussion while keeping early context in full.
- **Summarize up to here** -- compress everything before the selected message, keeping later messages intact, useful to compress setup discussion while keeping recent work detailed. [^5]

The summarize options are a targeted `/compact`. Instead of summarizing the whole conversation, you choose which side of a chosen message to compress, "and the original messages are preserved in the session transcript, so Claude can reference the details if needed." You can type optional instructions to guide what the summary focuses on. [^5]

### The pattern: rewind over correct

When Claude makes a mistake, the instinct is to append a correction: no, that is wrong, do it this way instead. The append works, and it is expensive in context terms. The wrong attempt, your correction, and the model's re-reasoning all stay in the window as noise that dilutes every turn that follows. You made the model dumber to fix one error.

The cleaner move is to rewind to before the mistake and re-prompt with a sharper instruction. The bad attempt never enters the permanent context, and the window stays clean. Anthropic's own cost guidance points the same direction: "Course-correct early: If Claude starts heading the wrong direction, press Escape to stop immediately. Use `/rewind` or double-tap Escape to restore conversation and code to a previous checkpoint." [^4] A corrective append is compounding interest on context debt. A rewind pays down the principal. Over a long session the engineer who rewinds keeps a sharp model, and the engineer who keeps appending corrections watches it go soft.

### The limitation that bites: checkpoints cover edits, not bash side effects

Checkpointing tracks files modified through Claude's file-editing tools, and nothing else. The docs are blunt about it: "Checkpointing does not track files modified by bash commands. For example, if Claude Code runs: `rm file.txt`, `mv old.txt new.txt`, `cp source.txt dest.txt` -- these file modifications cannot be undone through rewind." [^5]

So a `git push`, a `rm -rf`, a database write, a package install, an API call with side effects, none of these come back through rewind. Rewinding the conversation after a destructive bash command leaves the real-world side effect sitting right where it landed. Anthropic's framing is that checkpoints are "local undo" and Git is "permanent history": they complement each other, and one does not replace the other. [^5] External and concurrent-session changes are also normally not captured "unless they happen to modify the same files as the current session." [^5]

The boundary is the point. Rewind is your context-hygiene tool and your code-undo tool, and it is not a safety net for irreversible operations. Anthropic builds a second guardrail in for exactly this reason: actions that affect remote systems "can't be checkpointed, which is why Claude asks before running commands with external side effects." [^6] For the rest, you want deterministic guardrails and real version control (see Ch. 04 for the full treatment of side effects, and the hooks and verification chapters for the guardrails).

### Fork when you want to branch, not rewind

To try a different approach while keeping the original session intact, fork instead of rewinding: `claude --continue --fork-session`. [^5] Rewind and summarize keep you in the same session and mutate it; fork "copies the history into a new session ID, leaving the original unchanged." [^6] Reach for fork when the experiment might fail and you want a clean fallback waiting.

---

## 2.6 Just-in-time vs. front-loading

This is the central architectural decision in context engineering, and Anthropic has a clear default.

Front-loading means putting everything the agent might need into context up front, the whole codebase, all the docs, every file that could matter, before it starts working. It feels thorough. It produces context rot.

Just-in-time retrieval means keeping lightweight identifiers in context and loading the actual data at runtime, only when it becomes relevant. Anthropic: agents "maintain lightweight identifiers (file paths, stored queries, web links, etc.) and use these references to dynamically load data into context at runtime using tools." [^1] This is already how Claude Code works on your codebase, in a hybrid form: "CLAUDE.md files are naively dropped into context up front, while primitives like glob and grep allow it to navigate its environment and retrieve files just-in-time." [^1] The payoff is structural. JIT lets an agent work effectively over a codebase far larger than its window, because the window only ever holds the slice it is actively using.

What this means for how you set up a project:

- **Do not paste a 10,000-line file into chat so Claude has it.** Tell Claude where it is and let it read the relevant parts JIT. Pasting front-loads the whole thing and rots the window.
- **`@`-imports are organization, not token savings.** `@file` imports in `CLAUDE.md` "are expanded and loaded into context at launch," recursively, to "a maximum depth of four hops." [^7] They help you structure your memory across files, and they do not defer cost. To actually defer cost, use mechanisms that load on demand: `paths`-scoped rules in `.claude/rules/`, which "only load into context when Claude works with matching files"; nested `CLAUDE.md` files in subdirectories, which load when Claude reads a file in that subdirectory; and skill bodies, which load when invoked. [^7] (Full treatment in Ch. 03.)
- **Skills are JIT by design.** A skill's name and description sit in context always; for the full body, "the full content only loads when a skill is used." [^6] A 10,000-line reference packaged as a skill costs almost nothing until used, which is the JIT principle operationalized and the reason to move specialized workflow instructions out of `CLAUDE.md` and into skills. [^4]
- **Prefer CLI tools over MCP servers when both exist.** Tools like `gh`, `aws`, `gcloud`, and `sentry-cli` are "more context-efficient than MCP servers because they don't add any per-tool listing." Claude runs them directly as bash. [^4]
- **Offload preprocessing to hooks.** Instead of letting Claude read a 10,000-line log to find errors, a PreToolUse hook can grep for `ERROR` and return only matching lines, "reducing context from tens of thousands of tokens to hundreds." [^4]

The discipline underneath all of it is that every token entering context should earn its place. JIT is the default that makes earning it possible.

---

## 2.7 200K vs. 1M -- when to reach for the big window

### What's available

As of 2026-06-18, Claude Opus 4.8, the Claude Mythos Preview, Opus 4.7, Opus 4.6, and Sonnet 4.6 have a 1M-token context window on the Claude API, Amazon Bedrock, and Vertex AI. Other Claude models, including Sonnet 4.5, have a 200K-token window. [^2] Mind the platform caveat: on Microsoft Foundry, Opus 4.8 has a 200K window, not 1M. [^2] Claude Fable 5 and Claude Mythos 5 also carry a 1M-token window on the Claude API, where "the 1M maximum is also the default," and a single request can generate up to 128K output tokens. [^2]

In Claude Code, the 1M window is reachable through the bracketed aliases, which apply to aliases and full model names alike:

```bash
/model opus[1m]
/model sonnet[1m]
/model claude-opus-4-8[1m]
```
[^8]

Availability varies by plan. On Max, Team, and Enterprise plans, Opus is "automatically upgraded to 1M context with no additional configuration." On the Anthropic API, Fable 5, Opus 4.8, and Opus 4.7 "always run with the 1M window." Sonnet with 1M is not part of the automatic upgrade and "requires usage credits on every subscription plan, including Max." [^8] The 1M window "uses standard model pricing with no premium for tokens beyond 200K." [^8]

> **Lineup and pricing drift.** Model names, context windows, and especially pricing churn fast. Do not memorize per-model numbers; query the live models page (`https://platform.claude.com/docs/en/about-claude/models/overview`) or the Models API at runtime, and verify Haiku-class limits before assuming, since Haiku has historically capped at 200K.

### When to use 1M

Default to 200K for most work. A curated 200K window beats an uncurated 1M window, because context rot applies to both and the bigger window just gives rot more room to operate. Reach for 1M when the task genuinely needs to hold more than fits in 200K even after you have curated it:

- 100-plus-file refactors where the cross-file relationships matter simultaneously.
- Large-codebase or monorepo onboarding that needs broad cross-package understanding.
- Long research or analysis passes over large document sets.

Even then the JIT principle still applies and you should compact proactively. 1M is a bigger budget, and it is still a budget.

### A subtlety worth knowing: thinking tokens and the window

With extended and adaptive thinking, all input and output tokens, thinking included, count toward the window during generation. Previous turns' thinking blocks, though, "are automatically stripped from the context window calculation by the Claude API and are not part of the conversation history that the model 'sees' for subsequent turns." [^2] You do not pay for old thinking on future turns, and you do not have to strip the blocks yourself; the API does it when you pass them back. The one exception comes during a tool-use cycle, where the thinking block that accompanies a tool request "must be returned with the corresponding tool results. This is the only case wherein you have to return thinking blocks." [^2] The API uses cryptographic signatures to verify thinking-block authenticity, and "if you modify thinking blocks, the API returns an error." [^2]

### Overflow behavior

On Claude 4.5 models and newer, if input plus `max_tokens` exceeds the window, "the API accepts the request. If generation then reaches the context window limit, it stops with `stop_reason: 'model_context_window_exceeded'`." On earlier models the API returns a validation error instead, and you opt into the newer behavior with the `model-context-window-exceeded-2025-08-26` beta header. [^2] If you are building on the API, handle this stop reason distinctly from `max_tokens`. It means context exhausted, not output cap hit.

---

## 2.8 Auto-compaction and recovery

You will not always compact by hand. Claude Code compacts automatically as you approach the limit, "so a full context window doesn't end your session," and the automatic pass works the same way as a manual `/compact`. [^3]

### What auto-compaction does

The mechanism is a two-stage shed. "Claude Code manages context automatically as you approach the limit. It clears older tool outputs first, then summarizes the conversation if needed." [^6] The first stage is the cheapest thing to let go of, the raw output of a tool call buried deep in history, which is rarely wanted again. The same logic shows up at the API level as tool-result clearing (section 2.12). The second stage replaces the verbatim history with a structured summary. The summary "keeps: your requests and intent, key technical concepts, files examined or modified with important code snippets, errors and how they were fixed, pending tasks, and current work," and it discards "full tool outputs and intermediate reasoning." [^3] Anthropic's engineering writeup describes the same priorities: compaction "preserves architectural decisions, unresolved bugs, and implementation details while discarding redundant tool outputs or messages," on the logic that "once a tool has been called deep in the message history, why would the agent need to see the raw result again?" [^1]

### What survives compaction and what does not

Summaries are lossy. The cleanest way to predict what comes back is to know how each piece was loaded, because compaction summarizes message history and re-injects file-backed startup content from disk. [^3]

| Mechanism | After compaction |
|---|---|
| System prompt and output style | Unchanged; not part of message history |
| Project-root `CLAUDE.md` and unscoped rules | Re-injected from disk |
| Auto memory | Re-injected from disk |
| Rules with `paths:` frontmatter | Lost until a matching file is read again |
| Nested `CLAUDE.md` in subdirectories | Lost until a file in that subdirectory is read again |
| Invoked skill bodies | Re-injected, capped at 5,000 tokens per skill and 25,000 total; oldest dropped first |
| Hooks | Not applicable; hooks run as code, not context |

[^3]

The line to take from the table: anything you said only in conversation is at risk, and anything that lives in a project-root file is not. Because skill bodies are truncated to fit the per-skill cap and "truncation keeps the start of the file," put the most important instructions near the top of `SKILL.md`. [^3]

### Recovery posture -- how to not lose your work to compaction

1. **Persist standing rules to files, not chat.** Project-root `CLAUDE.md` and unscoped rules survive compaction because they are re-read from disk; conversation-only instructions do not. [^3]
2. **Use structured note-taking** (section 2.10) so progress lives outside the window. After compaction, the note is still on disk.
3. **Add focused compaction instructions** in `CLAUDE.md`, or run `/compact` with a focus before a long new task, "so the summary keeps what you choose instead of what the automatic pass guesses is important." [^3]
4. **When the felt quality dips, prefer a clean `/clear` plus a good restart note over riding a heavily-compacted session.** The decline is a gradient, so the work tells you before any gauge does.

> **The thrash guard.** If a single file or tool output is so large that context refills immediately after each summary, "Claude Code stops auto-compacting after a few attempts and shows an error instead of looping." [^6] The fix is structural: split the oversized file or defer it behind a skill so it loads JIT instead of dominating the window. See the troubleshooting page (`/en/troubleshooting`) for the recovery steps.

---

## 2.9 The three long-horizon techniques (Anthropic's framework)

Anthropic names three techniques for keeping agents productive across long horizons. Every one of them moves information out of the active window so the window stays high-signal. [^1] Compaction was section 2.8; the next two sections cover note-taking and subagents.

| Technique | Mechanism | When |
|---|---|---|
| **Compaction** | Summarize history nearing the limit and reinitialize from the summary, preserving decisions, bugs, and details while discarding redundant tool outputs. [^9] | Conversation approaching the window limit. |
| **Structured note-taking** | Agent writes "notes persisted to memory outside of the context window," which "get pulled back into the context window at later times." [^9] | Multi-step or multi-session work where state must survive. |
| **Sub-agent architectures** | A subagent does context-heavy work in its own window and returns a distilled summary. It "might explore extensively, using tens of thousands of tokens or more, but returns only a condensed, distilled summary of its work (often 1,000-2,000 tokens)." [^9] | Verbose, bounded sub-tasks (search, test runs, log scans). |

---

## 2.10 Structured note-taking -- state that survives the window

The pattern is to have the agent persist progress to files, a `progress.md`, a `tasks.json`, a scratchpad, so the state of the work lives on disk rather than in the conversation. Those notes "get pulled back into the context window at later times," letting the agent "track progress across complex tasks." [^1]

Here is why it is load-bearing. Compaction and `/clear` both destroy conversation state, and neither one touches files. A note on disk is the single form of memory immune to context rot, to compaction loss, and to session boundaries. For agents that span sessions, Anthropic's guidance is direct: "design your state artifacts so that context recovery is fast when a new session starts." [^2]

In practice, for a senior, that is a few concrete moves. For a multi-step implementation, instruct Claude to maintain a `PLAN.md` or `progress.md` with the task list, the decisions made, and the current status, and to keep updating it as it goes. Before a `/clear` or an anticipated compaction, have Claude write a where-we-are note so the next window rehydrates in one read instead of re-deriving everything from scratch. This is the manual version of what auto-compaction's summary does for you, except you control the format and fidelity, which is what makes recovery reliable rather than best-effort.

This runs straight into the second-brain philosophy. Durable state belongs in files, where it outlives the ephemerality of any single session. A long agentic task is a small knowledge system in its own right, and its state deserves the same the-system-is-the-files discipline.

---

## 2.11 Offload to subagents -- context isolation as the point

A subagent gets "its own fresh context, completely separate from your main conversation," along with its own system prompt and tool set. "Their work doesn't bloat your context. When done, they return a summary." [^6] The verbose middle, the dozens of tool calls, the raw test output, the log scan, never enters the parent's window. Anthropic's image is that a subagent "might explore extensively, using tens of thousands of tokens or more, but returns only a condensed, distilled summary of its work." [^1]

Context isolation here is the whole feature. The official cost guidance: "Running tests, fetching documentation, or processing log files can consume significant context. Delegate these to subagents so the verbose output stays in the subagent's context while only a summary returns to your main conversation." [^4]

What to offload:

- **Test runs**, where the full output stays in the subagent and you get "12 passed, 1 failed: `test_auth_expiry`" back.
- **Documentation fetches and web research**, where the raw pages stay isolated and you get the synthesized answer.
- **Log scans and large-file processing**, where the bulk stays out and you get the findings.
- **First-pass exploration**, where the built-in Explore agent scans a codebase and reports back without flooding your window. (The Explore and Plan agents even skip loading the project `CLAUDE.md` to keep their own context smaller. [^3])

There is a cost to keep in view. Each subagent runs its own context window, so token usage scales with how many run and how long. For coordination-heavy agent teams, the docs note "approximately 7x more tokens than standard sessions when teammates run in plan mode, because each teammate maintains its own context window and runs as a separate Claude instance." [^4] Anthropic recommends Sonnet for teammates as the balance of capability and cost, and agent teams stay off by default behind `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`. [^4] (Subagent mechanics and model routing get the full treatment in Ch. 07; the multi-agent uplift figures live in Ch. 10. Here the point is the context benefit, the isolation that keeps the parent window clean.)

The decision rule is one line: if a task will generate a lot of tokens you will not need again, do it in a subagent. The parent should hold conclusions, not transcripts.

---

## 2.12 API-level context engineering (for SDK builders)

If you are building on the Claude API or the Agent SDK rather than, or in addition to, using Claude Code interactively, the same principles have direct API analogues. Two mechanisms matter.

### Context editing -- prune stale content server-side

Context editing clears specific content from history server-side, before the prompt reaches the model. It requires the beta header `anthropic-beta: context-management-2025-06-27` and "is available on all supported Claude models." [^10]

**Tool-result clearing -- `clear_tool_uses_20250919`.** This "clears tool results when conversation context grows beyond your configured threshold," replacing each cleared result "with placeholder text so Claude knows it was removed." By default it clears only results, not the tool calls themselves. [^10] Config parameters:

- `trigger` -- when to activate. Default 100,000 input tokens; format `{"type": "input_tokens", "value": N}` or `{"type": "tool_uses", "value": N}`.
- `keep` -- how many recent tool use/result pairs to preserve. Default 3 tool uses.
- `clear_at_least` -- the minimum tokens to clear each time, which "helps determine if context clearing is worth breaking your prompt cache."
- `exclude_tools` -- tool names to never clear, for example `["web_search"]`.
- `clear_tool_inputs` -- if `true`, also clears tool call parameters. Default `false`. [^10]

```python
response = client.beta.messages.create(
    model="claude-opus-4-8",
    max_tokens=4096,
    messages=[...],
    tools=[...],
    betas=["context-management-2025-06-27"],
    context_management={
        "edits": [
            {
                "type": "clear_tool_uses_20250919",
                "trigger": {"type": "input_tokens", "value": 30000},
                "keep": {"type": "tool_uses", "value": 3},
                "clear_at_least": {"type": "input_tokens", "value": 5000},
                "exclude_tools": ["web_search"],
                "clear_tool_inputs": False,
            }
        ]
    },
)
```
[^10]

**Thinking-block clearing -- `clear_thinking_20251015`.** This manages `thinking` blocks when extended thinking is enabled. The `keep` parameter is `"all"` or `{"type": "thinking_turns", "value": N}` with N greater than 0, and defaults are model-specific (Opus 4.5-plus and Sonnet 4.6-plus keep all prior thinking by default; earlier models keep only the last turn). When combining both strategies, "the `clear_thinking_20251015` strategy must be listed first in the `edits` array." [^10]

**Prompt-cache interaction.** Tool-result clearing "invalidates cached prompt prefixes when content is cleared," which is exactly why `clear_at_least` exists, to make sure each clear is worth the cache write. Thinking-block clearing "preserves the cache when blocks are kept" and "invalidates it at the point where clearing occurs." [^10] The response carries a `context_management.applied_edits` field reporting exactly what was cleared, and for streaming it arrives "in the final `message_delta` event." [^10] You can preview savings before committing through the token-counting endpoint with the same `context_management` config; it returns `original_input_tokens` alongside the post-clearing `input_tokens`. [^10]

> **Header vs. strategy type, a common confusion.** The beta header is `context-management-2025-06-27`. The strategy type strings (`clear_tool_uses_20250919`, `clear_thinking_20251015`) are values inside the `context_management.edits` array. They are different things.

### Server-side compaction -- for conversations past the window

For long-running conversations that may exceed the 1M window, the API offers server-side compaction behind the beta header `compact-2026-01-12`, supported on Fable 5, Mythos 5, the Claude Mythos Preview, Opus 4.8, Opus 4.7, Opus 4.6, and Sonnet 4.6. [^11] The API automatically summarizes earlier context as it nears the threshold and returns a `compaction` block in the response.

The one handling rule that matters: append the full `response.content`, not just the extracted text, back to your `messages` on every turn. "You must pass the `compaction` block back to the API on subsequent requests to continue the conversation with the shortened prompt." [^11] Extracting only the text string silently drops the compaction state.

```python
response = client.beta.messages.create(
    betas=["compact-2026-01-12"],
    model="claude-opus-4-8",
    max_tokens=4096,
    messages=messages,
    context_management={"edits": [{"type": "compact_20260112"}]},
)
# Append the response (including any compaction block) to continue:
messages.append({"role": "assistant", "content": response.content})
```
[^11]

**Editing vs. compaction, which to reach for.** Context editing prunes (it removes stale tool results and thinking while keeping the structure) and operates within a session; compaction summarizes when you near the limit. They compose: many long-running agents use editing to keep the transcript lean turn to turn and compaction as the backstop when the window fills. For state that must survive across sessions, neither suffices, and that is what the memory tool and file-based notes are for. [^2]

---

## 2.13 Senior decision tables

**Which command for which situation:**

| Situation | Reach for |
|---|---|
| Switching to unrelated work | `/clear` (after `/rename` so it is recoverable) |
| Same task, window filling, want the thread preserved | `/compact` with a focus argument |
| A recent decision went wrong; undo to a point | `/rewind` -> Restore code and/or conversation |
| Want to discard a side discussion but keep early context | `/rewind` -> Summarize from here |
| Want to compress setup but keep recent work detailed | `/rewind` -> Summarize up to here |
| Try a different approach without losing the original | Fork (`claude --continue --fork-session`) |
| Diagnose what is eating the window | `/context` |
| See token spend and per-component attribution | `/usage` |

**Which context strategy for which problem:**

| Problem | Strategy |
|---|---|
| Window rotting mid-session | Curate: `/clear`, `/compact`, or `/rewind`; check `/context` for bloat |
| State must survive `/clear`, compaction, or a new session | Structured note-taking to files |
| A sub-task will generate tokens you will not reuse | Offload to a subagent |
| Codebase bigger than the window | JIT retrieval (let Claude `glob`/`grep`/read on demand) |
| Verbose log or test output flooding context | PreToolUse hook to pre-filter, or a subagent |
| Standing rules keep getting lost | Move them to project-root `CLAUDE.md` (files survive) |
| Big task genuinely exceeds curated 200K | `opus[1m]` / `sonnet[1m]`, and still compact proactively |
| Building on the API: stale tool results | Context editing (`clear_tool_uses_20250919`) |
| Building on the API: conversation past the window | Server-side compaction (`compact-2026-01-12`); preserve `response.content` |

---

## 2.14 The mental model to walk away with

1. **The window is a budget spent every turn, not a bucket filled once.** Every token competes for finite attention on every pass.
2. **More context is not better; curated context is better.** Context rot is a gradient, so the work degrades silently before any error shows up.
3. **Diagnose with `/context` and `/usage`; act with `/clear`, `/compact`, `/rewind`.**
4. **Rewind over correct.** Undo to before a mistake instead of appending corrections that pollute the window, and remember rewind does not undo bash side effects.
5. **Default to just-in-time.** Keep identifiers in context, load data on demand, and let Claude navigate large codebases through `glob`, `grep`, and read.
6. **Default to 200K; reach for 1M only when curated 200K genuinely is not enough.**
7. **Move durable state into files.** Notes, plans, and project-root rules on disk survive compaction, `/clear`, and session boundaries. It is the one memory immune to context rot.
8. **Offload verbose work to subagents** so conclusions, not transcripts, return to the parent.
9. **On the API the same principles are first-class:** context editing prunes, server-side compaction summarizes, the memory tool persists.

The through-line is that a great agentic engineer is, more than anything, a curator of attention. The model brings throughput and recall. You bring the judgment about what deserves to be in front of it.

---

## Sources

- Anthropic Engineering -- *Effective context engineering for AI agents.* [^1]
- Claude platform docs -- *Context windows.* [^2]
- Claude platform docs -- *Context editing.* [^10]
- Claude platform docs -- *Compaction* (server-side, `compact-2026-01-12`). [^11]
- Claude Code docs -- *Manage costs effectively* (`/context`, `/usage`, `/clear`, `/compact`, JIT, hooks, subagents, agent-team costs). [^4]
- Claude Code docs -- *Explore the context window* (startup breakdown, what survives compaction, auto-compaction summary). [^3]
- Claude Code docs -- *How Claude Code works* (auto-compaction two-stage shed, thrash guard, subagent isolation, fork). [^6]
- Claude Code docs -- *Checkpointing* (`/rewind`, summarize-from/up-to-here, bash limitation). [^5]
- Claude Code docs -- *Model configuration* (`opus[1m]`/`sonnet[1m]` aliases, extended-context availability by plan). [^8]
- Claude Code docs -- *Memory* (`@`-imports load in full at launch, four-hop depth; `.claude/rules/` `paths` scoping). [^7]

[^1]: web | 2026-06-18 | [https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
[^2]: web | 2026-06-18 | [https://platform.claude.com/docs/en/build-with-claude/context-windows](https://platform.claude.com/docs/en/build-with-claude/context-windows)
[^3]: web | 2026-06-18 | [https://code.claude.com/docs/en/context-window](https://code.claude.com/docs/en/context-window)
[^4]: web | 2026-06-18 | [https://code.claude.com/docs/en/costs](https://code.claude.com/docs/en/costs)
[^5]: web | 2026-06-18 | [https://code.claude.com/docs/en/checkpointing](https://code.claude.com/docs/en/checkpointing)
[^6]: web | 2026-06-18 | [https://code.claude.com/docs/en/how-claude-code-works](https://code.claude.com/docs/en/how-claude-code-works)
[^7]: web | 2026-06-18 | [https://code.claude.com/docs/en/memory](https://code.claude.com/docs/en/memory)
[^8]: web | 2026-06-18 | [https://code.claude.com/docs/en/model-config](https://code.claude.com/docs/en/model-config)
[^9]: web \ | 2026-06-18 \ | [https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
[^10]: web | 2026-06-18 | [https://platform.claude.com/docs/en/build-with-claude/context-editing](https://platform.claude.com/docs/en/build-with-claude/context-editing)
[^11]: web | 2026-06-18 | [https://platform.claude.com/docs/en/build-with-claude/compaction](https://platform.claude.com/docs/en/build-with-claude/compaction)
