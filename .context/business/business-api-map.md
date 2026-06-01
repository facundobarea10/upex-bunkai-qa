# Business API Map — Bunkai TMS

> **Last verified against OpenAPI on 2026-05-31**  
> **Discovery date**: 2026-05-31  
> **Generated from**: `business-data-map.md`, `business-feature-map.md`, SRS/architecture.md, infrastructure/backend.md, functional-specs.md — all Phase 1 discovery outputs  
> **Refresh command**: `/business-api-map` (run after new auth schemes, critical routes, or integrations are added)  
> **Purpose**: Business-first narrative of how the API powers user journeys — auth tiers, critical flows, architecture, integrations. Not an endpoint catalog.

---

## 1. Executive Summary

Bunkai TMS exposes a REST API under `/api/v1/` that supports two distinct types of callers: browser users who authenticate via Supabase magic-link and receive an httpOnly session cookie, and headless agents (CI pipelines, AI assistants, CLI tools) that sign in with credentials and receive a scoped Personal Access Token (PAT). Every piece of business data — workspaces, projects, modules, user stories, ATCs — lives in a Supabase PostgreSQL database protected by Row Level Security, which means the API cannot return data a caller has no business seeing, regardless of how endpoints are called.

The core product loop is authoring Acceptance Test Cases (ATCs): a QA engineer logs in, is onboarded to a workspace, navigates the project hierarchy, creates and edits ATCs linked to user stories and acceptance criteria, and manages team access by inviting colleagues. Every critical mutation — workspace creation, ATC save, invite acceptance — is backed by an atomic PostgreSQL RPC, meaning these operations either fully succeed or fully roll back. There are no partial writes on any critical path.

The API is intentionally narrow in Phase 1. Projects and modules are read-only from the API surface; they are populated via DB seed or direct Supabase migrations. Run execution (recording pass/fail results) is defined at the scope level (`run:execute` PAT scope) but has no endpoint yet. The gap between "data model capability" and "API surface" is the primary source of discovery gaps documented at the end of this file.

---

## 2. Permission & Auth Model

### Tier Table

| Tier | Who it applies to | How to acquire | Where enforced |
|------|-------------------|----------------|----------------|
| **Public** | Anyone — no credentials needed | No action required | Next.js route handler — no `requireAuth()` call |
| **Cookie (Browser)** | Any authenticated browser user | Magic-link OTP flow → session cookie set by `/auth/callback` | `lib/api/auth.ts: requireAuth()` cookie path + `middleware.ts` (page routes) |
| **Bearer PAT** | CLI tools, AI agents, CI pipelines | `POST /api/v1/auth/signin` returns `bk_pat_<prefix>.<secret>` | `lib/api/auth.ts: requireAuth()` bearer path → `lib/api/pat.ts` |
| **Role: Member+** | Active workspace members (viewer, member, admin, owner) | Implicit after joining a workspace | Supabase RLS (`bunkai_can_write_workspace`) |
| **Role: Admin+** | Workspace admins and owners | Role assigned at invite time or promoted by owner | RLS + handler-level check (`bunkai_is_workspace_admin`) |
| **Role: Owner** | Workspace creator (and anyone explicitly promoted) | `bunkai_bootstrap_workspace` assigns `role=owner` at creation | RLS + handler-level check (`bunkai_is_workspace_owner`) |

### PAT Scope Table

Bearer tokens carry explicit scopes. A call that needs a scope but the token doesn't carry it → 403.

| Scope | What it unlocks | Status |
|-------|-----------------|--------|
| `atc:read` | Read ATCs, steps, assertions, AC bindings | Active |
| `atc:write` | Create/update ATCs via `bunkai_save_atc` | Active |
| `run:execute` | Record test run results against ATCs | Planned — no endpoint yet (FEAT-033) |
| `workspace:admin` | Workspace-level admin operations | Active |

### Magic-Link Token Flow (Browser)

