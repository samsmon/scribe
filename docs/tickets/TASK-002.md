# TASK-002: Database Schema & Migrations (Full Schema)

Status: In Review
Priority: High
Created: 2026-07-23 20:00
Request: Create the PostgreSQL migrations for all tables per docs/SRS.md: users (sso_id, sso_role, local role, token fields), novels, novel_alt_titles, creators, novel_author, novel_illustrator, volumes, chapters (with draft/on_revision/published status), genres, tags, novel_genre, novel_tag, favorites (device-based), chapter_reads (device-based), tickets. Including UNIQUE constraints and enums per the spec.

---

## DEV Response

Full schema per SRS.md, PostgreSQL. `users` base migration modified in place (no password/email_verified_at — SSO-only auth per CLAUDE.md). Rest as new migrations in dependency order.

- [x] Modify `create_users_table`: sso_id, username, email, avatar, sso_role enum(user,superadmin), role enum(pending,translator,admin) default pending, bio, donation_url, access_token/refresh_token (encrypted text), token_expires_at — drop password/email_verified_at/rememberToken
- [x] `creators` table (name unique)
- [x] `novels` table (user_id FK, slug unique, status enum, origin/translation language)
- [x] `novel_alt_titles` table (novel_id FK, language, title)
- [x] `novel_author` / `novel_illustrator` pivot tables (novel_id, creator_id composite PK)
- [x] `volumes` table (novel_id FK, number, title nullable)
- [x] `chapters` table (novel_id FK, volume_id nullable FK, chapter_number decimal(8,1), status enum, published_at, last_autosaved_at, UNIQUE(novel_id,volume_id,chapter_number))
- [x] `genres` / `tags` tables (name+slug unique) + `novel_genre` / `novel_tag` pivots
- [x] `favorites` table (device_id, novel_id FK, UNIQUE(device_id,novel_id))
- [x] `chapter_reads` table (device_id, chapter_id FK, read_at, UNIQUE(device_id,chapter_id))
- [x] `tickets` table (type/from_type/to_type enums, from_user_id/to_user_id nullable FK, from_device_id nullable, status enum)
- [x] Verify migration order/FK dependencies — ran `migrate:fresh` against a scratch SQLite DB via tinker, all 18 migrations applied cleanly (note: PostgreSQL is the target per SRS/CLAUDE.md; schema uses no PG-specific syntax so this should carry over, but recommend QA re-verify against actual Postgres)

---

## QA Response
[QA fills this]

- [ ] test case
