# TASK-010: Volume CRUD (Translator Dashboard)

Status: In Review
Priority: Medium
Created: 2026-07-23 20:00
Request: Volume CRUD under a translator's own novel: number, title (nullable). Volumes are optional; a chapter may have no volume at all (shown flat).

---

## DEV Response

SRS's route list doesn't actually enumerate volume routes (only novel and chapter routes are listed, despite volumes being module #3 in the breakdown) — treating this as an oversight and adding nested routes under the novel, consistent with the rest of the dashboard. Since volumes are a lightweight, optional organizational unit (not a full page-worthy resource per SRS's Pages list), managing them inline as a section on the existing `Dashboard/Novels/Edit` page rather than separate Create/Edit pages.

- [x] Ownership check reuses `NovelPolicy@update` on the parent novel (volume mutation = part of managing the novel you own) — no separate `VolumePolicy` needed
- [x] `StoreVolumeRequest`/`UpdateVolumeRequest`: `number` required int ≥1, unique per novel (not explicitly in SRS schema but duplicate volume numbers on one novel would be a real data-integrity problem); `title` nullable string
- [x] `Dashboard\VolumeController`: `store`, `update`, `destroy` nested under `/dashboard/novels/{novel}/volumes` — kept in the controller directly (simple CRUD, no complex logic warranting a service class). Added an explicit `abort_unless($volume->novel_id === $novel->id, 404)` guard on `update`/`destroy` since Laravel's implicit route-model binding does *not* automatically verify a nested child belongs to its parent in the URL — without it, a translator could edit/delete their own volume through a URL naming a different (even someone else's) novel ID, as long as the outer novel's ownership check passed independently
- [x] `NovelController::edit` eager-loads `volumes` (ordered by number) and passes them to the Edit page
- [x] Frontend: `Components/VolumeManager.tsx` — inline list + add/edit/delete form embedded in `Dashboard/Novels/Edit.tsx`
- [x] Verified via tinker: create volume, duplicate number on same novel rejected, same number on a different novel allowed, non-owner blocked by `authorize()`, update, delete, and — importantly — the cross-novel mismatch guard: attempted `VolumeController::update` with a volume belonging to Novel A but a URL naming Novel B correctly 404s instead of silently succeeding

---

## QA Response
[QA fills this]

- [ ] test case