```
[User]                        [Next.js]                    [Supabase Auth]
  |                               |                               |
  | POST /api/v1/auth/magic-link  |                               |
  | { email, next: "/projects" }  |                               |
  +------------------------------>|                               |
  |                               | auth.signInWithOtp({ email }) |
  |                               +------------------------------>|
  |                               |                     Send OTP email
  |<-- { ok: true } -------------|<------------------------------|
  |                               |                               |
  | [User clicks email link]      |                               |
  |                               |                               |
  | GET /auth/callback?code=...   |                               |
  | &next=/projects               |                               |
  +------------------------------>|                               |
  |                               | exchangeCodeForSession(code)  |
  |                               +------------------------------>|
  |                               |<-- session + user ------------|
  |<-- Set-Cookie: sb-session ----|                               |
  |<-- 302 /projects -------------|                               |
```

### Bearer PAT Flow (CLI / Agent)

```
[CLI / Agent]                 [Next.js /api/v1/auth/signin]
  |                               |
  | POST /api/v1/auth/signin      |
  | { email, password,            |
  |   pat_name, pat_scopes,       |
  |   expires_in_days }           |
  +------------------------------>|
  |                               | supabase.signInWithPassword()
  |                               | INSERT access_tokens (prefix, scopes, expires_at)
  |                               | INSERT access_token_secrets (SHA-256 hash)
  |<-- { session, pat: {          |
  |       token: "bk_pat_<12>.<32>",
  |       scopes, expires_at } }  |
  |                               |
  | [All subsequent calls]        |
  | Authorization: Bearer bk_pat_+>  lib/api/auth.ts: requireAuth()
  |                               |  prefix lookup (O(1) index)
  |                               |  SHA-256 verify (access_token_secrets)
  |                               |  resolve userId + scopes
```

**Key rules**:
- Bearer tokens can NOT create other tokens (chicken-and-egg guard — `POST /api/v1/tokens` accepts cookie only)
- `requireScopeOrCookie()`: Cookie callers bypass scope checks (RLS enforces instead); Bearer callers must carry the required scope
- Token raw secret returned exactly once at creation and never again
- Revoked or expired tokens: 401 on any subsequent call

---

## 3. Critical Business Journeys

---

### Journey 1 — Browser Login via Magic-Link

**Business purpose**: Authenticate a QA engineer or workspace member with zero passwords — they receive a one-time link by email.

```
Client --> Middleware --> POST /api/v1/auth/magic-link --> Supabase Auth --> (Email sent)
                                                                                  |
Client --> GET /auth/callback?code=...&next=/projects --> exchangeCodeForSession --> Set-Cookie --> 302 /projects
```

**Narrative**:
1. User submits email on `/login` → `POST /api/v1/auth/magic-link` calls `supabase.auth.signInWithOtp()`.
2. Supabase sends OTP link; handler returns `{ ok: true }` immediately (fire-and-forget from API perspective).
3. User clicks email link → browser lands on `/auth/callback?code=...&next=/projects`.
4. Server calls `exchangeCodeForSession(code)` → sets httpOnly `supabase-auth-token` cookie.
5. `next` parameter validated: must start with `/`, must not start with `//` (open-redirect guard).
6. Valid session → 302 to `next`; failure → `/login?error=otp_exchange_failed`.

**Endpoints**: `POST /api/v1/auth/magic-link`, `GET /auth/callback`  
**Entities touched**: `auth.users` (Supabase managed), `magic_link_tokens`, `magic_link_token_secrets`  
**Feature IDs**: FEAT-001, FEAT-004  
**Failure mode**: Supabase email provider down → link never arrives; API returns 200 regardless (silent failure from user POV)

---

### Journey 2 — Headless Agent Sign-in + PAT Mint

**Business purpose**: A CI pipeline or AI agent authenticates and receives a scoped token to read/write ATCs without browser interaction.

```
Agent --> POST /api/v1/auth/signin --> supabase.signInWithPassword() --> Mint PAT (INSERT access_tokens + access_token_secrets)
      <-- { session, pat: { token: "bk_pat_...", scopes, expires_at } }

Agent (subsequent) --> Authorization: Bearer bk_pat_<prefix>.<secret>
               --> requireAuth() --> prefix lookup (O(1)) --> SHA-256 verify --> userId + scopes resolved
```

