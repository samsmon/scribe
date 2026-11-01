# TASK-012: Reader — Browse & Search/Filter Novel

Status: In Review
Priority: High
Created: 2026-07-23 20:00
Request: The public home page (no login): a list of novels from all translators with genre, tag, status filters and search by title (PostgreSQL ILIKE). Only non-draft novels are shown.

---

## DEV Response

Replaces the placeholder `Home.tsx` from TASK-007 (a stub I left specifically for this ticket). Adding pagination even though it's technically P1 in the PRD — Laravel's paginator is nearly free to wire up now and building the unpaginated version first just to redo it later isn't worth it.

- [x] `HomeController@index` (public, no auth) — `Novel::where('status', '!=', 'draft')`, search by title OR any `novel_alt_titles.title` via `ILIKE` (Postgres-only per SRS decision #15, same caveat as TASK-008's creator search — verified match logic with a portable `LOWER()+LIKE` equivalent locally), filter by genre slug / tag slug / status (status filter itself still excludes `draft` — a reader can't use the status filter to peek at drafts), paginated
- [x] Route: `GET /` — replaces the placeholder closure; also added `GET /novels/{slug}` as a minimal placeholder detail page (`Public\NovelController@show`, 404s on draft/unknown slugs) since `NovelCard` needs somewhere to link — same stub-ahead-of-its-ticket precedent as TASK-007's `Home.tsx`; TASK-013 will build out the real detail page
- [x] `Components/NovelCard.tsx`, `Components/FilterBar.tsx` (both named explicitly in SRS's component list)
- [x] `Pages/Home.tsx` rewritten to use them, genre/tag options passed as props for the filter dropdowns
- [x] Verified via tinker: draft excluded from the base query regardless of any filter combination (including trying to abuse the status filter to select `draft` explicitly — still yields 0 due to the base exclusion), search matches title and alt-titles and never surfaces a draft even when its title matches the search term, genre filter isolates correctly, genre+status combine as AND not OR, direct controller invocation confirms the real query builder chain produces the same correct filtered results, draft novel detail page 404s while a published one returns 200

---

## QA Response
[QA fills this]

- [ ] test case
