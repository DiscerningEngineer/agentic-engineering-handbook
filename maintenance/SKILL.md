---
name: handbook-maintenance
description: Use to keep The Agentic Engineering Handbook current and self-consistent. Diffs the official Claude Code changelog since the book's last verification date, finds which chapters a feature change touched, re-verifies and updates only those against primary docs at the book's standard, and bumps the date. Also runs a structural-consistency audit (section numbering, chapter headers, cross-references, ASCII) that catches drift a claim-level pass misses, such as a chapter renumber leaving stale in-body section numbers. Run manually or on a schedule.
---

# Handbook Maintenance

You are maintaining **The Agentic Engineering Handbook: Claude Code Edition** (a primary-sourced, opinionated reference for senior engineers using Claude Code). Claude Code ships near-daily, so the book drifts. Your job is to find what changed and fix only what the change touched, holding the book's standard. Do not rewrite chapters that nothing changed.

This skill does two independent jobs. The **Procedure** below keeps the book's *content* current against the changelog. The **structural-consistency audit** after it keeps the book internally *coherent* -- section numbering, chapter headers, cross-references, ASCII. Run the audit on every pass: it needs no network, and claim-level verification does not catch structural drift. A chapter renumber that leaves stale in-body section numbers passes every fact check and still ships broken -- that is the defect that put this section here.

## The book's standard (do not lower it)

- Every factual/technical claim traces to a PRIMARY source: official Anthropic docs (code.claude.com, platform.claude.com), anthropic.com/engineering, official claude.com/blog, modelcontextprotocol.io, or the official changelog. Practitioner workflows/opinions cite that person's OWN writing. No secondary aggregators, no orphan stats. A cited URL must actually support the claim.
- Voice: DevLore hybrid (campfire-philosopher prose; tables/code/attributions preserved). Standard-keyboard ASCII only (`--`, `->`).
- Right-sized, opinionated, searchable, self-contained with correct cross-references.

## Procedure

1. **Find the baseline.** Read `README.md` and take the date from the line `Verified \`YYYY-MM-DD\``. That is LAST_VERIFIED.

2. **Diff the changelog.** WebFetch `https://code.claude.com/docs/en/changelog.md`. Collect every entry dated AFTER LAST_VERIFIED: new features, renamed/removed commands and flags, changed defaults, new/deprecated model names, behavior changes, version pins.

3. **Map changes to chapters.** For each changed item, Grep the chapter files in this folder for the feature/command/model name (and obvious synonyms). The chapters that match are the AFFECTED SET. Also always re-check `17_reference.md` (the command/glossary/decision tables) and `05_models_and_prompting.md` + `06_cost_caching_and_limits.md` when models or pricing changed. If the AFFECTED SET is empty, stop after step 6 (nothing to update but still bump the date and log "no changes").

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

## Structural-consistency audit (run every pass)

Local, fast, no network. Run it every time, whether or not the changelog moved.

1. **Section numbering.** For each chapter file `NN_title.md`, collect the numbered section headers (`##`/`###`/`####` lines beginning with a number) and classify the chapter's scheme:
   - **Chapter-prefixed** -- every numbered `##` shares one leading value `V` (all `## V.1`, `## V.2`, ...). Then `V` MUST equal the file's chapter number `NN`. If `V != NN`, the chapter was renumbered and its body never caught up. Real bug.
   - **Within-chapter** -- the leading numbers run `1, 2, 3, ...` (the chapter number lives only in the H1). Reorder-proof; leave it.
   - **Unnumbered** -- descriptive headers, no numbers. Fine.
   - Anything that fits none of these: flag for a human, do not guess.

   *Fixing a desync (`V != NN`):* in that one file, every `V.<digit>` is a reference to that chapter's own sections -- the headers and every in-body cross-reference, in all its forms: `section V.x`, `Section V.x`, `sections V.x and V.y`, `(see V.x)`, a bare `(V.x)`, three-level `V.x.y`. Re-prefix them all from `V.` to `NN.`. **First scan the file for `V.<digit>` that is NOT a section reference** -- a version pin (`v2.1.V...`), a statistic, a price, a date. If any exist, re-prefix surgically rather than globally. (In the 2026-06 reorder the affected chapters had no such collisions, so a scoped global re-prefix was safe -- verify per run, never assume.)

2. **Chapter headers (H1).** Every chapter H1 should read `# Chapter NN -- Title`: `NN` matching the filename, separator ` -- ` (ASCII). Flag colons (`Chapter 09: ...`), a missing `Chapter NN` prefix, or a number that disagrees with the filename.

3. **Cross-references resolve.** Every `Ch. NN` / `Chapter NN` points to a chapter that exists; every `section X.Y` resolves to a section that exists in the chapter it names; the table of contents in the front matter (`README.md` in the public repo, `handbook.md` in the vault) links to files that exist with matching numbers. A reference to a moved chapter must carry its new number.

4. **ASCII-only.** No curly quotes, em dashes, en dashes, arrows, or ellipsis characters. Use `--` and `->`.

For a deliberate reorder, fix numbering with a scoped, scripted re-prefix and then re-run this audit until it reports zero desync -- never hand-edit a hundred references and hope. Report structural fixes alongside the content report and log them in `maintenance/log.md`.

## Guardrails

- Surgical, not a rewrite. Most weeks touch zero to a few chapters.
- Never introduce a secondary source to "fill a gap." If primary docs do not yet cover a new feature, FLAG it for a human and leave the chapter honest.
- If a change is large enough to alter a chapter's thesis (not just a fact), do not silently rewrite it; report it and propose the rewrite.
- Keep backups: the chapter's prior state is in git; commit before and after if running in a repo.
- Chart/image-locked figures need a browser, not a text fetch. Some facts -- benchmark scores, pricing graphics -- live only inside chart images on the source page, where a text fetch (of the changelog or the page) cannot see them. A text-only diff has a blind spot here. When a changed or suspect claim is a number the source renders in an image, drive a browser (playwright or chrome-devtools), screenshot the chart, and read it with vision before you trust or update it.
- Run the structural-consistency audit every pass, not only when the changelog moved. The reorder that motivated it changed no facts -- every claim still verified against its source -- yet left ten chapters with section numbers one off from their chapter. Structure is a separate failure surface from content.