**Narrative**:
1. Agent posts `{ email, password, pat_name, pat_scopes, expires_in_days }` to `POST /api/v1/auth/signin`.
2. Supabase validates password credentials → returns session JWT.
3. Handler generates crypto-random 32-byte secret; stores SHA-256 hash in `access_token_secrets`, metadata in `access_tokens`.
4. Returns `{ session, pat: { token: "bk_pat_<12>.<32>", scopes, expires_at } }` — raw token shown once only.
5. Subsequent calls carry `Authorization: Bearer bk_pat_...`; `lib/api/pat.ts` resolves it via prefix index lookup + hash verify.
6. Scope mismatch → 403. Token revoked → 401. Token expired → 401.

**Endpoints**: `POST /api/v1/auth/signin`  
**Entities touched**: `auth.users`, `access_tokens`, `access_token_secrets`  
**Feature IDs**: FEAT-002, FEAT-005  
**Security note**: Agent must cache the raw token — no recovery endpoint exists. Tokens are scoped; a `atc:read`-only token cannot write.

---

### Journey 3 — Workspace Onboarding (First-Time Setup)

**Business purpose**: Authenticated user without a workspace creates one — the single gateway to all product features.

```
Browser --> POST /api/v1/workspaces { slug, name }
         --> requireAuth() (cookie)
         --> CALL bunkai_bootstrap_workspace(slug, name)
             [PostgreSQL transaction]:
               INSERT workspaces
               INSERT workspace_members (role=owner, status=active)
         <-- { id, slug, name, plan: "community" }
         --> 302 /projects
```

**Narrative**:
1. `OnboardingForm` submits `{ slug, name }` to `POST /api/v1/workspaces`.
2. Handler validates slug format (`^[a-z0-9][a-z0-9-]{1,38}[a-z0-9]$`) and blocks reserved values (`api`, `auth`, `admin`, `login`, etc.).
3. Calls `bunkai_bootstrap_workspace` RPC — one DB transaction creates workspace + owner membership atomically.
4. 409 Conflict if slug is already taken (human-readable error for form feedback).
5. Success → `{ id, slug, name, plan }` returned; UI redirects to `/projects`.

**Endpoints**: `POST /api/v1/workspaces`  
**Entities touched**: `workspaces`, `workspace_members` (data-map: Workspace Bootstrap Flow)  
**Feature IDs**: FEAT-010  
**Why it matters for QA**: If the RPC partial-fails (workspace created, membership not), the user is authenticated but locked out of all content — this is a catastrophic state with no self-recovery.

---

### Journey 4 — Team Invite + Acceptance

**Business purpose**: Workspace admin adds a collaborator who accepts via a one-time link — the only way to grow the team.

```
[Admin]
  --> POST /api/v1/workspaces/{id}/invites { email, role }
      requireAuth() + requireAdmin()
      INSERT workspace_invites (expires_at = now+7d)
      INSERT workspace_invite_secrets (SHA-256 hash)
  <-- { accept_url, raw_token }  [raw token shown once; admin copies link]

[Invitee]
  --> GET /invites/accept?token=...  (AcceptClient checks auth)
  --> POST /api/v1/invites/accept { token }
      hash(token) --> lookup workspace_invite_secrets
      validate: not revoked, not accepted, not expired, email match
      UPSERT workspace_members (status=active)
      stamp invite.accepted_at
  <-- 302 /projects
```

**Narrative**:
1. Admin issues invite: `POST /api/v1/workspaces/{id}/invites { email, role }`.
2. Handler generates crypto-random token; stores SHA-256 hash; returns `accept_url` + raw token (copied to clipboard — no email in Phase 1).
3. Invite expires in 7 days from creation.
4. Invitee navigates to `/invites/accept?token=...`; `AcceptClient` prompts auth if not signed in.
5. `POST /api/v1/invites/accept { token }`: hash → lookup → 5-condition validation chain (exists, not revoked, not accepted, not expired, email match case-insensitive).
6. On success: upsert `workspace_members (status=active)`, stamp `accepted_at` → 302 `/projects`.
7. Validation failure returns distinct status codes: 404 (invalid), 403 (email mismatch), 410 (expired), 409 (already accepted/revoked).

