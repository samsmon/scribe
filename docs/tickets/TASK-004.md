# TASK-004: Admin — Statistics Dashboard

Status: In Review
Priority: Medium
Created: 2026-07-23 20:00
Request: The `/admin` page shows platform statistics: number of active translators, number of released novels (status not draft), number of published chapters, and other relevant metrics (number of open tickets, etc.).

---

## DEV Response

No `Ticket` model exists yet (full ticketing is TASK-017's scope) — adding a minimal one here since the stats need to count open tickets, same pattern as adding minimal Novel/Chapter models ahead of their CRUD tickets in TASK-003.

- [x] Minimal `Ticket` model (fillable, `open()` scope) — no controllers/UI, just enough to query counts
- [x] `Admin\DashboardController@index` — counts: active translators (`role=translator`), released novels (`status != 'draft'`), published chapters (`status = 'published'`), open tickets (`status = 'open'`), plus pending users awaiting role grant (useful context for admins even though not explicitly requested — it's the queue TASK-005 acts on)
- [x] Route: `GET /admin` inside the existing `role:admin` group
- [x] `Pages/Admin/Index.tsx` — stat cards, reuses existing `Card`/`Badge` primitives, no new UI components needed
- [x] `Layouts/AdminLayout.tsx` — separate from `DashboardLayout` since admin nav will differ (moderation, users, genres/tags, tickets link out as those tickets land)
- [x] Verified via tinker (seeded 2 translators, 1 pending, 1 admin, 1 draft + 1 ongoing novel, 1 draft + 1 published chapter, 1 open + 1 closed ticket): all five counts came back exactly right — draft novel and draft chapter correctly excluded, pending user correctly excluded from the translator count, closed ticket correctly excluded from open count. Also confirmed `role:admin` middleware blocks a translator from `/admin`.

---

## QA Response
[QA fills this]

- [ ] test case
