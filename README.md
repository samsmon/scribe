# Scribe

A centralized light novel publishing CMS for freelance translators and readers.

**Readers need no login**: they browse and read directly, and favorites & reading progress are tracked per device (not per account). **Translators & admins log in via SSO whitearchive.id.** There is no per-translator subdomain: all works appear in one catalog.

## Docs

| Document | Description |
|----------|-------------|
| [docs/PRD.md](docs/PRD.md) | Product Requirements — problem, user stories, features, success metrics |
| [docs/SRS.md](docs/SRS.md) | Software Requirements — tech stack, schema, SSO integration, route structure |
| [docs/STRUCTURE.md](docs/STRUCTURE.md) | Full file structure with explanation of every folder and file |
| [docs/FLOW.md](docs/FLOW.md) | User flow & SSO auth sequence (Mermaid flowchart) |
| [docs/TODO.md](docs/TODO.md) | Planned work and known gaps |
| [SESSION-PROMPTS.md](SESSION-PROMPTS.md) | Copy-paste prompts for PM / DEV / QA Claude Code sessions |

---

## Roles

| Role | Login | Description |
|------|-------|-----------|
| Reader | Not required | Browse, read, favorite, continue reading: all device-based, no account. |
| Translator | SSO whitearchive.id | Granted by an admin (defaults to `pending` after first login). Manages their own novels & chapters. |
| Admin | SSO whitearchive.id (`superadmin`) | Full access: statistics, role management, moderation, taxonomy, tickets. |

---

## Features

- **Anonymous reader**: browse (genre/tag filter, search), read chapters, favorite, continue reading + read/unread indicator, all without an account (device-based)
- **Translator dashboard**: novel CRUD (original title + multi-language alternative titles, cover, author/illustrator via autocomplete, genre/tag), volume CRUD (optional), chapter CRUD with a blog-style editor (Tiptap, inline images) + autosave
- **3-level chapter status**: draft (private), on_revision (shown with a badge, cannot be opened), published (shown immediately, no admin approval)
- **Admin panel**: statistics dashboard, view users (read-only, data from SSO) + grant/revoke the translator role, content moderation, genre/tag CRUD, ticket management
- **Ticketing**: translator → superadmin (bug/feature request), reader → superadmin or a specific translator

---

## Tech Stack

| Layer | Choice |
|-------|--------|
| Backend | Laravel 11 + Inertia.js |
| Database | PostgreSQL |
| Frontend | React (via Inertia) |
| UI | Shadcn UI |
| Editor | Tiptap |
| Auth | SSO whitearchive.id (OAuth2 Authorization Code + PKCE) |
| Storage | Local disk (MVP) → S3/R2 (later) |
| Hosting | EC2 / any Linux VM |

---

## Project Structure

```
scribe/
  backend/                  # Laravel 11 + Inertia.js (monolith)
    app/
      Http/Controllers/     # Novel, Chapter, Volume, Favorite, Reading, Ticket, Auth/Sso, Admin/*
      Models/                # User, Novel, NovelAltTitle, Volume, Chapter, Creator, Genre, Tag,
                              # Favorite, ChapterRead, Ticket
      Policies/              # NovelPolicy, ChapterPolicy
      Services/Sso/          # SsoClient (OAuth2 PKCE wrapper)
    resources/js/
      Pages/                 # Home, Novels/Show, Novels/Chapter, Dashboard/*, Admin/*
      Components/            # NovelCard, ChapterEditor (Tiptap), CreatorAutocomplete, FavoriteButton
    routes/web.php
    database/migrations/
  scripts/
    deploy.sh                # First-time deploy wizard
    update.sh                # Pull latest + migrate + rebuild + redeploy
  docs/
    PRD.md
    SRS.md
    STRUCTURE.md
    FLOW.md
    TODO.md
    tickets/                 # TASK-XXX.md per feature ticket
      bugs/                  # BUG-XXX.md per bug ticket
  .claude/
    CLAUDE.md                 # Project instructions for Claude Code
    agents/                   # PM.md, DEV.md, QA.md session personas
  Makefile                    # make sync / make update / make deploy
  sync.sh                     # Sync stack from SRS into .claude/CLAUDE.md
  SESSION-PROMPTS.md
```

---

## Local Development

### Prerequisites

- PHP 8.2+
- Composer
- Node.js 18+
- PostgreSQL

### Setup

```bash
cd backend
composer install
cp .env.example .env
php artisan key:generate
```

Set in `backend/.env`:
```env
APP_URL=http://localhost:8000
DB_CONNECTION=pgsql
DB_HOST=127.0.0.1
DB_PORT=5432
DB_DATABASE=scribe
DB_USERNAME=postgres
DB_PASSWORD=

SSO_CLIENT_ID=xxx
SSO_CLIENT_SECRET=xxx
SSO_REDIRECT_URI=http://localhost:8000/auth/callback
SSO_BASE_URL=https://sso.whitearchive.id
```

```bash
php artisan migrate
npm install
npm run dev          # Vite dev server (HMR for React)
php artisan serve    # → http://localhost:8000
```

---

## Deployment

- Hosting: EC2 / Linux VM
- Staging: `scribe.suryatmaja.dev`
- Production (planned): `scribe.whitearchive.id`
- First deploy: `make deploy` → `sudo bash scripts/deploy.sh`
- Update: `make update` → `bash scripts/update.sh` (git pull + migrate + rebuild)

---

## Security

- Auth is entirely via SSO (OAuth2 + PKCE): no local passwords, no register form
- Logout is mandatory 2-step: clear the local session + redirect to SSO logout
- Authorization via Laravel Policy: translators can only modify their own content
- Admins cannot edit user profiles (name/email/avatar): that is purely SSO's domain, admins only manage the local role
- Reader tracking (favorites, reading progress) is anonymous and based on `device_id`, not tied to any identity
- Rate-limit ticket submissions from anonymous readers to prevent spam

---

## TODO

See [docs/TODO.md](docs/TODO.md) for planned work and known gaps.