**Endpoints**: `POST /api/v1/workspaces/{id}/invites`, `GET /api/v1/workspaces/{id}/invites`, `POST /api/v1/invites/accept`, `DELETE /api/v1/workspaces/{id}/invites/{id}`, `POST /api/v1/workspaces/{id}/invites/{id}` (rotate)  
**Entities touched**: `workspace_invites`, `workspace_invite_secrets`, `workspace_members` (data-map: Team Invite Flow, Workspace Member Status)  
**Feature IDs**: FEAT-014, FEAT-015, FEAT-016, FEAT-017, FEAT-018  
**Critical edge cases**: Email mismatch → silently rejects (403); expired token → 410 with no self-service refresh (admin must rotate). MVP has no automated email — if admin loses the link, they must issue a new invite.

---

### Journey 5 — ATC Save (Core Product Mutation)

**Business purpose**: A QA engineer creates or updates a test case with steps, assertions, and acceptance-criteria bindings — the central authoring act in the product.

```
[AtcEditor (browser)]
  --> saveAtcAction (Next.js Server Action)
      --> CALL bunkai_save_atc(atcId, title, layer, tags, userStoryId, steps[], assertions[], acIds[])
          [PostgreSQL transaction]:
            DELETE atc_steps WHERE atc_id
            INSERT atc_steps (positions 0..N)
            DELETE atc_assertions WHERE atc_id
            INSERT atc_assertions
            DELETE atc_acceptance_criteria WHERE atc_id
            INSERT atc_acceptance_criteria (new AC bindings)
            UPDATE atcs SET title, layer, tags, version+1, updated_at
            TRIGGER: atcs_refresh_tsv (refreshes full-text search index)
            TRIGGER: atcs_set_updated_at
  <-- void
  --> revalidatePath (SSR re-render of ATC view)
```

**Narrative**:
1. Editor collects `{ title, layer, tags, userStoryId, steps[], assertions[], acIds[] }`.
2. Server Action calls `bunkai_save_atc` RPC — one transaction replaces ALL child records.
3. Steps and assertions are delete-then-reinsert (not diffed); positions are always 0..N after save.
4. AC bindings are also fully replaced — partial AC binding state cannot persist.
5. Version counter incremented on every save (optimistic locking groundwork).
6. Two triggers fire: `atcs_refresh_tsv` (keeps full-text search current), `atcs_set_updated_at`.
7. Server Action calls `revalidatePath` so the RSC view reflects the new state without a full page reload.

**Endpoints**: `saveAtcAction` (Next.js Server Action — not a REST endpoint; invoked by browser via React form action, not `fetch`)  
**Entities touched**: `atcs`, `atc_steps`, `atc_assertions`, `atc_acceptance_criteria`, `acceptance_criteria` (data-map: ATC Save Flow)  
**Feature IDs**: FEAT-023, FEAT-024, FEAT-025  
**Why it matters for QA**: Delete-then-reinsert means a failed INSERT mid-RPC (e.g., position uniqueness violation) would leave the ATC with no steps/assertions at all. The transaction rolls back atomically — but any bug that prevents the final UPDATE still wipes child records. This is the highest-blast-radius mutation in the product.

---

### Journey 6 — PAT Revocation (Security Lifecycle)

**Business purpose**: User or security process immediately invalidates a compromised or unused token — the sole security control once a PAT leaves the system.

```
[Owner / Security admin]
  --> DELETE /api/v1/tokens/{id}
      requireAuth() (cookie only — a Bearer token cannot revoke tokens)
      RLS: user_id = auth.uid() enforced
      UPDATE access_tokens SET revoked_at = now()   [soft-delete]
  <-- 204 No Content

[Subsequent Bearer call with revoked token]
  --> Authorization: Bearer bk_pat_<prefix>.<secret>
      lib/api/pat.ts: prefix lookup --> revoked_at IS NOT NULL --> 401 Unauthorized
```

