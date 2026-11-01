# TASK-014: Reader — Read Chapter + Chapter Read Tracking

Status: In Review
Priority: High
Created: 2026-07-23 20:00
Request: The chapter reading page, with routes with and without a volume (`/novels/{slug}/{chapter}` and `/novels/{slug}/vol-{vol}/{chapter}`). Render Tiptap content. Generate a `device_id` (cookie/localStorage) if the reader does not have one, then upsert a `chapter_reads` row (device_id + chapter_id) every time a chapter is opened.

---

## DEV Response

Deviating from SRS's literal "cookie/localStorage" wording: going cookie-only (httpOnly, 5-year expiry), not localStorage. Since Inertia renders full pages server-side per request (not a pure client SPA), a server-set httpOnly cookie is simpler, more secure (not readable/tamperable by injected client JS), and needs zero client-side code — the middleware transparently assigns a `device_id` on first visit and every subsequent request already carries it. This same middleware is reusable infrastructure for TASK-015 (favorites) and TASK-016 (continue reading), which need the identical device-identity mechanism.

- [x] `EnsureDeviceId` middleware — reads `device_id` cookie; if absent, generates a UUID, queues a long-lived cookie on the response, AND sets it directly on the current `$request` (cookie queueing only affects the *next* request, so without this the very first page load that creates the device_id wouldn't have it available to itself). Registered on the `web` middleware group so it's active for all public routes, not just chapter reading
- [x] Minimal `ChapterRead` model (no relation needed beyond FK) — same "add the model now, full feature ticket comes later" pattern as `Ticket` in TASK-004
- [x] `ChapterService::recordRead()` — `ChapterRead::updateOrCreate(['device_id', 'chapter_id'], ['read_at' => now()])`, matches the schema's `UNIQUE(device_id, chapter_id)` so repeat reads update `read_at` instead of erroring on the unique constraint
- [x] `Public\ChapterController` (replacing TASK-013's stub logic): after resolving the published chapter, call `recordRead()` with the request's `device_id` before rendering
- [x] Verified via full HTTP-kernel round trips through tinker: first visit sets an httpOnly `device_id` cookie and creates 1 `chapter_reads` row; a genuine same-device round trip (reusing the actual encrypted `Set-Cookie` value the server issued, not a hand-typed plaintext UUID) correctly upserts rather than duplicating; draft chapters still create zero read records since they 404 before `recordRead()` is ever reached.

**Test-harness note, not a product bug:** my first verification pass used a hand-constructed plaintext device_id cookie for the "repeat visit" simulation and got false failures (looked like every visit was creating a new device). Root cause: Laravel's `EncryptCookies` middleware encrypts `device_id` on the way out and expects an encrypted value on the way in — a real browser round-trips the exact `Set-Cookie` value it received, so this only breaks a hand-rolled test, not actual usage. Re-ran using the real encrypted cookie value and the upsert behavior is confirmed correct.

---

## QA Response
[QA fills this]

- [ ] test case
