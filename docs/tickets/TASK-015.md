# TASK-015: Reader — Favorites (Device-Based)

Status: In Review
Priority: Medium
Created: 2026-07-23 20:00
Request: A reader (no account) can favorite/unfavorite a novel via POST/DELETE `/favorites`, keyed by `device_id`. UNIQUE(device_id, novel_id) prevents duplicates. The `/favorites` page shows the list of novels favorited by that device.

---

## DEV Response

Uses the `EnsureDeviceId` middleware/cookie built in TASK-014 — no new device-identity mechanism needed.

- [x] `Favorite` model (minimal, fillable `device_id`/`novel_id`)
- [x] `StoreFavoriteRequest`: `novel_id` required, must exist AND `status != draft` — a reader can't reach a draft novel's ID through any UI, but this closes the direct-API-abuse path (same defense-in-depth pattern as blocking draft/on_revision chapter access server-side in TASK-013/014)
- [x] `Public\FavoriteController@store` — `Favorite::firstOrCreate(['device_id', 'novel_id'])`, not plain `create()`: double-clicking "favorite" (or a retried request) must be idempotent, not throw a unique-constraint violation
- [x] `Public\FavoriteController@destroy` — deletes by `device_id` + `novel_id`, silently no-ops if it wasn't favorited (idempotent unfavorite too)
- [x] `Public\FavoriteController@index` — lists the current device's favorited novels; if a novel was favorited while public and later set back to `draft` by its translator, the favorite row is kept (so it reappears if republished) but the novel is excluded from what's *displayed* — same invisibility rule as the browse page, not a special case
- [x] `Components/FavoriteButton.tsx` (named in SRS's component list) — toggles via Inertia, wired into the novel detail page (`Public\NovelController@show` now also returns `isFavorited` for the current device)
- [x] Routes: `POST /favorites`, `DELETE /favorites/{novel}`, `GET /favorites`
- [x] Verified via tinker: double-favorite idempotent (1 row), double-unfavorite idempotent (no error), two different devices favoriting the same novel are correctly independent, a since-unpublished favorited novel is hidden from the list while its favorite row survives underneath

**Bug found and fixed during verification:** the draft-exclusion validation (`Rule::exists('novels', 'id')->where('status', '!=', 'draft')`) silently did nothing — `Rule::exists()->where()` only supports 2-arg column=value equality, not a 3-arg operator form. Passing `('status', '!=', 'draft')` bound `$column='status'` and `$value='!='`, so it was actually checking for a novel with `status` literally equal to the string `"!="` (never true, but also never triggered for the wrong reason). Fixed with the closure form: `->where(fn ($query) => $query->where('status', '!=', 'draft'))`. Confirmed no other validation rule in the codebase has the same 3-arg misuse (grepped for the pattern). Re-verified: draft novels are now correctly rejected, valid novels still pass.

---

## QA Response
[QA fills this]

- [ ] test case