**Narrative**:
1. Owner calls `DELETE /api/v1/tokens/{id}` with a valid cookie session.
2. RLS enforces ownership — forged IDs return 404 (indistinguishable from "not found").
3. Handler stamps `revoked_at = now()` (soft-delete — audit trail preserved in `access_tokens`).
4. Next call by the revoked token: `pat.ts` finds the prefix, hash verifies, then checks `revoked_at IS NOT NULL` → 401 immediately.
5. No recovery — revoked is terminal. User must mint a new PAT via `POST /api/v1/tokens`.

**Endpoints**: `DELETE /api/v1/tokens/{id}`  
**Entities touched**: `access_tokens`, `access_token_secrets` (data-map: PAT Lifecycle)  
**Feature IDs**: FEAT-007  
**Why it matters for QA**: The gap between revoke and enforcement must be zero — any caching of token validity state would create a window where a revoked token still works. Tests must verify enforcement is synchronous, not eventually consistent.

---

## 4. Architecture Behind the API

```
+---------------------------------------------+
|  Callers                                     |
|  [Browser (cookie)]  [CLI/Agent (Bearer PAT)]|
+------------------+---------------------------+
                   |
                   | HTTPS
                   |
+------------------v---------------------------+
|  Next.js 15 (Vercel Edge / Node)             |
|                                              |
|  +-------------------+  +-----------------+ |
|  |  App Router (FE)  |  |  Route Handlers | |
|  |  RSC + Client     |  |  /api/v1/*      | |
|  |  Components       |  |                 | |
|  |  Server Actions   |  |  lib/api/       | |
|  +-------------------+  |  auth.ts        | |
|                          |  pat.ts         | |
|  middleware.ts           |  idempotency.ts | |
|  (session refresh +      |  handler.ts     | |
|   route protection)      +-----------------+ |
+------------------+---------------------------+
                   |
                   | Supabase JS SDK (@supabase/ssr)
                   |
+------------------v---------------------------+
|  Supabase Cloud                              |
|                                              |
|  +--------------------+  +--------------+   |
|  |  PostgreSQL 16     |  |  Auth        |   |
|  |  18 tables         |  |  Magic-link  |   |
|  |  12 migrations     |  |  OTP email   |   |
|  |  RLS on all tables |  |  auth.users  |   |
|  |  4 SECURITY DEF.   |  |  Session mgmt|   |
|  |  helper functions  |  +--------------+   |
|  |  2 RPCs (DEFINER)  |                     |
|  |  2 triggers        |                     |
|  +--------------------+                     |
+----------------------------------------------+
```

| Component | Role | Persistence / Integrations | Why it matters for QA |
|-----------|------|-----------------------------|----------------------|
| Next.js Route Handlers (`/api/v1/`) | REST API — request validation (Zod), auth enforcement, response shaping | Supabase DB + Auth | Entry point for all API tests; `lib/api/auth.ts` is the shared auth gate |
| Next.js Server Actions | Browser-to-server mutations without REST (ATC save) | Supabase RPC `bunkai_save_atc` | Not testable via HTTP — requires browser context or direct Supabase client call |
| `middleware.ts` | Session refresh + page-level route protection | Supabase Auth (cookie) | Redirect loop risks; unauthenticated users hitting `/projects` must land on `/login?next=...` |
| `lib/api/auth.ts` | Dual-mode auth gate (`requireAuth()`) | `access_tokens`, `access_token_secrets` | All protected endpoints funnel through here; a bug here is universal |
| `lib/api/pat.ts` | PAT validation (prefix lookup → hash verify → scope check) | `access_tokens`, `access_token_secrets` | O(1) prefix lookup; SHA-256 comparison — timing-safe? Gap to verify |
| `lib/api/idempotency.ts` | POST replay deduplication (24h TTL) | `idempotency_keys` table | Clients must send `Idempotency-Key` header; which endpoints enforce it is unclear from Phase 1 |
| Supabase PostgreSQL | Single source of truth for all business data | 18 tables, RLS, 4 SECURITY DEFINER functions | RLS = last line of defense; bugs in helper functions expose cross-tenant data |
| Supabase Auth | OTP email + session cookie lifecycle | `auth.users`, `magic_link_tokens`, email provider | External dependency — failures are silent from API POV (magic-link returns 200 regardless) |
| Vercel Edge Functions | Hosting + deployment | Git push → Vercel webhook | No `vercel.json` found; deployment config unverified (discovery gap) |

