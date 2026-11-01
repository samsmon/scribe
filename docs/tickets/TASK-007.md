# TASK-007: Novel/Series CRUD (Translator) + Cover Upload + Alternative Titles

Status: In Review
Priority: High
Created: 2026-07-23 20:00
Request: A translator dashboard to create/edit/delete their own novels: title (original title), slug (auto-generated), synopsis, cover_image (local disk upload), origin_language, translation_language, status. Including management of multi-language alternative titles (novel_alt_titles: language + title, more than one row per novel allowed).

---

## DEV Response

This is the first frontend-touching ticket, and there was no Inertia/React/Shadcn scaffolding in `backend/` yet (only a bare Laravel skeleton + an unrelated, unwired top-level `frontend/` Vite SPA — confirmed with the user this should be ignored and everything built inside `backend/` per CLAUDE.md's monolith architecture). Folding that foundational setup into this ticket since every subsequent frontend ticket depends on it.

- [x] Foundation: install `inertiajs/inertia-laravel` + `tightenco/ziggy` (composer), `@inertiajs/react`, `react`, shadcn deps (`class-variance-authority`, `clsx`, `tailwind-merge`, `lucide-react`, radix primitives), `@tiptap/*` (npm, ahead of TASK-011's editor need)
- [x] `resources/views/app.blade.php` root view, `resources/js/app.tsx` entrypoint, `vite.config.js` React plugin + `@` alias, `tsconfig.json` (strict mode)
- [x] `HandleInertiaRequests` middleware sharing `auth.user` (id/name/username/avatar/role only — no tokens), registered in `bootstrap/app.php` web group
- [x] Base shadcn-style primitives in `resources/js/Components/ui`: Button, Input, Textarea, Label, Card, Badge, Select + `cn()` util + CSS theme tokens in `app.css`
- [x] `NovelService` (slug generation from title with uniqueness suffixing, alt-title sync, cover image store/replace/delete on local `public` disk)
- [x] `StoreNovelRequest` / `UpdateNovelRequest`: title, synopsis, origin/translation language required; cover_image nullable image upload; status enum; alt_titles array of {language,title}
- [x] `NovelPolicy` wired into controller (`$this->authorize()`) — update/delete restricted to owner, create restricted to translator role (already built in TASK-003)
- [x] `Dashboard\NovelController`: index (own novels only), create, store, edit, update, destroy — thin, delegates to `NovelService`
- [x] Routes registered inside the `role:translator` `/dashboard` group from TASK-003
- [x] Pages: `Dashboard/Index.tsx` (own novel list), `Dashboard/Novels/Create.tsx`, `Dashboard/Novels/Edit.tsx` — Inertia `useForm`, alt-titles as dynamic repeatable rows
- [x] `storage:link` so uploaded covers are web-accessible
- [x] Verification: full create → edit (slug regen, alt-title resync) → policy-block → delete round trip run directly against `NovelService`/policies via tinker, plus HTTP-kernel round trips through the real middleware stack for `/` and `/dashboard`

**Bug found and fixed during verification:** unauthenticated access to any `role:translator`/`role:admin` route 500'd with `RouteNotFoundException: Route [login] not defined` — Laravel's default `Authenticate` middleware calls `route('login')` synchronously to build its redirect, and this app has no `login` route (SSO-only, named `sso.login`). Fixed via `$middleware->redirectGuestsTo(fn () => route('sso.login'))` in `bootstrap/app.php`. This affects every protected route, not just Novel CRUD, so it's a fix worth calling out to QA explicitly — confirmed `/dashboard` now cleanly 302s to `/auth/login` when logged out.

Also removed the default `welcome.blade.php` and `resources/js/app.js` (replaced by the Inertia root view/entrypoint), and pointed `/` at a placeholder `Home.tsx` (full reader browse page is TASK-012's scope) since deleting the old default view meant `/` needed *something* to render.

**Note for QA:** could not get a live screenshot/interaction confirmation in the Browser pane — `https://scribe.test` hit a "requires per-action approval" gate I could not clear myself (possibly needs one-time manual approval, or a `.test`-domain TLS quirk). Substituted with direct HTTP-kernel round-trips through tinker (same middleware/routing/controller pipeline Laravel actually runs) plus a static `npm run build` to confirm the TSX compiles without type/syntax errors. Recommend an actual browser click-through before sign-off.

---

## QA Response
[QA fills this]

- [ ] test case
