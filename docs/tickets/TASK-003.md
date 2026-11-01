# TASK-003: Authorization Policies (Novel & Chapter Ownership)

Status: In Review
Priority: High
Created: 2026-07-23 20:00
Request: Create NovelPolicy and ChapterPolicy: a translator may only update/delete their own novels & chapters. Apply role middleware: `pending` cannot access `/dashboard` at all, `translator` can only CRUD their own content, `admin` has full access to `/admin`.

---

## DEV Response

Policies need `Novel`/`Chapter`/`Volume` Eloquent models to type-hint against, which don't exist yet (full CRUD controllers are TASK-007/010/011's scope) — creating minimal models here (fillable, casts, relations per SRS) as a prerequisite, without building controllers/requests for them.

- [x] `Volume` model (belongsTo Novel, hasMany Chapter) — needed as a relation target for Chapter
- [x] `Novel` model (belongsTo User, hasMany Volume/Chapter/NovelAltTitle, belongsToMany Genre/Tag, belongsToMany Creator via `novel_author`/`novel_illustrator`) — also added `NovelAltTitle`, `Creator`, `Genre`, `Tag` minimal models since Novel's relations reference them
- [x] `Chapter` model (belongsTo Novel, belongsTo Volume nullable)
- [x] `NovelPolicy`: `update`/`delete` → only owner (`$user->id === $novel->user_id`); `create` → role is translator
- [x] `ChapterPolicy`: `update`/`delete` → only via parent novel ownership; `create` → role is translator (ownership of parent novel checked at controller/request level since chapter doesn't exist yet)
- [x] `role:{role}` middleware (`EnsureUserHasRole`) — exact-match role gate per SRS route groups; registered as `role` alias in `bootstrap/app.php`
- [x] Wired `Route::middleware(['auth', 'role:translator'])->prefix('dashboard')` and `Route::middleware(['auth', 'role:admin'])->prefix('admin')` group scaffolding in `routes/web.php` (empty groups — routes filled in by their respective feature tickets) so `pending` gets a 403 immediately, not a broken route once those routes land
- [x] Verified: policies auto-discovered by Laravel's Novel→NovelPolicy / Chapter→ChapterPolicy naming convention (no manual registration needed in Laravel 11) — confirmed via `Gate::getPolicyFor()`
- [x] Manual tinker verification against scratch SQLite DB: owner can update/delete own novel & chapter, a different translator is blocked on both, pending is blocked from `create`, translator passes `role:translator` middleware, admin passes `role:admin`, translator is blocked by `role:admin`

---

## QA Response
[QA fills this]

- [ ] test case