---

## 5. External Integrations

| Service | Trigger | Direction | Failure mode (user-visible) | Journeys affected |
|---------|---------|-----------|------------------------------|-------------------|
| **Supabase Auth** | `POST /api/v1/auth/magic-link` — any OTP request | Outbound sync | Auth service down → magic-link silently never arrives; API returns 200 | Journey 1 (Magic-Link Login) |
| **Supabase Auth** | `GET /auth/callback?code=...` — OTP redemption | Outbound sync | Exchange failure → `/login?error=otp_exchange_failed` | Journey 1 |
| **Supabase Auth** | `POST /api/v1/auth/signin` — password sign-in | Outbound sync | Wrong credentials → 401; service down → 500 | Journey 2 (Headless Sign-in) |
| **Supabase PostgreSQL** | Every API call that reads or writes data | Outbound (SDK) | DB unreachable → 500 on all protected endpoints | All journeys |
| **Supabase PostgreSQL** | `saveAtcAction` — `bunkai_save_atc` RPC | Outbound (RPC) | RPC failure mid-transaction → full rollback; ATC state unchanged | Journey 5 (ATC Save) |
| **Supabase PostgreSQL** | `POST /api/v1/workspaces` — `bunkai_bootstrap_workspace` RPC | Outbound (RPC) | RPC failure → workspace not created; user stays on `/onboarding` | Journey 3 (Workspace Onboarding) |
| **Vercel** | Git push → auto-deploy | Inbound webhook | Deploy failure → stale code on edge; no rollback found in source | All journeys |
| **Jira (BK project)** | `user_stories.external_id` linkage | Bidirectional (manual/read) | No live API call from app — Jira key stored as string; no sync failure mode | FEAT-019, FEAT-020 |
| **Resend** | Invite email delivery | Outbound sync (planned) | Not wired — MVP uses clipboard copy instead | Journey 4 (Team Invite) — FEAT-034 |
| **n8n** | Workflow automation (planned) | Outbound REST (planned) | Not wired — `N8N_API_URL` env var present but no code | FEAT-035 |

---

## 6. Cross-References

### Data-map entities this API exposes

| Entity group | Journeys / API | Anchor in data-map |
|---|---|---|
| `auth.users`, `magic_link_tokens`, `magic_link_token_secrets` | Journey 1, Journey 2 | business-data-map.md §Flow 1, §Flow 2 |
| `access_tokens`, `access_token_secrets` | Journey 2, Journey 6 | business-data-map.md §Flow 2, §PAT Lifecycle |
| `workspaces`, `workspace_members` | Journey 3 | business-data-map.md §Flow 3, §Workspace Member Status |
| `workspace_invites`, `workspace_invite_secrets` | Journey 4 | business-data-map.md §Flow 5, §Workspace Invite Lifecycle |
| `atcs`, `atc_steps`, `atc_assertions`, `atc_acceptance_criteria` | Journey 5 | business-data-map.md §Flow 4, §ATC Status |
| `idempotency_keys`, `activity_log`, `feature_flags` | FEAT-030, FEAT-031, FEAT-032 | business-data-map.md §Scheduled Jobs / Webhooks |

### Feature-map features this API backs

| Domain | Feature IDs | Anchor in feature-map |
|---|---|---|
| Authentication | FEAT-001, FEAT-002, FEAT-003, FEAT-004 | business-feature-map.md §Domain: Authentication |
| Personal Access Tokens | FEAT-005, FEAT-006, FEAT-007 | business-feature-map.md §Domain: Personal Access Tokens |
| Identity / Me | FEAT-008, FEAT-009 | business-feature-map.md §Domain: Identity / Me |
| Workspace Management | FEAT-010, FEAT-011, FEAT-012, FEAT-013 | business-feature-map.md §Domain: Workspace Management |
| Team Management | FEAT-014, FEAT-015, FEAT-016, FEAT-017, FEAT-018 | business-feature-map.md §Domain: Team Management |
| ATC Management | FEAT-023, FEAT-024, FEAT-025, FEAT-026 | business-feature-map.md §Domain: ATC Management |
| Observability & Platform | FEAT-029, FEAT-030, FEAT-031 | business-feature-map.md §Domain: Observability |

