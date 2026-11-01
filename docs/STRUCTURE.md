# Project Structure

A full explanation of every folder and file in this project.

```
scribe/
├── docs/                              # Project documentation
│   ├── PRD.md                         # Product Requirements Document
│   ├── SRS.md                         # Software Requirements Specification
│   ├── TODO.md                        # List of unfinished work
│   ├── STRUCTURE.md                   # This file
│   └── tickets/                       # TASK-XXX.md and bugs/BUG-XXX.md
│
├── backend/                           # Laravel 11 + Inertia.js (serve React, monolith)
│   ├── app/
│   │   ├── Http/
│   │   │   ├── Controllers/
│   │   │   │   ├── Auth/
│   │   │   │   │   └── SsoController.php        # login redirect, callback, 2-stage logout
│   │   │   │   ├── NovelController.php          # Browse, detail, CRUD (dashboard)
│   │   │   │   ├── ChapterController.php        # Read, CRUD, autosave (dashboard)
│   │   │   │   ├── VolumeController.php         # CRUD volume
│   │   │   │   ├── CreatorController.php        # Autocomplete author/illustrator
│   │   │   │   ├── FavoriteController.php       # Favorite/unfavorite (device-based)
│   │   │   │   ├── ReadingController.php        # Record chapter_reads, continue reading
│   │   │   │   ├── TicketController.php         # Submit tickets (translator & reader)
│   │   │   │   ├── TranslatorProfileController.php # Public translator profile page
│   │   │   │   ├── Admin/
│   │   │   │   │   ├── DashboardController.php  # Platform statistics
│   │   │   │   │   ├── UserController.php       # Read-only user list + grant/revoke role
│   │   │   │   │   ├── NovelController.php      # Novel moderation
│   │   │   │   │   ├── ChapterController.php    # Chapter moderation
│   │   │   │   │   ├── GenreController.php      # CRUD genre
│   │   │   │   │   ├── TagController.php        # CRUD tag
│   │   │   │   │   └── TicketController.php     # Manage incoming tickets
│   │   │   └── Requests/                         # Form request validation per resource
│   │   ├── Models/
│   │   │   ├── User.php                          # sso_id, role lokal, sso_role
│   │   │   ├── Novel.php
│   │   │   ├── NovelAltTitle.php
│   │   │   ├── Volume.php
│   │   │   ├── Chapter.php
│   │   │   ├── Creator.php                       # author & illustrator (shared entity)
│   │   │   ├── Genre.php
│   │   │   ├── Tag.php
│   │   │   ├── Favorite.php                      # device-based, no user relation
│   │   │   ├── ChapterRead.php                   # device-based, no user relation
│   │   │   └── Ticket.php
│   │   ├── Services/
│   │   │   └── Sso/
│   │   │       └── SsoClient.php                 # OAuth2 PKCE flow wrapper for sso.whitearchive.id
│   │   ├── Policies/
│   │   │   ├── NovelPolicy.php                   # Translators can only edit their own novels
│   │   │   └── ChapterPolicy.php                  # Translators can only edit their own chapters
│   │   └── Providers/
│   │       └── AppServiceProvider.php
│   │
│   ├── database/
│   │   └── migrations/                            # users (+ sso fields), novels, novel_alt_titles,
│   │                                               # creators, novel_author, novel_illustrator,
│   │                                               # volumes, chapters, genres, tags, pivot tables,
│   │                                               # favorites, chapter_reads, tickets
│   │
│   ├── resources/
│   │   ├── js/
│   │   │   ├── Pages/                             # Inertia pages (see CLAUDE.md)
│   │   │   ├── Components/                        # NovelCard, ChapterEditor (Tiptap), CreatorAutocomplete, etc.
│   │   │   └── app.tsx                            # Entry point Inertia + React
│   │   └── css/
│   │       └── app.css                            # Tailwind + Shadcn base styles
│   │
│   ├── routes/
│   │   └── web.php                                # All routes (public, SSO auth, dashboard, admin)
│   │
│   ├── storage/
│   │   └── app/public/covers/                     # Novel cover images (local disk, MVP)
│   │
│   ├── .env                                       # Local environment (not committed), including SSO_*
│   ├── .env.example
│   ├── artisan
│   ├── composer.json
│   └── package.json                               # Vite + React + Inertia + Shadcn + Tiptap deps
│
├── scripts/
│   ├── deploy.sh                      # First-time deploy wizard for EC2
│   └── update.sh                      # Pull latest from GitHub + migrate + rebuild + redeploy
│
├── Makefile                           # make deploy / make update
├── README.md                          # Main documentation
└── .claude/
    └── CLAUDE.md                      # Instructions for Claude Code
```

---

## Request Flow

```
Browser (reader, anonymous — device_id in a cookie)
  │
  ├─► GET  /                    → NovelController@index    (Inertia::render('Home'))
  ├─► GET  /novels/{slug}       → NovelController@show      (Inertia::render('Novels/Show'))
  ├─► GET  /novels/{slug}/{ch}  → ChapterController@show    (record ChapterRead via device_id)
  ├─► POST /favorites           → FavoriteController@store  (upsert by device_id + novel_id)
  │
Browser (translator/admin — session from SSO)
  ├─► GET  /auth/login          → SsoController@redirect    (PKCE challenge + state → SSO authorize)
  ├─► GET  /auth/callback       → SsoController@callback    (exchange code, sync profile, create session)
  │
  ├─► POST /dashboard/novels    → NovelController@store     (NovelPolicy::create, role:translator)
  ├─► PUT  /dashboard/novels/{id} → NovelController@update  (NovelPolicy::update — check owner)
  ├─► PATCH /dashboard/.../chapters/{id}/autosave → ChapterController@autosave
  │
  └─► PUT  /admin/users/{id}/role → Admin\UserController@updateRole (role:admin middleware, only changes the local role)
```

Inertia renders React components directly from the Laravel response; there is no separate REST API layer for the main pages.

---

## Environment Variables

### backend/.env

| Key | Example | Description |
|-----|--------|------------|
| `APP_ENV` | `production` | Laravel mode |
| `APP_KEY` | `base64:...` | Generate via `artisan key:generate` |
| `APP_URL` | `https://scribe.suryatmaja.dev` | Application URL |
| `DB_CONNECTION` | `pgsql` | PostgreSQL |
| `DB_HOST` / `DB_PORT` / `DB_DATABASE` / `DB_USERNAME` / `DB_PASSWORD` | — | PostgreSQL credentials |
| `FILESYSTEM_DISK` | `public` | Local disk for cover images (MVP) |
| `SSO_CLIENT_ID` | `xxx` | Client ID from sso.whitearchive.id |
| `SSO_CLIENT_SECRET` | `xxx` | Client secret (never expose it to the frontend) |
| `SSO_REDIRECT_URI` | `https://scribe.suryatmaja.dev/auth/callback` | Callback URL registered in SSO |
| `SSO_BASE_URL` | `https://sso.whitearchive.id` | Identity provider base URL |

---

## Migration Notes from Previous Iterations

This repo has gone through 2 pivots:
1. File-to-markdown converter (Laravel API + separate Vite frontend, no DB) → fully replaced.
2. A light novel CMS with local reader login & translators who self-register and then get promoted → replaced by the current model: **a centralized platform, translator/admin auth via SSO whitearchive.id, fully anonymous device-based readers, and no publish-approval flow.**

The structure above is the final MVP version in effect now.
