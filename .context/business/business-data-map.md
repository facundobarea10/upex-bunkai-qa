# Business Data Map — Bunkai TMS

> **Discovery date**: 2026-05-31  
> **Generated from**: All 4 discovery phases — DB migrations (18 tables), API routes, page components, auth flows  
> **Refresh command**: `/business-data-map` (run in a clean session after schema changes)  
> **Purpose**: Single reference for KATA test design — flows, entities, state machines, integration points

---

## Entity Map

```
auth.users (Supabase managed — not a Bunkai table)
    |
    +──────────────── TENANCY ────────────────────────────────────────
    |
    +── workspaces
    |       slug (unique), name, plan (community|cloud|enterprise)
    |       owner_user_id → auth.users
    |       |
    |       +── workspace_members ──── user_id → auth.users
    |       |       role: viewer|member|admin|owner
    |       |       status: active|invited|suspended
    |       |
    |       +──────────── CONTENT HIERARCHY ─────────────────────────
    |       |
    |       +── projects
    |       |       slug (unique within workspace), name, description
    |       |       |
    |       |       +── modules (self-referential tree, max depth 6)
    |       |       |       path (materialized: "Feature/Login/UI")
    |       |       |       parent_module_id → modules (nullable)
    |       |       |       position (sort order among siblings)
    |       |       |       |
    |       |       |       +── user_stories
    |       |       |               external_id (Jira key, e.g. BK-42)
    |       |       |               external_url (Jira link)
    |       |       |               |
    |       |       |               +── acceptance_criteria
    |       |       |                       position (ordered within story)
    |       |       |                       |
    |       |       |                       +──── atc_acceptance_criteria ──┐
    |       |       |                                                        │ M:N
    |       |       +── atcs ──────────────────────────────────────────────-+
    |       |               slug (unique within project)
    |       |               layer: UI|API|Unit
    |       |               status: unrun|running|pass|fail|blocked|skipped
    |       |               tags[] (GIN indexed for full-text search)
    |       |               version (optimistic lock counter)
    |       |               tsv (tsvector: title + tags, auto-refreshed by trigger)
    |       |               |
    |       |               +── atc_steps (ordered, position unique per atc)
    |       |               |       content, input_data, expected
    |       |               |
    |       |               +── atc_assertions (ordered, position unique per atc)
    |       |                       content
    |       |
    |       +──────────── ACCESS & AUTH ──────────────────────────────
    |       |
    |       +── access_tokens ──── access_token_secrets (1:1, isolated)
    |       |       token_prefix (12 chars, indexed for O(1) lookup)
    |       |       scopes[]: atc:read|atc:write|run:execute|workspace:admin
    |       |       revoked_at (soft-delete), expires_at, last_used_at
    |       |
    |       +── workspace_invites ── workspace_invite_secrets (1:1, isolated)
    |       |       email, role, expires_at (now + 7 days)
    |       |       accepted_at, accepted_by_user_id
    |       |       revoked_at (soft-delete)
    |       |
    |       +──────────── CROSS-CUTTING ──────────────────────────────
    |       |
    |       +── activity_log
    |       |       entity_type, entity_id, action (create|update|delete)
    |       |       payload (JSONB diff), actor_user_id
    |       |
    |       +── idempotency_keys (24h TTL)
    |       |       (user_id, endpoint, key) UNIQUE
    |       |       response_snapshot (JSONB), response_status
    |       |
    |       +── feature_flags
    |       |       scope: global|workspace
    |       |       key, enabled, payload (JSONB)
    |       |
    |       +── user_view_state (per user + project)
    |               view_kind (module_tree|atc_list), state (JSONB)
    |
    +── magic_link_tokens ── magic_link_token_secrets (1:1, isolated)
            expires_at (now + 1 hour), consumed_at (single-use)
```

---

## System Flows

### Flow 1 — Magic-Link Authentication

