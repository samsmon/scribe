# SRS - Scribe (Light Novel Publishing CMS)

## Tech Stack

| Layer | Choice |
|-------|--------|
| Backend | Laravel 11 |
| Database | PostgreSQL |
| Frontend | Inertia.js + React |
| UI | Shadcn UI |
| Rich Text / Blog Editor | Tiptap (chapter content, supports inline images) |
| Auth | SSO whitearchive.id (OAuth2 Authorization Code + PKCE) — no local password |
| Storage | Local disk (MVP) → S3/R2 (later) via Laravel Filesystem |
| Domain (staging) | scribe.suryatmaja.dev |
| Domain (production, planned) | scribe.whitearchive.id |

---

## Platform Model

Scribe is one of the projects under the White Archive umbrella, not the main White Archive platform. `whitearchive.id` (the root domain) is the White Archive ecosystem landing page (a list of all projects), which is outside Scribe's scope. Scribe runs on its own subdomain (`scribe.whitearchive.id`).

Within Scribe's scope: one centralized platform. **Not** a subdomain per translator. All novels from all translators appear in one public catalog, marked with each translator's ownership. Translators get their identity through a profile page (`/translator/{username}`), not a separate domain.

---

## Auth: SSO Integration

Reference: `AI_INTEGRATION.md` from `sso.whitearchive.id`.

### Env Vars

```
SSO_CLIENT_ID=xxx
SSO_CLIENT_SECRET=xxx
SSO_REDIRECT_URI=https://scribe.suryatmaja.dev/auth/callback
SSO_BASE_URL=https://sso.whitearchive.id
```

### Endpoints (SSO)

| Endpoint | Purpose |
|----------|---------|
| `GET /oauth/authorize` | Authorization request + PKCE challenge |
| `POST /oauth/token` | Exchange authorization code / refresh token |
| `GET /api/user` | Fetch the user profile (needs an access token) |
| `GET /logout` | Terminate the SSO session, supports `redirect_uri` |

### Flow

1. **Redirect**: generate a PKCE code verifier + challenge, generate `state`, and store it in the server-side session.
2. **Authorize**: redirect the user to `SSO_BASE_URL/oauth/authorize` with client_id, redirect_uri, code_challenge, state.
3. **Callback** (`/auth/callback`): validate `state` against the session; exchange the authorization code + code_verifier at `/oauth/token`.
4. **Profile sync**: fetch the profile via `/api/user`; create/update the local `users` record by `sso_id`.
5. **Session**: create a local application session; store the access token (60 minutes) & refresh token (30 days, single-use) encrypted on the server.
6. **Refresh**: implement refresh logic before the access token expires, and update the refresh token each time it is used.
7. **Logout (mandatory 2-step)**: (a) delete the local session, (b) redirect to `SSO_BASE_URL/logout` with `redirect_uri`. Skipping step (b) causes auto-login on the next login attempt because the SSO session is still active.

### Role Mapping

SSO only has 2 global roles: `user` and `superadmin`. Scribe has its own separate local roles:

| SSO role | Scribe role (default) | Notes |
|----------|------------------------|-------|
| `superadmin` | `admin` | Auto-mapped on every login |
| `user` | `pending` (first login) | A Scribe admin manually grants `translator` via `/admin/users/{id}/role` |

Avatars from SSO are **not available yet**: use a default placeholder in the UI until the SSO avatar feature is released (briefing to follow).

---

## Database Schema

### users
```
id                 bigint PK
sso_id             varchar unique
name               varchar
username           varchar unique
email              varchar unique
avatar             varchar nullable    -- not used yet, placeholder in the UI for now
sso_role           enum(user, superadmin)   -- synced on every login
role               enum(pending, translator, admin) default pending  -- local Scribe role
bio                text nullable
donation_url       varchar nullable    -- Trakteer/Tako link
access_token       text nullable       -- encrypted
refresh_token      text nullable       -- encrypted
token_expires_at   timestamp nullable
timestamps
```

