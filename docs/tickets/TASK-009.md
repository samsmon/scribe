# TASK-009: Attach/Detach Genres & Tags to a Novel (Translator)

Status: In Review
Priority: Medium
Created: 2026-07-23 20:00
Request: UI in the novel form (translator dashboard) to attach/detach genres and tags (pivots novel_genre, novel_tag). Genres and tags themselves are managed by admins (TASK-006); translators only pick from the existing list.

---

## DEV Response

Genre/tag master data CRUD is TASK-006's scope (admin-side, not built yet) — this ticket only needs the translator-facing pick-from-existing-list mechanism, which works regardless of whether the admin UI exists yet (verified by seeding rows directly).

- [x] `NovelService`: `genres()->sync()`/`tags()->sync()` inline in `create`/`update` — plain sync on the pivot, no create-if-not-exist (unlike Creators, these are admin-managed closed lists)
- [x] Extend `StoreNovelRequest`/`UpdateNovelRequest`: `genres`/`tags` arrays of int ids, `exists:genres,id`/`exists:tags,id`
- [x] `NovelController`: `create`/`edit` pass all available `Genre`/`Tag` rows as props (small reference tables, no need for search-as-you-type like Creators); `edit` also passes the novel's currently-selected ids via eager-loaded relations; `store`/`update` pass validated ids to the service
- [x] `Components/GenreTagPicker.tsx` — toggleable badge list against the full option set, no free-text creation
- [x] Wire into `NovelForm.tsx`, `Create.tsx`, `Edit.tsx`
- [x] Verified via tinker (seeding genres/tags directly since TASK-006's admin CRUD doesn't exist yet): attach 2 genres + 1 tag on create, detach one genre and clear all tags on update (pivot rows correctly removed, remaining genre correct), invalid genre id rejected by `StoreNovelRequest` validation, `edit()`'s eager-loaded relation shape matches what the frontend expects

---

## QA Response
[QA fills this]

- [ ] test case