```
[Browser]                           [Next.js]                      [Supabase]
    |                                   |                               |
    |  POST /api/v1/auth/magic-link     |                               |
    |  { email, next: "/projects" }     |                               |
    +──────────────────────────────────>|                               |
    |                                   | auth.signInWithOtp({ email }) |
    |                                   +──────────────────────────────>|
    |                                   |                               | Send OTP email
    |                                   |<──────────────────────────────|
    |<──────────────────────────────────|                               |
    |  { ok: true }                     |                               |
    |                                   |                               |
    | [User clicks email link]          |                               |
    |                                   |                               |
    |  GET /auth/callback?code=...      |                               |
    |  &next=/projects                  |                               |
    +──────────────────────────────────>|                               |
    |                                   | exchangeCodeForSession(code)  |
    |                                   +──────────────────────────────>|
    |                                   |<── session + user ────────────|
    |<── Set-Cookie: supabase session ──|                               |
    |<── 302 redirect /projects ────────|                               |
```

**Test entry point**: `POST /api/v1/auth/magic-link` + `GET /auth/callback?code=...&next=...`  
**Auth result**: httpOnly session cookie (`supabase-auth-token`)  
**Open redirect guard**: `next` must start with `/`, must not start with `//`

---

### Flow 2 — Headless Agent Authentication (Bearer PAT)

```
[CLI/Agent]                          [Next.js /api/v1/auth/signin]
    |                                   |
    |  POST /api/v1/auth/signin         |
    |  { email, password,               |
    |    pat_name, pat_scopes,          |
    |    workspace_id?, expires_in_days }|
    +──────────────────────────────────>|
    |                                   | supabase.signInWithPassword()
    |                                   | → Mint PAT:
    |                                   |   INSERT access_tokens (prefix, scopes, expires_at)
    |                                   |   INSERT access_token_secrets (SHA-256 hash)
    |                                   |
    |<──────────────────────────────────|
    |  {                                |
    |    session: { access_token, ... },|
    |    pat: {                         |
    |      token: "bk_pat_<12>.<32>",   | ← raw token returned ONCE ONLY
    |      scopes, expires_at           |
    |    }                              |
    |  }                                |
    |                                   |
    | [All subsequent calls]            |
    |  Authorization: Bearer bk_pat_...+>|
    |                                   | requireAuth(): Bearer path
    |                                   | → prefix lookup (access_tokens.token_prefix idx)
    |                                   | → SHA-256 verify (access_token_secrets.hash)
    |                                   | → resolve userId + scopes
```

**Token format**: `bk_pat_<12-char-prefix>.<32-byte-secret-base64url>`  
**Scope enforcement**: `requireScopeOrCookie()` — Bearer tokens checked against `scopes[]`; cookies bypass (RLS enforces instead)

---

### Flow 3 — Workspace Bootstrap (Onboarding)

```
[Browser + OnboardingForm]          [Next.js]                     [PostgreSQL]
    |                                   |                               |
    |  POST /api/v1/workspaces          |                               |
    |  { slug: "acme-qa",              |                               |
    |    name: "Acme QA" }             |                               |
    +──────────────────────────────────>|                               |
    |                                   | CALL bunkai_bootstrap_workspace(slug, name)
    |                                   +──────────────────────────────>|
    |                                   |  (in one transaction):        |
    |                                   |  INSERT workspaces            |
    |                                   |  INSERT workspace_members     |
    |                                   |    (owner_user_id, role=owner,|
    |                                   |     status=active)            |
    |                                   |<──── workspace_id ────────────|
    |<──────────────────────────────────|                               |
    |  { id, slug, name, plan }         |                               |
    |                                   |                               |
    |  [Toast: "Workspace created"]     |                               |
    |  [Redirect: /projects]            |                               |
```

**Slug validation**: `^[a-z0-9][a-z0-9-]{1,38}[a-z0-9]$`  
**Reserved slugs blocked**: `api`, `auth`, `admin`, `login`, etc.  
**409 on conflict**: slug already taken — form shows "Slug taken" error

---

### Flow 4 — ATC Save (Atomic Editor Write)

