# TODO

## MVP Build (see docs/tickets/ for the full breakdown)

- [ ] TASK-001 to TASK-018 — see `docs/tickets/`
- [ ] BUG-001 — fix the `sync.sh` regex (`parse_stack` escapes the grep alternation incorrectly)

## CI/CD Pipeline

- [ ] GitHub Actions workflow, trigger on push to `main`
  - Job: `composer install --no-dev` → `php artisan migrate --force` → `npm ci` → `vite build` → `artisan optimize` → SSH deploy to EC2
  - Gate: `php artisan test` must pass before the deploy step runs
  - Note: this is now a Laravel+Inertia monolith, no longer a separate frontend & backend build like the previous iteration

## Upcoming / Needs Follow-up Briefing

- [ ] SSO avatars — the feature does not exist yet in sso.whitearchive.id, use a placeholder for now. Update the UI after the follow-up briefing.
- [ ] Reader page UI for sending a ticket to a specific translator (target translator selection) — to be discussed in detail separately, the backend/model is already part of TASK-017.

## P1 (Should Have, outside the initial MVP)

- [ ] Full public translator profile page (TASK-018 already covers the basics)
- [ ] Pagination / infinite scroll in the novel listing
