# TASK-001: SSO Integration — Login, Callback, Logout, Role Mapping

Status: In Review
Priority: High
Created: 2026-07-23 20:00
Request: Implement the OAuth2 Authorization Code + PKCE flow to sso.whitearchive.id per AI_INTEGRATION.md: login redirect, callback with state validation, code-to-token exchange, profile sync to the local `users` (sso_id, name, username, email, sso_role). Logout must be 2-step: clear the local session, then redirect to SSO logout. Role mapping: SSO `superadmin` → Scribe `admin`, SSO `user` (first login) → Scribe `pending`. Env vars: SSO_CLIENT_ID, SSO_CLIENT_SECRET, SSO_REDIRECT_URI, SSO_BASE_URL.

---

## DEV Response

Follows `sso.whitearchive/docs/AI_INTEGRATION.md` pattern, adapted to Scribe's route names (`/auth/login` not `/auth/redirect`) and schema (tokens persisted encrypted on `users` row, not session — per SRS/PRD security requirements). Logic lives in `SsoService`, controller stays thin.

- [x] `config/sso.php` (client_id, client_secret, redirect_uri, base_url) + `.env.example` entries
- [x] `app/Services/Sso/SsoService.php`: `redirectUrl()` (PKCE verifier+challenge, state, stored server-side in session), `handleCallback()` (validate state, exchange code, fetch profile, upsert user + role mapping), `logoutUrl()`
- [x] Role mapping logic: SSO `superadmin` → local `role=admin` every login; SSO `user` → local `role=pending` only on first create (never downgrade an already-promoted translator on subsequent logins)
- [x] `User` model: encrypted casts for `access_token`/`refresh_token`, hide both + never expose to client, fillable sso fields (fixed a bug during testing where these 3 columns were missing from the `Fillable` attribute, silently dropping them on mass-assignment)
- [x] `app/Http/Controllers/Auth/SsoController.php`: `login()`, `callback()`, `logout()`
- [x] Routes: `GET /auth/login`, `GET /auth/callback`, `POST /auth/logout` (+ `GET /auth/logout` for SSO-initiated redirect back)
- [x] Logout 2-stage: clear local session (`Auth::logout()` + invalidate + regenerate token) then redirect to `SSO_BASE_URL/logout?redirect_uri=...`
- [x] Refresh token handling: `SsoService::refreshToken()` exchanges + rotates stored refresh token (single-use); not yet wired to an automatic before-expiry trigger (no scheduled job or middleware calls it yet — flagging for follow-up, see note below)
- [x] Manual verification via tinker against scratch SQLite DB: PKCE challenge generation confirmed correct (S256 of verifier), tokens confirmed encrypted at rest and decrypt correctly, hidden from JSON serialization, role-promotion preserved across re-login, superadmin auto-maps to `admin` every login

**Note for QA / follow-up:** `refreshToken()` exists but nothing currently calls it automatically — a middleware or scheduled check should invoke it before the 60-minute access token expiry. Recommend a small follow-up ticket if this needs to ship before token expiry becomes a problem in practice. Also could not test the live end-to-end OAuth round-trip against the real `sso.whitearchive.id` since that requires a registered client_id/secret — verified the PKCE/crypto/persistence logic in isolation instead.

---

## QA Response
[QA fills this]

- [ ] test case
