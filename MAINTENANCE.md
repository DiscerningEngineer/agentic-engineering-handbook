# Maintaining the Handbook

Claude Code ships near-daily, so this book is a dated snapshot (see the "Verified" date in the README) that keeps itself current rather than rotting.

## How

A Claude Code skill ([maintenance/SKILL.md](maintenance/SKILL.md)) does the work:

1. Reads the last verification date.
2. Diffs the official changelog for everything after that date.
3. Finds which chapters reference the changed features.
4. Re-verifies and surgically updates only those chapters against primary docs, holding the book's standard (primary-sourced, in-voice, ASCII).
5. Bumps the verification date and logs the run.
6. Runs a structural-consistency audit every pass (section numbering, chapter-header format, cross-reference resolution, ASCII), independent of the changelog. A claim-level check does not catch structural drift -- a chapter renumber that leaves stale in-body section numbers passes every fact check and still ships broken.

It is surgical by design -- most runs touch a handful of chapters. It never lowers the standard to fill a gap: if primary docs do not yet cover a new feature, it flags it instead of reaching for a blog. Chart or image-locked figures (benchmark scores, pricing graphics) get a browser-and-vision check, because a text-only diff cannot see them.

## Running it

Activate the skill by copying `maintenance/SKILL.md` into `.claude/skills/handbook-maintenance/`, then run `/handbook-maintenance` manually, or schedule it (for example weekly via `/schedule`) to open a pull request with the changes.
