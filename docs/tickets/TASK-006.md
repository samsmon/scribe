# TASK-006: Admin — Novel/Chapter Moderation & Genre/Tag CRUD

Status: In Review
Priority: Medium
Created: 2026-07-23 20:00
Request: An admin page to list & delete violating novels/chapters (content moderation). Plus CRUD for the genre and tag master data (name, slug) that translators later pick when attaching to a novel (see TASK-009).

---

## DEV Response

Another small SRS omission (same class as the missing volume routes in TASK-010): SRS's Pages list has no `Admin/Chapters.tsx` even though its own Route Structure lists `GET /admin/chapters` + `DELETE /admin/chapters/{id}`. Building the page anyway since the route needs somewhere to render.

Per SRS's route list, genre/tag only have `POST` (create) and `DELETE` — no update route. Following that literally: name/slug are set once at creation, not editable after. If a name typo needs fixing, delete and recreate (acceptable for admin-managed low-churn master data).

- [x] `Admin\NovelModerationController@index`/`@destroy` — lists novels across *all* translators (not scoped to the admin), reuses `NovelService::delete()` so cover-image cleanup logic isn't duplicated; no ownership check needed since admin moderation is exempt from `NovelPolicy` by design (route-level `role:admin` gate is the authorization)
- [x] `Admin\ChapterModerationController@index`/`@destroy` — same pattern, reuses `ChapterService::delete()`
- [x] `Admin\GenreController@index`/`@store`/`@destroy`, `Admin\TagController@index`/`@store`/`@destroy` — name required unique, slug auto-generated from name (same `Str::slug` + uniqueness-suffix pattern as `NovelService::uniqueSlug`); delete relies on the existing `cascadeOnDelete` FK on `novel_genre`/`novel_tag` from TASK-002's migrations so removing a genre/tag cleanly detaches it from any novels using it, no orphaned pivot rows
- [x] Routes: `GET/DELETE /admin/novels[/{id}]`, `GET/DELETE /admin/chapters[/{id}]`, `GET/POST/DELETE /admin/genres[/{id}]`, same for `/admin/tags`
- [x] Pages: `Admin/Novels.tsx`, `Admin/Chapters.tsx`, `Admin/Genres.tsx`, `Admin/Tags.tsx` (Genres/Tags share a `MasterDataList` component since the pattern is identical), nav links added to `AdminLayout`
- [x] Verified via tinker: admin deleted a novel owned by a *different* translator with no ownership check blocking it (confirming moderation genuinely bypasses `NovelPolicy`), the novel's chapter cascaded via the FK, genre create generates a correct slug, attaching a genre to a novel then deleting the genre cleanly removes only the pivot row (novel itself untouched, pivot count back to 0), same for tags

---

## QA Response
[QA fills this]

- [ ] test case
