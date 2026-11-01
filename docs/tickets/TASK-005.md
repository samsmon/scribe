# TASK-005: Admin — User List (Read-Only) + Grant/Revoke Role Translator

Status: In Review
Priority: High
Created: 2026-07-23 20:00
Request: The `/admin/users` page shows all users who have ever logged in (name/email/avatar data is read-only, purely from SSO sync, and cannot be edited from Scribe). An admin can change Scribe's local `role` column (grant `pending` → `translator`, or revoke it back) via PUT `/admin/users/{id}/role`. There must be no UI for editing user profiles.

---

## DEV Response

- [x] `UpdateUserRoleRequest`: `role` restricted to `in:pending,translator` only — **`admin` is deliberately excluded**. Per SRS role mapping, `admin` is auto-derived from SSO `superadmin` on every login (`SsoService::syncUser`), never admin-settable locally; also blocks the request entirely if the target user's *current* role is already `admin`, since that role is SSO-owned and any local change would just get overwritten (or worse, silently diverge) on the user's next login
- [x] `Admin\UserController@index` — read-only list of all synced users (name/username/email/avatar/sso_role/role), explicitly no name/email/avatar edit fields anywhere in the payload or UI (SRS: that's SSO's domain)
- [x] `Admin\UserController@updateRole` — `PUT /admin/users/{id}/role`
- [x] `Pages/Admin/Users.tsx` — card list with grant/revoke buttons touching only `role`; no edit affordance for any SSO-owned field. Linked from `AdminLayout` nav
- [x] Verified via tinker: grant pending→translator, revoke translator→pending, `role=admin` rejected by the validation rule itself, attempting to change an already-`admin` user's role rejected by the `withValidator` guard (and confirmed the role stayed unchanged), index payload shape confirmed read-only-appropriate

---

## QA Response
[QA fills this]

- [ ] test case
