---
name: handbook-maintenance
description: Use to keep The Agentic Engineering Handbook current. Diffs the official Claude Code changelog since the book's last verification date, finds which chapters a feature change touched, re-verifies and updates only those chapters against primary docs at the book's standard, and bumps the date. Run manually or on a schedule.
---

# Handbook Maintenance

You are maintaining **The Agentic Engineering Handbook: Claude Code Edition** (a primary-sourced, opinionated reference for senior engineers using Claude Code). Claude Code ships near-daily, so the book drifts. Your job is to find what changed and fix only what the change touched, holding the book's standard. Do not rewrite chapters that nothing changed.

## The book's standard (do not lower it)

- Every factual/technical claim traces to a PRIMARY source: official Anthropic docs (code.claude.com, platform.claude.com), anthropic.com/engineering, official claude.com/blog, modelcontextprotocol.io, or the official changelog. Practitioner workflows/opinions cite that person's OWN writing. No secondary aggregators, no orphan stats. A cited URL must actually support the claim.
- Voice: DevLore hybrid (campfire-philosopher prose; tables/code/attributions preserved). Standard-keyboard ASCII only (`--`, `->`).
- Right-sized, opinionated, searchable, self-contained with correct cross-references.

## Procedure

1. **Find the baseline.** Read `README.md` and take the date from the line `Verified \`YYYY-MM-DD\``. That is LAST_VERIFIED.

2. **Diff the changelog.** WebFetch `https://code.claude.com/docs/en/changelog.md`. Collect every entry dated AFTER LAST_VERIFIED: new features, renamed/removed commands and flags, changed defaults, new/deprecated model names, behavior changes, version pins.

3. **Map changes to chapters.** For each changed item, Grep the chapter files in this folder for the feature/command/model name (and obvious synonyms). The chapters that match are the AFFECTED SET. Also always re-check `16_reference.md` (the command/glossary/decision tables) and `05_models_and_prompting.md` + `17_cost_caching_and_limits.md` when models or pricing changed. If the AFFECTED SET is empty, stop after step 6 (nothing to update but still bump the date and log "no changes").

4. **Re-verify and update each affected chapter.** For each chapter in the AFFECTED SET, in its own subagent if available:
   - Read the chapter. Locate every claim about the changed item.
   - WebFetch the relevant PRIMARY doc(s) and confirm current behavior.
   - Update the prose/tables to match, keeping the standard (primary citation with locator detail, voice, ASCII, right-sized). Correct version pins to the current version. If a feature was removed, cut or past-tense it. If renamed, rename and note the old name once.
   - Do NOT touch claims the change did not affect. Surgical edits only.
   - Update the chapter frontmatter `updated:` to today and leave `source:` accurate.

5. **Adversarial check.** For each updated chapter, re-read it once as a skeptic: every changed claim now cites a primary source that supports it; no secondary source crept in; ASCII-only; voice intact. Fix slips.

6. **Stamp and log.**
   - Update the `Verified \`...\`` date in `README.md` to today.
   - Append one line per run to `maintenance/log.md`: the date, the changelog entries handled, and the chapters touched (create the file if missing).

7. **Report** a short summary: changelog entries since LAST_VERIFIED, chapters updated, claims changed, anything that needs a human (a change with no primary doc yet, a removed feature that reshapes a chapter's argument, a pricing/model shift worth a rewrite rather than a patch).

## Guardrails

- Surgical, not a rewrite. Most weeks touch zero to a few chapters.
- Never introduce a secondary source to "fill a gap." If primary docs do not yet cover a new feature, FLAG it for a human and leave the chapter honest.
- If a change is large enough to alter a chapter's thesis (not just a fact), do not silently rewrite it; report it and propose the rewrite.
- Keep backups: the chapter's prior state is in git; commit before and after if running in a repo.
- Chart/image-locked figures need a browser, not a text fetch. Some facts -- benchmark scores, pricing graphics -- live only inside chart images on the source page, where a text fetch (of the changelog or the page) cannot see them. A text-only diff has a blind spot here. When a changed or suspect claim is a number the source renders in an image, drive a browser (playwright or chrome-devtools), screenshot the chart, and read it with vision before you trust or update it.