### API spec and types

| Resource | Location | Notes |
|---|---|---|
| OpenAPI 3.0 spec (live) | `GET /api/openapi` | Runtime-generated via `@asteasolutions/zod-to-openapi`; reflects deployed code, no build step |
| Interactive API docs | `GET /api/docs` | Scalar UI — no auth required |
| TypeScript API types | `api/openapi-types.ts` (after `bun run api:sync`) | Synced from live spec |
| Route sidecar schemas | `app/api/v1/**/route.openapi.ts` | Zod schemas per-endpoint; source of truth for validation |

---

## 7. Discovery Gaps

These behaviors could not be fully verified from Phase 1 source and remain open questions for test design:

| Gap | Impact on QA | Recommended Action |
|-----|-------------|-------------------|
| **Idempotency enforcement** — `idempotency_keys` table exists but it's unclear which POST endpoints actually enforce the `Idempotency-Key` header | Cannot design replay tests without knowing which routes opt in | Grep `lib/api/idempotency.ts` usages across route handlers |
| **Activity log write coverage** — `activity_log` table exists with `entity_type/action/payload`; unclear which mutations write to it automatically | Cannot verify audit completeness | Grep `activity_log` inserts in route handlers + RPC bodies |
| **`bk_active_ws` cookie on logout** — does the logout flow (if any) clear the active-workspace cookie? | Could leak workspace context to a subsequent user on shared machine | Trace logout route + session teardown |
| **`bunkai_save_atc` partial failure modes** — what happens if the DELETE succeeds but INSERT fails mid-RPC? PostgreSQL rolls back the transaction, but is the RPC `SECURITY DEFINER`? Can a low-privilege user trigger it? | Determines blast radius of a failed save | Read `supabase/migrations/0007_save_atc.sql` — check `SECURITY DEFINER` flag |
| **Timing-safe PAT comparison** — `lib/api/pat.ts` does prefix lookup + SHA-256 verify; whether the comparison is timing-safe (constant-time) is unverified | Timing attack on token validation (low probability but relevant for a security-sensitive product) | Read `lib/api/pat.ts` hash comparison implementation |
| **Scope enforcement completeness** — `requireScopeOrCookie()` is the enforcement function; which endpoints call it and which only call `requireAuth()` without scope check is not fully traced | Bearer tokens with broad scopes might access endpoints not intended for programmatic use | Audit all `requireAuth()` vs `requireScopeOrCookie()` call sites in `/api/v1/` handlers |
| **No ATC deletion** — no delete endpoint or UI button exists; ATCs can be abandoned but not removed | Test data hygiene in automation — no teardown path for ATCs created during tests | Confirm with product team: intentional soft-preservation? Needs a `DELETE /api/v1/atcs/{id}` for QA |
| **Vercel deployment config** — no `vercel.json` found; edge function region, timeout, and environment variable mapping are unverified | Integration tests might behave differently on Vercel edge vs. local Node | Inspect Vercel dashboard or request `vercel.json` from dev team |
| **Run execution API** — `run:execute` PAT scope defined, `atcs.status` enum exists, but no endpoint under `/api/v1/runs/` or similar | Cannot automate test result recording until this is built (FEAT-033) | Track on product roadmap; test the scope definition exists but endpoint returns 404 |
| **Resend transactional email** — invite delivery is clipboard-only in Phase 1; Resend not in `package.json` | Team invite E2E tests cannot verify email delivery (they must test the token-acceptance path only) | Proceed with token-based invite tests; monitor for FEAT-034 implementation |

---

*Cross-reference: `business-data-map.md` (entities, state machines, flows) · `business-feature-map.md` (feature catalog, CRUD matrix, integrations)*  
*Update after: new auth schemes added · critical route group added · external integration wired (Resend, n8n) · run-execution API (FEAT-033) lands*
