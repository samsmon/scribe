# TASK-018: Public Translator Profile Page

Status: In Review
Priority: Low
Created: 2026-07-23 20:00
Request: The `/translator/{username}` page: the translator's identity on the centralized platform (replacing the subdomain concept). Show the bio, donation_url, and the list of novels published by that translator.

---

## DEV Response

- [x] `Public\TranslatorController@show(username)` — 404s unless `role = translator` (matches SRS wording literally: this is "translator's public profile," not "any user's page" — a revoked translator's profile page disappears along with their dashboard access, consistent with how a novel reverting to draft disappears from browse/favorites/continue-reading elsewhere in this app)
- [x] Shows `bio`, `donation_url`, and the translator's non-draft novels (draft novels stay owner-only, same visibility rule as everywhere else); response only exposes the safe subset of `User` fields, never `access_token`/`refresh_token`
- [x] Route: `GET /translator/{username}`
- [x] `Pages/TranslatorProfile.tsx`; also linked the translator's `@username` on the novel detail page to their new profile, since "give translators an identity via a profile page instead of a subdomain" is this ticket's whole point per SRS's platform model
- [x] Verified via tinker: profile 404s for a pending user, shows the published novel and hides the draft one for a real translator, no token leakage in the payload, and — a stronger check than just "shows correctly" — revoking a translator's role makes their profile 404 immediately while their previously-published novel's `status` is untouched and still visible everywhere else (browse, favorites, etc.), confirming the profile's visibility is independent of the novel's own visibility rules

---

## QA Response
[QA fills this]

- [ ] test case