### novels
```
id                    bigint PK
user_id               bigint FK(users)      -- owning translator
title                 varchar               -- original title (e.g. Japanese)
slug                  varchar unique
synopsis              text
cover_image           varchar nullable
status                enum(draft, ongoing, completed, hiatus, dropped) default draft
origin_language       varchar               -- e.g. Japanese, Korean, Chinese
translation_language  varchar               -- e.g. Indonesian
timestamps
```

### novel_alt_titles
```
id           bigint PK
novel_id     bigint FK(novels)
language     varchar     -- e.g. Indonesian, English, Spanish
title        varchar
timestamps
```

### creators
A combined entity for authors & illustrators: the same person can be an author on one novel and an illustrator on another without duplicated data.
```
id      bigint PK
name    varchar unique
timestamps
```

### novel_author (pivot)
```
novel_id     bigint FK(novels)
creator_id   bigint FK(creators)
PRIMARY KEY(novel_id, creator_id)
```

### novel_illustrator (pivot)
```
novel_id     bigint FK(novels)
creator_id   bigint FK(creators)
PRIMARY KEY(novel_id, creator_id)
```

### volumes
```
id           bigint PK
novel_id     bigint FK(novels)
number       integer
title        varchar nullable   -- e.g. "Volume 1: The Beginning"
timestamps
```

### chapters
```
id                  bigint PK
novel_id            bigint FK(novels)
volume_id           bigint FK(volumes) nullable
chapter_number      decimal(8,1)      -- supports 1.5 for side stories/interludes
title               varchar nullable
content             text              -- markdown/Tiptap JSON, supports inline images
status              enum(draft, on_revision, published) default draft
published_at        timestamp nullable
last_autosaved_at   timestamp nullable
timestamps
UNIQUE(novel_id, volume_id, chapter_number)
```

### genres
```
id     bigint PK
name   varchar unique
slug   varchar unique
```

### novel_genre (pivot)
```
novel_id   bigint FK(novels)
genre_id   bigint FK(genres)
PRIMARY KEY(novel_id, genre_id)
```

### tags
Free-form labels, different from genres (a fixed taxonomy). E.g. isekai, reincarnation, slow-burn.
```
id     bigint PK
name   varchar unique
slug   varchar unique
```

### novel_tag (pivot)
```
novel_id   bigint FK(novels)
tag_id     bigint FK(tags)
PRIMARY KEY(novel_id, tag_id)
```

### favorites
Device-based, readers have no account.
```
id           bigint PK
device_id    varchar     -- UUID in the reader's cookie/localStorage
novel_id     bigint FK(novels)
timestamps
UNIQUE(device_id, novel_id)
```

### chapter_reads
Device-based. The source for continue reading (last chapter = MAX(read_at) per novel) and the read/unread indicator per chapter.
```
id           bigint PK
device_id    varchar
chapter_id   bigint FK(chapters)
read_at      timestamp
UNIQUE(device_id, chapter_id)
```

### tickets
```
id                bigint PK
type              enum(bug, feature_request, chapter_request, other)
from_type         enum(translator, reader)
from_user_id      bigint FK(users) nullable    -- filled when from_type = translator
from_device_id    varchar nullable             -- filled when from_type = reader
to_type           enum(superadmin, translator)
to_user_id        bigint FK(users) nullable    -- target translator or a specific admin
subject           varchar
message           text
status            enum(open, in_progress, resolved, closed) default open
timestamps
```

---

## Module Breakdown

### 1. Auth (SSO)
- Login redirect + callback (OAuth2 PKCE)
- Profile sync to the local `users`
- Local role gating (pending/translator/admin)
- 2-step logout

### 2. Novel/Series Management (translator)
- Novel CRUD: original title, alternative titles (multiple), synopsis, cover, origin/translation language
- Author & illustrator: pick via autocomplete from `creators`, or create a new one if missing
- Attach/detach genres & tags
- Set novel status (draft/ongoing/completed/hiatus/dropped)

