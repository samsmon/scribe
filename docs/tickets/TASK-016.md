# TASK-016: Reader — Continue Reading + Read/Unread Chapter Indicator

Status: In Review
Priority: Medium
Created: 2026-07-23 20:00
Request: The `/continue-reading` page shows novels with the last chapter read per device (query MAX(read_at) per novel from `chapter_reads`). On the novel detail page, each chapter in the list shows a read/unread indicator based on whether a row exists in `chapter_reads` for that device.

---

## DEV Response

Grouping "max read_at per novel" in PHP rather than a SQL window function/subquery — a device's reading history is small, and doing it in PHP keeps the query portable across SQLite (dev) and PostgreSQL (prod) without writing DB-specific SQL, consistent with how search/filter already avoid non-portable constructs where reasonably possible.

- [x] `Public\ContinueReadingController@index` — pulls this device's `chapter_reads` with their chapters, groups by `novel_id`, picks the max-`read_at` entry per group, excludes novels that have gone back to `draft` (same invisibility rule as favorites), sorted by most-recently-read first
- [x] `NovelController@show` extended to also return `readChapterIds` — the subset of chapter IDs on this page that the current device has a `chapter_reads` row for, so the frontend can render a read/unread indicator per chapter row without a second round trip
- [x] `Pages/ContinueReading.tsx` — one row per novel: cover, title, "last read: Ch. X", link resumes at that novel's detail page
- [x] Route: `GET /continue-reading`
- [x] Verified via tinker: with reads scattered across two novels and multiple chapters within one of them, the correct chapter (most recent `read_at`, not the highest chapter number or insertion order) is picked as "last read", and novels correctly sort most-recently-read first; a device with zero read history gets an empty list, not an error; a novel reverted to `draft` disappears from continue-reading despite its read history still existing in the DB; `readChapterIds` on the novel detail page correctly reflects the requesting device's own history and stays empty for a different device that hasn't read anything there

---

## QA Response
[QA fills this]

- [ ] test case
