# Scribe — User Flow & Auth Sequence

## Main Flow (Reader, Translator, Admin)

```mermaid
flowchart TD
    Start([Visit scribe.whitearchive.id]) --> Visitor{Who is visiting?}

    Visitor -->|Anonymous reader| RGetDevice[Generate/get device_id from cookie]
    RGetDevice --> RBrowse[Browse novels: genre/tag/status filters, title search]
    RBrowse --> RDetail[Open the novel detail page]
    RDetail --> RChapterList{Chapter status?}
    RChapterList -->|published| RRead[Read chapter]
    RChapterList -->|on_revision| RBlocked["'Under revision' badge, cannot be opened"]
    RChapterList -->|draft| RHidden[Not shown to readers]
    RRead --> RTrack[Upsert chapter_reads by device_id]
    RTrack --> RContinue["Appears in /continue-reading + read/unread indicator"]
    RDetail --> RFav[Favorite a novel - device based]
    RFav --> RFavList[Appears in /favorites]
    RRead --> RTicket["Optional: send a ticket to a translator/superadmin"]

    Visitor -->|Translator / Admin| SSOLogin["Click Login -> redirect to SSO authorize + PKCE"]
    SSOLogin --> SSOCallback["Callback: exchange code, sync profile"]
    SSOCallback --> RoleCheck{Role from SSO?}
    RoleCheck -->|superadmin| AdminRole[Local role = admin]
    RoleCheck -->|user, first login| PendingRole[Local role = pending]
    PendingRole --> WaitGrant[Waiting for an admin to grant access]
    WaitGrant -->|granted by admin| TranslatorRole[Local role = translator]

    TranslatorRole --> Dashboard[Translator dashboard]
    Dashboard --> NovelCRUD["Novel CRUD: title + alternative titles, cover, author/illustrator, genre/tag"]
    NovelCRUD --> VolumeCRUD[Volume CRUD - optional]
    VolumeCRUD --> ChapterEditor[Write a chapter: blog-style editor + inline images]
    ChapterEditor --> Autosave[Periodic autosave while typing]
    Autosave --> StatusToggle{Translator sets chapter status}
    StatusToggle -->|draft| Draft[Private, not yet shown to readers]
    StatusToggle -->|published| Published[Shown to readers immediately, no admin approval]
    StatusToggle -->|on_revision| Revision[Shown with a badge, readers cannot open it]
    Dashboard --> TranslatorTicket[Send a bug/feature ticket to the superadmin]

    AdminRole --> AdminPanel[Admin panel]
    AdminPanel --> AdminStats[Platform statistics dashboard]
    AdminPanel --> AdminUsers["View users - read only + grant/revoke the translator role"]
    AdminPanel --> AdminModerate[Moderate violating novels/chapters]
    AdminPanel --> AdminTaxonomy[Genre & tag CRUD]
    AdminPanel --> AdminTickets[Manage incoming tickets from translators & readers]
```

---

## SSO Auth Sequence (Login & 2-Step Logout)

```mermaid
sequenceDiagram
    participant U as Browser (Translator/Admin)
    participant S as Scribe (Laravel)
    participant SSO as sso.whitearchive.id

    U->>S: GET /auth/login
    S->>S: Generate PKCE verifier+challenge, state -> store in session
    S-->>U: Redirect to SSO authorize
    U->>SSO: GET /oauth/authorize (client_id, code_challenge, state)
    SSO-->>U: Redirect to /auth/callback?code=...&state=...
    U->>S: GET /auth/callback
    S->>S: Validate state against the session
    S->>SSO: POST /oauth/token (code + code_verifier)
    SSO-->>S: access_token (60m), refresh_token (30d)
    S->>SSO: GET /api/user (Bearer access_token)
    SSO-->>S: profile: sso_id, name, username, email, sso_role
    S->>S: Upsert local users, map role (superadmin->admin, user->pending), create session
    S-->>U: Redirect to dashboard/admin

    Note over U,SSO: Logout is mandatory 2-step - skipping step 2 = auto-login next time
    U->>S: POST /auth/logout
    S->>S: Delete the local session
    S-->>U: Redirect to SSO /logout?redirect_uri=...
    U->>SSO: GET /logout
    SSO-->>U: Redirect back to Scribe, SSO session invalidated
```

---

## Notes

- Readers never touch the SSO flow: all reader interaction is anonymous, purely based on a `device_id` in a cookie/localStorage.
- There is no approval/publish-request step between "translator sets published" and "chapter shown to readers": the platform is centralized, publishing is direct.
- An `on_revision` chapter stays in the database and stays linked to its novel/volume; it is only hidden from reader access.