### 3. Volume & Chapter Management (translator)
- Volume CRUD (optional, nullable)
- Chapter CRUD: blog-style editor (Tiptap) with inline images
- Periodic autosave while typing
- Chapter status: draft / on_revision / published

### 4. Reader (no login)
- Browse novels (filter by genre, tag, status, title search)
- Novel detail page (full metadata + chapter list per volume; on_revision chapters are marked & cannot be opened)
- Read a chapter: automatically recorded to `chapter_reads` via device_id
- Favorite/unfavorite novels (device-based)
- Continue reading + read/unread chapter indicator

### 5. Admin
- Statistics dashboard (number of translators, number of released novels, etc.)
- View users (read-only, data from SSO) + grant/revoke the translator role
- Moderate novels & chapters (delete)
- Genre & tag CRUD
- Receive & respond to tickets (translators & readers)

### 6. Ticketing
- Translator → superadmin (bug report / feature request)
- Reader → superadmin or a specific translator (e.g. a chapter request): device-based, no account
- The reader page UI for submitting tickets is discussed separately (P1)

---

## Route Structure

### Public (no login)
```
GET  /                                    → browse novels (filter & search)
GET  /novels/{slug}                       → novel detail + chapter list
GET  /novels/{slug}/{chapter}             → read a chapter (no volume)
GET  /novels/{slug}/vol-{vol}/{chapter}   → read a chapter (with volume)
GET  /translator/{username}               → public translator profile page
GET  /favorites                           → favorites list (device-based)
POST /favorites
DELETE /favorites/{novel}
GET  /continue-reading                    → continue reading list (device-based)
POST /tickets                             → reader submits a ticket
```

### Auth (SSO)
```
GET  /auth/login       → redirect to SSO authorize
GET  /auth/callback    → handle the SSO callback, profile sync
POST /auth/logout      → 2-stage logout
```

### Translator Dashboard (role: translator)
```
GET    /dashboard                                   → list of the translator's novels
GET    /dashboard/novels/create
POST   /dashboard/novels
GET    /dashboard/novels/{id}/edit
PUT    /dashboard/novels/{id}
DELETE /dashboard/novels/{id}
GET    /dashboard/novels/{id}/chapters
GET    /dashboard/novels/{id}/chapters/create
POST   /dashboard/novels/{id}/chapters
GET    /dashboard/novels/{id}/chapters/{cid}/edit
PUT    /dashboard/novels/{id}/chapters/{cid}
PATCH  /dashboard/novels/{id}/chapters/{cid}/autosave
DELETE /dashboard/novels/{id}/chapters/{cid}
GET    /dashboard/creators/search                    → author/illustrator autocomplete
POST   /dashboard/tickets                            → translator submits a ticket to the superadmin
```

### Admin (role: admin)
```
GET    /admin                          → statistics dashboard
GET    /admin/users                    → read-only, data from SSO
PUT    /admin/users/{id}/role          → grant/revoke the translator role (local Scribe role)
GET    /admin/novels
DELETE /admin/novels/{id}
GET    /admin/chapters
DELETE /admin/chapters/{id}
GET    /admin/genres
POST   /admin/genres
DELETE /admin/genres/{id}
GET    /admin/tags
POST   /admin/tags
DELETE /admin/tags/{id}
GET    /admin/tickets
PUT    /admin/tickets/{id}
```

---

## Key Architectural Decisions

