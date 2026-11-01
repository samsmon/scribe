# TASK-013: Reader — Novel Detail Page

Status: In Review
Priority: High
Created: 2026-07-23 20:00
Request: The public novel detail page: synopsis, cover, original title + alternative titles, author/illustrator, origin/target language info, status, genres, tags, and the chapter list grouped by volume (chapters without a volume are shown flat). Published chapters display normally (clickable); draft chapters are not shown; on_revision chapters are shown with a "being revised" badge and are not clickable.

---

## DEV Response

Replaces the placeholder `Public\NovelController@show` stubbed in TASK-012. Chapter reading itself (TASK-014) isn't built yet, so — same precedent as TASK-007→012 and TASK-012→013 — adding minimal stub routes/pages for `GET /novels/{slug}/{chapter}` and `GET /novels/{slug}/vol-{vol}/{chapter}` (per SRS's exact route structure) so published-chapter links have somewhere real to go; TASK-014 fills in the actual reading experience and `chapter_reads` tracking.

- [x] `NovelController@show`: eager-load `altTitles`, `authors`, `illustrators`, `genres`, `tags`, and chapters via two separate constrained paths — `volumes.chapters` (ordered, `status != draft`) and top-level `chapters` (volume-less, `status != draft`) — so the frontend gets pre-grouped data instead of grouping client-side
- [x] Chapter visibility rule enforced at the query level, not just UI: `draft` chapters are excluded from the eager-load entirely (never sent to the client, not just hidden by CSS) — a reader inspecting the page source or Inertia's JSON payload cannot see draft content exists
- [x] `on_revision` chapters ARE included (with their status) so the frontend can render the "being revised" badge, but rendered as non-interactive (no `<Link>`, per spec) — the stub read routes reject `on_revision`/`draft` server-side too (404), not just omitted from the UI, so a reader can't reach one by guessing/bookmarking the URL
- [x] Stub routes: `GET /novels/{slug}/{chapter}`, `GET /novels/{slug}/vol-{vol}/{chapter}` → minimal placeholder page, `Public\ChapterController@show`/`showInVolume` resolve by `chapter_number` (+ volume number when present) scoped to the novel, 404s on draft/on_revision/nonexistent/wrong-volume
- [x] `Pages/Novels/Show.tsx` rewritten: metadata, alt titles, authors/illustrators, genre/tag badges, volume-grouped chapter list + flat volume-less chapters, on_revision badge
- [x] Verified via tinker: draft chapter never appears in the eager-loaded payload while on_revision does (with correct status), volume grouping correctly excludes the draft and includes the other two, flat volume-less published chapter present in the separate `chapters` array; direct-URL access to an on_revision or draft chapter both 404 correctly (not just hidden client-side), a published chapter renders, and — an edge case I added while testing — requesting a real published chapter's `chapter_number` but under the *wrong* volume number also 404s instead of resolving by number alone across volumes

---

## QA Response
[QA fills this]

- [ ] test case
