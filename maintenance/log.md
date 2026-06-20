# Handbook maintenance log

One entry per run: date, what was handled, chapters touched.

## 2026-06-20 -- structural-consistency audit (no changelog diff)

Structural pass only; no content re-verification, so the `Verified` date is unchanged.

Fixed the section-numbering desync left by the 2026-06-19 Cost-and-Caching reorder. That reorder remapped filenames, H1 numbers, TOCs, and cross-chapter `Ch. NN` references, but missed the chapter-prefixed in-body section numbering (`## 11.1`, `section 11.5`, ...). Re-prefixed 148 occurrences -- section headers and intra-chapter `section N.M` references -- to their new chapter numbers across ch06, 07, 08, 09, 11, 12, 13, 15, 16, 17. Also normalized the ch09 and ch15 H1s from `: ` to ` -- `.

The structural audit (added to the skill this run) now reports zero desync across all 18 chapters.