1. **Auth is entirely via SSO whitearchive.id**: no local passwords, no register form. Only translators & admins log in; readers are fully anonymous.
2. **SSO role and Scribe role are separate.** SSO only has `user`/`superadmin`. Scribe has the local roles `pending`/`translator`/`admin`. The SSO `superadmin` auto-maps to Scribe admin; the SSO `user` enters as `pending` until granted by an admin.
3. **A centralized platform, not a subdomain per translator.** One catalog for all translators: readers browse/read across translators without switching domains. Translators get their identity through a profile page, not their own domain.
4. **Readers are fully anonymous and device-based.** Favorites and reading progress (`chapter_reads`) are keyed by `device_id` (cookie/localStorage), not tied to an account. Trade-off: no cross-device sync, and data is lost if cookies are cleared; this was agreed to be an acceptable trade-off for the MVP.
5. **chapter_number uses `decimal(8,1)`**, not integer. Supports chapters 1.5, 2.5 for side stories/interludes.
6. **Volumes are optional.** `volume_id` is nullable in chapters.
7. **No publish-approval flow.** Because the platform is centralized (not subdomains), translators publish novels/chapters directly without admin approval.
8. **Chapters have 3 statuses**: `draft` (not yet published, private), `on_revision` (was published, now being revised: readers see a badge and cannot open it), `published` (shown normally to readers).
9. **Chapter autosave.** Content is saved periodically while the translator types (`last_autosaved_at` is updated), preventing loss of progress.
10. **Authors & illustrators as entities (`creators`)**, not free text: chosen via autocomplete, create-if-not-exist. One table serves both via separate pivots (`novel_author`, `novel_illustrator`) so the same person is not duplicated.
11. **Genre and Tag are separate.** Genre = a fixed taxonomy (managed by admins). Tag = additional free-form labels (managed by admins, used by translators).
12. **Avatars from SSO are not available yet.** The UI uses a default placeholder; the avatar feature follows a separate briefing.
13. **Two-way ticketing.** Translator → superadmin (bug/feature). Reader → superadmin or a specific translator (device-based, no account).
14. **Cover images on local disk for the MVP.** Relative paths, easy to swap to S3/R2 via Laravel Filesystem.
15. **Search via PostgreSQL ILIKE.** No need for Elasticsearch in the MVP.
16. **Authorization via Laravel Policies.** Translators can only edit their own novels & chapters.

---

## Acceptance Criteria

| Feature | Criteria |
|---------|----------|
| SSO Login | Translators/admins log in via the SSO PKCE flow, and the profile is synced to the local `users`. |
| Role gating | A new user from SSO (`user`) enters as `pending` and cannot access `/dashboard` until an admin grants `translator`. The SSO `superadmin` automatically becomes a Scribe admin. |
| Logout | Logout deletes the local session AND redirects to SSO logout, so the next login does not auto-login. |
| Novel CRUD | A translator can create/edit/delete their own novels and cannot edit another translator's novel. Alternative titles can span more than one language. |
| Author/Illustrator | Can pick from the existing list via autocomplete or create a new one; no duplicate names. |
| Chapter CRUD | A chapter is saved with rich content (supports inline images). Draft & on_revision chapters do not appear / cannot be opened on public pages. |
| Autosave | Chapter content is saved automatically without the translator clicking save manually. |
| Optional volumes | A chapter without a volume is still valid and displayed flat on the novel detail page. |
| Browse & filter | Genre/tag/status filters and title search return matching results across all translators. |
| Favorite | A reader can favorite/unfavorite without an account, with no duplicates per device (UNIQUE constraint). |
| Continue reading | `chapter_reads` is upserted correctly per device+chapter; continue reading shows the last chapter per novel correctly. |
| Role promote | Only an admin can change a user's local role via `/admin/users/{id}/role`. |
| Admin read-only user data | An admin cannot edit a user's name/email/avatar: that data comes purely from SSO. |
| Authorization | A translator who tries to edit a novel/chapter that is not theirs is rejected (403). |
| Ticketing | Tickets from translators reach the superadmin; tickets from readers reach the superadmin or the correct destination translator. |

---

## Out of Scope (MVP)

- Internal coin / monetization system
- Comment & rating system
- Notifications (email/push)
- RSS feed
- PWA / mobile app
- Subdomain per translator
- Accounts/login for readers
- Cross-device sync of favorites/reading progress
