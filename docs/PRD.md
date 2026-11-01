# PRD - Scribe (Light Novel Publishing CMS)

## Problem Statement

Freelance light novel translators need a centralized platform to publish their translations without running their own infrastructure. Readers need a single place to browse and read works from many translators at once, without the friction of creating an account.

## Target Users

- **Translator**: freelance/hobbyist light novel translators (JP/KR/CN → ID/EN/etc.) who publish their work in a structured way (series → volume → chapter). Log in via SSO whitearchive.id.
- **Reader**: anonymous readers, no account. Browse, read, favorite, and continue reading across all translators on one platform.
- **Admin (superadmin)**: platform manager. Logs in via SSO whitearchive.id with the `superadmin` role.

## Platform Model

Scribe is **one of the projects under the White Archive umbrella**, not the main/centralized White Archive platform itself. `whitearchive.id` (the root domain) is the White Archive landing page that introduces the ecosystem and lists all the projects under it, which is outside Scribe's scope. Scribe lives on its own subdomain (`scribe.whitearchive.id`, staging: `scribe.suryatmaja.dev`).

Within Scribe's own scope, the model is still a single centralized platform, **not** a subdomain per translator. All novels from all Scribe translators appear in one catalog. Each novel/post is tagged with its owning translator; translators have their own profile page as their identity, without needing a separate domain.

## Core User Stories

### Reader (no login/account)
- As a reader, I want to browse the novel list with genre/tag/status filters and title search, so I can quickly find reading material from any translator.
- As a reader, I want to see novel details (synopsis, author, illustrator, alternative titles, chapter list per volume), so I know the novel's progress before reading.
- As a reader, I want to read chapters directly in the browser without an account.
- As a reader, I want chapters being revised by the translator to be clearly marked and unopenable, so I know they are not final.
- As a reader, I want to favorite novels (stored on my device), so I can check for updates without an account.
- As a reader, I want the system to remember which chapters I have and have not read per novel (device-based), so I can continue from my last chapter like on Wattpad.
- As a reader, I want to send a ticket to the superadmin or to a specific translator (e.g. a chapter request), so I can communicate without an account.

### Translator (login via SSO)
- As a translator, I want to log in with my existing SSO whitearchive.id account, so I do not need to register a new one.
- As a translator, I want to CRUD series (original title, multi-language alternative titles, synopsis, cover, author, illustrator, genre, tag), so I can manage my works with complete metadata.
- As a translator, I want to pick an author/illustrator from the existing list via autocomplete (or create a new one if missing), so data is not duplicated.
- As a translator, I want to CRUD volumes (optional) and write chapters with a blog-like editor (with inline images mid-text), so I can publish translations in a neat format.
- As a translator, I want the chapter I am writing to be autosaved automatically, so I do not lose progress if something goes wrong (laptop shuts off, etc.).
- As a translator, I want to set a chapter's status (draft / on revision / published), so I can mark a chapter that needs revision without removing it from the system.
- As a translator, I want to publish novels/chapters directly without admin approval (because the platform is centralized, not separate subdomains).
- As a translator, I want to send a ticket to the superadmin (bug report / feature request), so I can report technical problems.

### Admin (superadmin, login via SSO)
- As an admin, I want to see a statistics dashboard (number of translators, number of released novels, etc.), so I have a picture of the platform's condition.
- As an admin, I want to see the list of users/translators (read-only; profiles are managed via SSO, not in Scribe), and change the local Scribe role (grant/revoke translator access).
- As an admin, I want to moderate (delete) novels/chapters that violate the rules.
- As an admin, I want to manage the genre and tag master data.
- As an admin, I want to receive and respond to tickets from both translators and readers.

## Features

### P0 (Must Have — MVP)

- SSO login (OAuth2 + PKCE) for translators & admins, not a custom auth system
- Local role gating: new users from SSO are `pending` until an admin grants the `translator` role; the SSO role `superadmin` is automatically mapped to Scribe admin
- Admin dashboard: platform statistics (number of translators, number of released novels, etc.)
- Admin: view users (read-only) + grant/revoke the translator role
- Admin: moderate novels & chapters (delete)
- Admin: genre & tag CRUD
- Translator: series/novel CRUD: original title, alternative titles (multiple), synopsis, cover, origin/translation language
- Translator: author & illustrator as entities with autocomplete (create-if-not-exist)
- Translator: attach/detach genres & tags to a novel
- Translator: volume CRUD (optional for chapters)
- Translator: chapter CRUD: blog-style editor with inline images, autosave, draft/on_revision/published status
- Reader: browse novels with genre/tag/status filters + title search, no login
- Reader: novel detail page (full info + chapter list per volume); on_revision chapters are marked and cannot be opened
- Reader: read chapters, no login
- Reader: favorite novels (device-based, no account)
- Reader: continue reading + read/unread chapter indicator (device-based)
- Ticketing: translator → superadmin (bug/feature request); reader → superadmin or a specific translator

### P1 (Should Have)

- Public translator profile page (list of works + bio + donation link)
- Pagination / infinite scroll in the novel listing
- Reader ticket page with translator destination selection (UI details discussed separately)

### P2 (Nice to Have)

- Swap cover image storage from local disk to S3/R2
- Full-text search upgrade (PostgreSQL `to_tsvector`)
- Dark mode
- Syncing reader favorites/reading progress across devices (needs a revisit of the reader account decision if required later)

## Out of Scope (MVP)

- Internal coin / monetization system (only external Trakteer/Tako links on the translator profile)
- Comment & rating system
- Notifications (email/push)
- RSS feed
- PWA / mobile app
- Subdomain per translator
- Accounts/login for readers

## Security Requirements

- Auth is entirely via SSO (OAuth2 Authorization Code + PKCE), no local passwords
- The state parameter is validated server-side to prevent CSRF during login
- Access & refresh tokens are stored encrypted on the server and never sent to the client
- Logout is mandatory 2-step: clear the local session, then redirect to the SSO logout endpoint
- Laravel Policy-based authorization: translators may only modify their own novels/chapters
- Admins cannot edit user profile data (name/email/avatar): that is SSO's domain, admins only manage the local Scribe role
- Reader tracking (favorites, reading progress) is based on an anonymous device_id, not tied to any personal identity
- HTTPS enforced in production

## Success Metrics

- A translator can log in via SSO and publish their first novel (series + at least 1 chapter) end-to-end without admin approval
- A reader can browse → read → favorite in one session without needing any account
- Continue reading & read/unread status are 100% accurate per device
- Tickets from translators/readers reach the correct destination (superadmin or a specific translator)
