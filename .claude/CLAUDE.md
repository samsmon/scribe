# CLAUDE.md - Scribe (Light Novel Publishing CMS)

## Project Overview

Scribe is one of the projects under the White Archive umbrella (not the main White Archive platform itself: the root domain `whitearchive.id` is the ecosystem landing page, outside the scope of this repo). Scribe is a centralized light novel publishing CMS (not a per-translator subdomain) for freelance translators and readers, living at `scribe.whitearchive.id`.
Translators & admins log in via SSO whitearchive.id (the identity provider shared by all White Archive projects). Readers are fully anonymous, with no account, and tracking is device-based.

## Repo Structure

```
/
  backend/    Laravel 11 + Inertia.js (monolith: the frontend is served from here)
  docs/       PRD, SRS, STRUCTURE, TODO, tickets
  scripts/    deploy.sh, update.sh
  logs/       sync and deploy logs (gitignored)
```

---

<!-- STACK_START -->
## Stack (auto-synced from SRS.md)

- Backend: Laravel 11
- Database: PostgreSQL
- Frontend: Inertia.js + React
- UI: Shadcn UI
- Rich Text / Blog Editor: Tiptap (chapter content, supports inline images)
- Auth: SSO whitearchive.id (OAuth2 Authorization Code + PKCE) — no local password
- Storage: Local disk (MVP) → S3/R2 (later) via Laravel Filesystem
- Domain (staging): scribe.suryatmaja.dev
- Domain (production, planned): scribe.whitearchive.id
<!-- STACK_END -->

---

## Roles

| Role | Source | Description |
|------|--------|-----------|
| Reader | — (anonymous) | No login. Browse, read, favorite, continue reading: all device-based via a cookie/localStorage `device_id`. |
| `pending` | SSO `user` (first login) | Login succeeded but no dashboard access yet. Waiting for an admin grant. |
| `translator` | Local Scribe (granted by an admin) | Can manage their own novels & chapters. |
| `admin` | SSO `superadmin` (auto-mapped) | Full access to Scribe: admin dashboard, role management, moderation. |

**SSO role ≠ Scribe role.** SSO only has `user`/`superadmin`. Scribe keeps its own local role in the `users.role` column (`pending`/`translator`/`admin`), separate from `users.sso_role`, which is only synced as a reference.

---

## Auth: SSO Integration

Must use OAuth2 Authorization Code + PKCE against `sso.whitearchive.id`. Full reference: `AI_INTEGRATION.md` in the `sso.whitearchive` repo.

### Env Vars (backend/.env)

```
SSO_CLIENT_ID=xxx
SSO_CLIENT_SECRET=xxx
SSO_REDIRECT_URI=https://scribe.suryatmaja.dev/auth/callback
SSO_BASE_URL=https://sso.whitearchive.id
```

### Constraints

- PKCE is mandatory: there is no fallback auth method.
- The `state` parameter is validated server-side (session), not via cookie/localStorage.
- The access token (60 minutes) & refresh token (30 days, single-use) are stored **encrypted** on the server and never sent to the client.
- The refresh token is rotated on every use.
- **Logout is mandatory 2-step**: (1) delete the local session, (2) redirect to `SSO_BASE_URL/logout` with `redirect_uri`. Skipping step 2 leaves the SSO session alive, which causes auto-login on the next attempt.
- Profile sync (`name`, `username`, `email`, `sso_role`) happens on every login: do not let local data go stale.
- Avatars from SSO are **not available yet**: use a default placeholder in the UI. Do not build an avatar upload/edit feature until there is a follow-up briefing.
- Admins **must not** be able to edit a user's name/email/avatar in Scribe: that is SSO's domain. Admins may only change the local `users.role`.

---

## Backend (Laravel + Inertia)

### Constraints

- Inertia.js is the bridge between Laravel and React. Controllers return `Inertia::render()`.
- Authorization must go through Laravel Policies: translators may only modify their own novels/chapters.
- There is no publish-approval flow: translators publish novels/chapters directly (centralized platform, not a per-translator subdomain).
- Cover images are stored on local disk (MVP) via Laravel Filesystem, with relative paths.
- Chapter content is saved from Tiptap (markdown/JSON) and supports inline images.
- Chapters have autosave: the `PATCH .../chapters/{id}/autosave` endpoint updates `content` + `last_autosaved_at` without changing `status`.
- `chapter_number` pakai `decimal(8,1)` — supports fractional numbers (1.5) for side stories/interludes.
- `volume_id` in the chapters table is nullable: volumes are optional.
- Authors & illustrators are stored as entities (`creators`), not free text: resolved via autocomplete, create-if-not-exist.
- Reader data (favorites, chapter_reads) is keyed by `device_id` and is never tied to `users`.

### Commands

```bash
cd backend
composer install
php artisan migrate
php artisan serve
npm install
npm run dev
php artisan test
```

### Core Models & Relations

