# The Agentic Engineering Handbook: Claude Code Edition

> An opinionated, primary-sourced reference for senior engineers who want to be genuinely effective with coding agents. Built around **Claude Code**; the discipline travels. Published by **The Discerning Engineer** -- https://discerningengineer.com

This is not a "getting started" guide. It assumes you already know how to craft software and have a coding agent open most of the day. It is the reference you keep in a second pane: what every mechanism does, when a senior reaches for it, where it bites, and the judgment that separates a force-multiplier from a rubber-stamp for AI slop.

## How to read it

Read **Chapter 00** once, cover to cover -- it is the frame the rest hangs on. After that, jump to whatever you need. Every chapter stands alone, opens with a TL;DR, and cross-references the chapters that go deeper.

## The standard it is held to

- **Primary-sourced.** Every factual claim traces to official Anthropic documentation, an Anthropic engineering post, the official changelog, the MCP specification, or, for a practitioner's stated workflow, that practitioner's own writing. No secondary aggregators, no orphan statistics.
- **Verified `2026-06-18`.** Claude Code ships near-daily, so this is a dated snapshot; version-pinned claims are flagged inline. Across an independent editorial review and a 1,449-claim verification pass, every checkable fact was put against its source.
- **Plain ASCII.** Nothing breaks when the text is copied, diffed, or rendered.
- **One voice.** Opinionated, senior-to-senior, written to be read rather than skimmed for keywords.

## Contents

About 120,000 words across 18 chapters, in seven parts.

### Part I -- Foundations
- [00 - The Agentic Engineering Mindset](00_overview_and_mindset.md)
- [01 - Setup and Environment](01_setup_and_environment.md)
- [02 - Context Engineering](02_context_engineering.md)
- [03 - Memory and Instructions](03_memory_and_instructions.md)

### Part II -- Driving the Agent
- [04 - Driving the Agent](04_driving_the_agent.md)
- [05 - Models, Effort and Prompting](05_models_and_prompting.md)
- [06 - Cost, Caching and Limits](06_cost_caching_and_limits.md)

### Part III -- Extending the Agent
- [07 - Agent Skills](07_skills.md)
- [08 - Subagents and Agent Types](08_subagents.md)
- [09 - MCP (Model Context Protocol)](09_mcp.md)
- [10 - Hooks](10_hooks.md)

### Part IV -- Scale
- [11 - Multi-Agent Orchestration and Scale](11_orchestration_and_scale.md)

### Part V -- Trust (the spine)
- [12 - Verification and Review](12_verification_and_review.md)
- [13 - Evals for Coding Agents](13_evals.md)

### Part VI -- The Practice
- [14 - How Real Engineers Actually Work](14_real_world_workflows.md)
- [15 - Leading Team Adoption](15_team_adoption.md)

### Part VII -- Frontier and Reference
- [16 - Security, Sandboxing and the Frontier](16_security_and_frontier.md)
- [17 - Reference: Decision Tables, Commands and Glossary](17_reference.md)

## How it stays current

Claude Code ships near-daily, so the book maintains itself rather than rotting. A Claude Code skill in [`maintenance/`](maintenance/SKILL.md) diffs the official changelog since the last verification date, finds the chapters a change touched, re-verifies them against primary docs, and updates the date. See [MAINTENANCE.md](MAINTENANCE.md).

## License

Licensed under [CC BY 4.0](LICENSE) -- share and adapt with attribution.

## About

Published by **The Discerning Engineer** -- a property for senior engineers on wielding agents with judgment. More at https://discerningengineer.com.
