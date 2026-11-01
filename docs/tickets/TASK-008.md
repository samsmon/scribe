# TASK-008: Author & Illustrator Autocomplete (Creators)

Status: In Review
Priority: Medium
Created: 2026-07-23 20:00
Request: A search-as-you-type endpoint `/dashboard/creators/search` to look up existing `creators` entities by name. In the novel form, a translator can pick an author/illustrator from the autocomplete results or create a new creator if missing (create-if-not-exist, prevent duplicate names). A novel can have more than one author/illustrator.

---

## DEV Response

Extends the Novel form built in TASK-007. Create-if-not-exist happens on novel save (not on search), so no separate "create creator" endpoint — the search endpoint only reads, `NovelService` resolves names to creators (existing or new) when the novel is saved, deduped case-insensitively.

- [x] `Dashboard\CreatorController@search` — `GET /dashboard/creators/search?q=`, ILIKE match on name, limit 10
- [x] `NovelService::syncCreators()` — given arrays of `{id: number|null, name: string}` for authors/illustrators, find-or-create by name (case-insensitive, via `LOWER(name) =`, DB-portable) then sync `novel_author`/`novel_illustrator` pivots
- [x] Extend `StoreNovelRequest`/`UpdateNovelRequest`: `authors`/`illustrators` arrays of `{id: nullable int, name: required string}`
- [x] Wire into `NovelController::store`/`update`
- [x] `Components/CreatorAutocomplete.tsx` — debounced (250ms) search-as-you-type, multi-select chips, "create new" option when no exact match
- [x] Wire into `NovelForm.tsx` (author + illustrator fields), pass through `Create.tsx`/`Edit.tsx`; `NovelController::edit` loads `authors`/`illustrators` relations
- [x] Verified via tinker: same name (different casing) across two novels reuses the same `Creator` row rather than duplicating; multiple authors per novel; editing to remove authors clears pivot rows but leaves the `Creator` row intact (other novels may still reference it); `Dashboard\Novels\Edit` correctly reflects existing authors/illustrators on load

**Note for QA:** the search endpoint uses `ILIKE`, which is Postgres-only (per SRS's explicit decision to use `ILIKE` for search, no Elasticsearch for MVP) — it throws a syntax error against this dev sandbox's SQLite DB. Confirmed the underlying case-insensitive partial-match logic is correct by running the equivalent query with `LOWER(name) LIKE`, but the actual `search()` endpoint itself needs to be exercised against a real Postgres instance before sign-off, since that's untested here.

---

## QA Response
[QA fills this]

- [ ] test case