- `User` — hasMany `Novel` (as translator), hasMany `Ticket` (as target or sender)
- `Novel` — belongsTo `User`, hasMany `Volume`, hasMany `Chapter`, hasMany `NovelAltTitle`, belongsToMany `Genre`, belongsToMany `Tag`, belongsToMany `Creator` (via `novel_author` and `novel_illustrator`: two different relations to the same `creators` table)
- `Volume` — belongsTo `Novel`, hasMany `Chapter`
- `Chapter` — belongsTo `Novel`, belongsTo `Volume` (nullable)
- `Creator` — belongsToMany `Novel` (as author or illustrator)
- `Genre` / `Tag` — belongsToMany `Novel`
- `Favorite` / `ChapterRead` — no relation to `User`, just a `device_id` string + FK to `Novel`/`Chapter`
- `Ticket` — belongsTo `User` (nullable, when `from_type`/`to_type` = translator/superadmin)

### Route Groups

```php
// routes/web.php
Route::get('/', ...)                                  // public
Route::get('/novels/{slug}', ...)
Route::get('/novels/{slug}/{chapter}', ...)
Route::get('/novels/{slug}/vol-{vol}/{chapter}', ...)
Route::get('/translator/{username}', ...)
Route::post('/favorites', ...);
Route::delete('/favorites/{novel}', ...);
Route::get('/continue-reading', ...);
Route::post('/tickets', ...);                          // reader submits a ticket

Route::get('/auth/login', ...);
Route::get('/auth/callback', ...);
Route::post('/auth/logout', ...);

Route::middleware(['auth', 'role:translator'])->prefix('dashboard')->group(function () {
    // the translator's own novel & chapter CRUD, autosave, creators autocomplete
});

Route::middleware(['auth', 'role:admin'])->prefix('admin')->group(function () {
    // dashboard stats, user role management, moderation, genre/tag CRUD, tickets
});
```

### Authorization Pattern

```php
class NovelPolicy
{
    public function update(User $user, Novel $novel): bool
    {
        return $user->id === $novel->user_id;
    }
}
```

---

## Frontend (Inertia + React)

### Constraints

- Inertia.js: use `<Link>` and `router` from `@inertiajs/react`, not manual fetch/axios for page navigation.
- Shadcn UI is the component library.
- TypeScript strict mode.
- Tiptap for the chapter editor: a blog-writing-like experience, supports inline images mid-text, periodic (debounced) autosave.
- The reader `device_id` is generated & stored in a cookie/localStorage on first access and sent to the server for favorites & chapter_reads: there is no login UI for readers at all.
- No Redux/Zustand: use React state + Inertia shared props for global data (current translator/admin user, etc.).

### Page Structure (Pages)

```
resources/js/
  Pages/
    Home.tsx                    # browse novels + filter/search
    Novels/
      Show.tsx                  # novel detail + chapter list (on_revision badge)
      Chapter.tsx                # read a chapter, triggers recording chapter_reads
    TranslatorProfile.tsx        # public translator profile page
    Favorites.tsx
    ContinueReading.tsx
    Auth/
      Callback.tsx                # handles the SSO callback state (if a UI is needed)
    Dashboard/
      Index.tsx                  # list of the translator's novels
      Novels/Create.tsx
      Novels/Edit.tsx
      Chapters/Create.tsx
      Chapters/Edit.tsx          # Tiptap editor + autosave indicator
    Admin/
      Index.tsx                  # statistics dashboard
      Users.tsx                  # read-only + grant/revoke role buttons
      Novels.tsx
      Genres.tsx
      Tags.tsx
      Tickets.tsx
  Components/
    NovelCard.tsx
    ChapterEditor.tsx            # Tiptap wrapper + autosave
    FilterBar.tsx
    FavoriteButton.tsx
    CreatorAutocomplete.tsx      # search-as-you-type author/illustrator
```

---

## Do Not

- Do not build a local auth/password system: all translator & admin logins go through SSO PKCE.
- Do not build a register/login system for readers: readers are fully anonymous.
- Do not allow admins to edit a user's name/email/avatar directly: that is SSO's domain.
- Do not allow translators to edit/delete novels or chapters that are not theirs (always check via Policy).
- Do not build a publish-approval flow: the platform is centralized and translators publish directly.
- Do not store cover image files outside the configured Laravel Filesystem disk.
- Do not build a comment/rating/notification system in the MVP (out of scope).
- Do not build an avatar upload/edit feature before there is a follow-up briefing about SSO avatars.
- Do not add Redux, Zustand, or any other state management library.
- Do not skip step 2 of logout (redirect to SSO logout): it causes auto-login.
- Do not add `console.log` in production code.

---

## Code Style

- Laravel: PSR-12, type hints on all methods, thin controllers: put logic in a Service/Action class when it is complex.
- React: functional components only, TypeScript strict mode, no inline styles (use Tailwind via Shadcn).
- Commit messages: conventional commits (`feat:`, `fix:`, `chore:`).

---

## Deployment

- Hosting: EC2 / Linux VM.
- Staging domain: scribe.suryatmaja.dev
- Production domain (planned): scribe.whitearchive.id
- First deploy: `make deploy` → `sudo bash scripts/deploy.sh`
- Updates: `make update` → `bash scripts/update.sh` (git pull + migrate + rebuild)
- No CI/CD yet — see [docs/TODO.md](../docs/TODO.md).

---

## TODO

See [docs/TODO.md](../docs/TODO.md).