```
[AtcEditor (client)]                [saveAtcAction (server)]       [PostgreSQL RPC]
    |                                   |                               |
    | save({ atcId, title, layer,       |                               |
    |        tags, userStoryId,         |                               |
    |        steps[], assertions[],     |                               |
    |        acIds[] })                 |                               |
    +──────────────────────────────────>|                               |
    |                                   | CALL bunkai_save_atc(...)     |
    |                                   +──────────────────────────────>|
    |                                   |  (in one transaction):        |
    |                                   |  DELETE atc_steps             |
    |                                   |  INSERT atc_steps (new)       |
    |                                   |  DELETE atc_assertions        |
    |                                   |  INSERT atc_assertions (new)  |
    |                                   |  DELETE atc_acceptance_criteria|
    |                                   |  INSERT atc_acceptance_criteria|
    |                                   |  UPDATE atcs SET              |
    |                                   |    title, layer, tags,        |
    |                                   |    version = version + 1,     |
    |                                   |    updated_at = now()         |
    |                                   |    (trigger: atcs_set_updated_at)
    |                                   |    (trigger: atcs_refresh_tsv)|
    |                                   |<──── void ────────────────────|
    |<──────────────────────────────────|                               |
    |  [success feedback]               |                               |
```

**Key behavior**: Steps and assertions are **delete-then-reinsert** (not diffed) — positions are always 0..N contiguous after save.  
**Optimistic lock**: `version` counter incremented on every save (Phase E if-match support planned)

---

### Flow 5 — Team Invite Flow

```
[Admin (Sarah)]                     [Next.js]                     [Invitee (James)]
    |                                   |                               |
    | POST /api/v1/workspaces/{id}/invites|                             |
    | { email: "james@co.com",          |                               |
    |   role: "member" }                |                               |
    +──────────────────────────────────>|                               |
    |                                   | Generate raw token (crypto)   |
    |                                   | INSERT workspace_invites      |
    |                                   |   (expires_at = now + 7 days) |
    |                                   | INSERT workspace_invite_secrets|
    |                                   |   (SHA-256 hash of token)     |
    |<──────────────────────────────────|                               |
    |  { accept_url, raw_token }        |                               |
    |  [MVP: link copied to clipboard]  |                               |
    |                                   |                               |
    | [Sarah sends link to James]       |                               |
    |                                   |   GET /invites/accept?token=..|
    |                                   |<──────────────────────────────|
    |                                   |   [AcceptClient checks auth]  |
    |                                   |                               |
    |                                   |   POST /api/v1/invites/accept |
    |                                   |   { token: "<raw>" }          |
    |                                   |<──────────────────────────────|
    |                                   | Hash → lookup invite_id       |
    |                                   | Validate: email match,        |
    |                                   |   not expired, not revoked,   |
    |                                   |   not accepted                |
    |                                   | UPSERT workspace_members      |
    |                                   |   (status=active)             |
    |                                   | UPDATE workspace_invites      |
    |                                   |   SET accepted_at, accepted_by|
    |                                   +──────────────────────────────>|
    |                                   |   302 → /projects             |
```

**Expiry**: 7 days from creation  
**Email match**: Invitee's auth email must match `workspace_invites.email` (case-insensitive)  
**MVP note**: No automated email sending — admin copies link from API response

---

## State Machines

### ATC Status

```
          [unrun] ──────────────── default state
            |
            +──> [running] ──> [pass]
            |              \──> [fail]
            |              \──> [blocked]
            |              \──> [skipped]
            |
            +──> [blocked] (manual set, dependency blocker)
            +──> [skipped] (manual set, out of scope)

[pass] ──────────────────> [unrun] (reset for re-run)
[fail] ──────────────────> [unrun] (reset for re-run)
[blocked] ────────────────> [unrun] (blocker resolved)
```

Colors: `pass=#2fb673` `fail=#e5484d` `blocked=#e8a838` `skipped=#8a91a0` `running=#4f8cf7 (pulsing)`

---

### Workspace Invite Lifecycle

```
[issued]
    |
    +── expires_at reached ──────────> [expired] (read-only, no further transitions)
    |
    +── revoked_at stamped ──────────> [revoked] (immediate invalidation)
    |
    +── token accepted ─────────────> [accepted]
    |       └─> workspace_members upserted (status=active)
    |
    +── token rotated ──────────────> [revoked] + new [issued]
```

**Token validation order**: exists → not revoked → not accepted → not expired → email matches

---

### PAT (Personal Access Token) Lifecycle

```
[active]
    |
    +── expires_at reached ──────────> [expired] (bearer calls return 401)
    |
    +── DELETE /api/v1/tokens/{id} ──> [revoked] (revoked_at stamped, soft-delete)
    |
    +── last_used_at updated on use

[revoked] / [expired] ──> terminal, no recovery
```

**Lookup**: `token_prefix` (O(1) index) → SHA-256 verify → check `revoked_at IS NULL` + `expires_at > now()`

---

### Workspace Member Status

```
[invited] ─── invite accepted ──────> [active]
[invited] ─── invite expired ───────> (member record not created)
[invited] ─── invite revoked ───────> (member record not created)
[active]  ─── admin suspends ───────> [suspended]
[suspended] ── admin re-activates ──> [active]
```

---

### BK Story Workflow (Jira)

```
[Backlog]
    └──> [Shift-Left QA] ──> [Estimation] ──> [Ready For Dev]
                                                    └──> [In Progress]
                                                            ├──> [BLOCKED] ──> [In Progress]
                                                            └──> [Ready For QA]
                                                                     └──> [In Test]
                                                                              ├──> [QA Approved]
                                                                              │        └──> [Ready For Release]
                                                                              │                 └──> [Deployed to Production]
                                                                              └──> [In Progress] (QA failed, re-open)
[ABORTED] ── terminal from any state
```

---

## Integration Points

| System | Direction | Protocol | Purpose |
|---|---|---|---|
| **Supabase Auth** | Outbound | Supabase JS SDK | Magic-link OTP, session management, auth.users |
| **Supabase PostgreSQL** | Outbound | Supabase JS SDK (RLS-enforced) | All application data |
| **Vercel** | Deploy target | Git push → Vercel webhook | Hosting + edge functions |
| **Jira (BK project)** | Bidirectional | REST API v3 (POST /search/jql) | UserStory.external_id linkage; bug filing |
| **Resend** | Outbound (planned) | HTTP REST | Transactional email — not yet wired in Phase 1 |
| **n8n** | Planned | REST API | Workflow automation — `N8N_API_URL` in .env, not yet wired |
| **Tavily** | Outbound (MCP) | MCP | Web search — AI agent use only |

---

## DB Triggers

| Trigger | Table | Event | Action |
|---|---|---|---|
| `atcs_refresh_tsv` | `atcs` | BEFORE INSERT / UPDATE of `title` or `tags` | Refreshes `tsv` tsvector column for full-text search |
| `atcs_set_updated_at` | `atcs` | BEFORE UPDATE | Sets `updated_at = now()` |

---

## Scheduled Jobs / Webhooks

**Discovery Gap**: No cron jobs, background workers, or webhooks found in Phase 1 source.

| Concern | Status |
|---|---|
| Idempotency key TTL cleanup | No background job found — `expires_at` column exists but no worker purges expired rows |
| Magic-link token expiry cleanup | Same — `expires_at` exists; no purge job |
| Session refresh | Handled per-request by Next.js middleware (`supabase.auth.getUser()`) |

---

## Critical Test Surfaces (for KATA)

| Surface | Why it matters | Suggested layer |
|---|---|---|
| Magic-link auth flow | Entry point for all browser users | API (signin) + UI (form) |
| Workspace creation (onboarding) | First action after auth; atomic RPC | API + UI |
| ATC save (bunkai_save_atc) | Core mutation; delete-then-reinsert means bad positions corrupt data | API |
| Invite accept flow | Multi-step auth chain; email match validation | API + UI |
| PAT auth (Bearer) | Agent execution path; scope enforcement | API |
| RLS enforcement | Any cross-tenant data access attempt | API |
| ATC full-text search | GIN index on tsv; tag-based filtering | API |
| Module tree max depth | CHECK constraint at 6 levels | API |
| Slug uniqueness | Workspace + project slugs unique within parent | API |
| Invite expiry (7 days) | Time-sensitive — requires date manipulation in tests | API |
| ATC status transitions | No DB constraint on transition order — app-layer only | API |
