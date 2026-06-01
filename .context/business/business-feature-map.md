# Business Feature Map — Bunkai TMS

> **Discovery date**: 2026-05-31
> **Generated from**: API route handlers (OpenAPI specs + handlers), Next.js page components, DB migrations, functional-specs.md, business-data-map.md
> **Refresh command**: `/business-feature-map` (run after new routes, UI pages, or migrations are added)
> **Purpose**: Complete feature inventory for test design — what the system *does*, not just what it *stores*. Pair with `business-data-map.md`.

---

## 1. Inventory Summary

| Domain          | Features | Status        |
|-----------------|----------|---------------|
| Authentication  | 4        | Stable        |
| Personal Access Tokens | 3 | Stable     |
| Identity / Me   | 2        | Stable        |
| Workspace Management | 4   | Stable        |
| Team Management (Invites) | 5 | Stable   |
| Project & Module Browsing | 4 | Stable   |
| ATC Management  | 4        | Stable        |
| API & Developer Surface | 3 | Stable     |
| Observability & Platform | 3 | Stable    |
| Planned / WIP   | 4        | In Development |
| **Total**       | **36**   |               |

---

## 2. Feature Catalog

---

### Domain: Authentication

#### Feature: Magic-Link Login (Browser)

| Aspect           | Value                                                       |
|------------------|-------------------------------------------------------------|
| **ID**           | FEAT-001                                                    |
| **Status**       | Stable                                                      |
| **Endpoints**    | `POST /api/v1/auth/magic-link`, `GET /auth/callback`        |
| **UI**           | `/login` — `MagicLinkForm` component                        |
| **Users**        | Any unauthenticated user                                    |
| **Dependencies** | Supabase Auth (OTP email), Supabase session cookie          |
| **Evidence**     | `app/api/v1/auth/magic-link/route.ts`, `app/auth/callback/route.ts`, `app/(auth)/login/` |

**Capabilities:**
- [x] Submit email to request OTP link
- [x] Receive magic-link email (via Supabase)
- [x] Exchange OTP code for session cookie via `/auth/callback`
- [x] Redirect to `next` param (default: `/projects`) after auth
- [x] Guard open-redirect: `next` must start with `/` and not `//`
- [ ] Resend magic-link (no retry UI — user must re-submit form)

---

#### Feature: Headless Sign-in + PAT (CLI / Agent)

| Aspect           | Value                                                       |
|------------------|-------------------------------------------------------------|
| **ID**           | FEAT-002                                                    |
| **Status**       | Stable                                                      |
| **Endpoints**    | `POST /api/v1/auth/signin`                                  |
| **UI**           | None (API-only, for CLI and test agents)                    |
| **Users**        | Automation agents, QA testers via CLI                       |
| **Dependencies** | Supabase Auth (password sign-in), `access_tokens` table     |
| **Evidence**     | `app/api/v1/auth/signin/route.ts`, `lib/api/pat.ts`         |

**Capabilities:**
- [x] Authenticate with email + password
- [x] Optionally mint a PAT in the same request (name, scopes, expiry)
- [x] Return session JWT + PAT raw token (shown once)
- [x] Scope validation: `atc:read | atc:write | run:execute | workspace:admin`

---

#### Feature: Headless Sign-up + PAT (QA / CI)

| Aspect           | Value                                                       |
|------------------|-------------------------------------------------------------|
| **ID**           | FEAT-003                                                    |
| **Status**       | Stable (QA path only — bypasses email confirmation)         |
| **Endpoints**    | `POST /api/v1/auth/signup`                                  |
| **UI**           | None (API-only)                                             |
| **Users**        | QA automation, CI pipelines                                 |
| **Dependencies** | Supabase Admin API (`email_confirm: true`)                  |
| **Evidence**     | `app/api/v1/auth/signup/route.ts`                           |

**Capabilities:**
- [x] Create user without email click (admin bypass — CI/QA use only)
- [x] Auto-sign-in after creation
- [x] Mint PAT in the same request
- [x] Idempotency guard: 409 if email already exists

---

#### Feature: OAuth Callback Handler

| Aspect           | Value                                                       |
|------------------|-------------------------------------------------------------|
| **ID**           | FEAT-004                                                    |
| **Status**       | Stable                                                      |
| **Endpoints**    | `GET /auth/callback?code=...&next=...`                      |
| **UI**           | None (redirect-only server route)                           |
| **Users**        | Browser (redirected by Supabase after OTP click)            |
| **Dependencies** | Supabase Auth (`exchangeCodeForSession`)                    |
| **Evidence**     | `app/auth/callback/route.ts`                                |

**Capabilities:**
- [x] Exchange Supabase OTP code for session
- [x] Set httpOnly session cookie
- [x] Redirect to `next` (validated) or fallback to `/projects`
- [x] On failure → redirect to `/login?error=otp_exchange_failed`

---

### Domain: Personal Access Tokens (PATs)

#### Feature: PAT Issuance

| Aspect           | Value                                                       |
|------------------|-------------------------------------------------------------|
| **ID**           | FEAT-005                                                    |
| **Status**       | Stable                                                      |
| **Endpoints**    | `POST /api/v1/tokens`                                       |
| **UI**           | None visible in Phase 1 (minted via sign-in or API call)   |
| **Users**        | Authenticated users with active cookie session              |
| **Dependencies** | `access_tokens`, `access_token_secrets` tables              |
| **Evidence**     | `app/api/v1/tokens/route.ts`, `lib/api/pat.ts`              |

**Capabilities:**
- [x] Mint PAT with name, scopes, optional workspace scope, optional expiry (max 365 days)
- [x] Token format: `bk_pat_<12-char-prefix>.<32-byte-base64url-secret>`
- [x] Store only SHA-256 hash — raw token returned once and never again
- [x] Cookie auth required — PATs cannot self-replicate (chicken-and-egg guard)

---

#### Feature: PAT Listing

| Aspect           | Value                                                       |
|------------------|-------------------------------------------------------------|
| **ID**           | FEAT-006                                                    |
| **Status**       | Stable                                                      |
| **Endpoints**    | `GET /api/v1/tokens`                                        |
| **UI**           | None in Phase 1                                             |
| **Users**        | Token owner                                                 |
| **Dependencies** | `access_tokens` table, RLS                                  |
| **Evidence**     | `app/api/v1/tokens/route.ts`                                |

**Capabilities:**
- [x] List all caller-owned tokens (metadata only — no raw secret)
- [x] Shows: name, scopes, workspace_id, token_prefix, expires_at, revoked_at, last_used_at

---

#### Feature: PAT Revocation

| Aspect           | Value                                                       |
|------------------|-------------------------------------------------------------|
| **ID**           | FEAT-007                                                    |
| **Status**       | Stable                                                      |
| **Endpoints**    | `DELETE /api/v1/tokens/{id}`                                |
| **UI**           | None in Phase 1                                             |
| **Users**        | Token owner                                                 |
| **Dependencies** | `access_tokens` table (soft-delete via `revoked_at`)        |
| **Evidence**     | `app/api/v1/tokens/[id]/route.ts`                           |

**Capabilities:**
- [x] Soft-delete: stamp `revoked_at` (preserves audit trail)
- [x] Revoked token immediately rejected by bearer middleware
- [x] RLS enforces ownership — forged IDs return 404

---

### Domain: Identity / Me

#### Feature: Principal Introspection

| Aspect           | Value                                                       |
|------------------|-------------------------------------------------------------|
| **ID**           | FEAT-008                                                    |
| **Status**       | Stable                                                      |
| **Endpoints**    | `GET /api/v1/me`                                            |
| **UI**           | None (used by frontend navigation + CLI)                    |
| **Users**        | Any authenticated caller (cookie or bearer)                 |
| **Dependencies** | Supabase Auth, `workspace_members` table                    |
| **Evidence**     | `app/api/v1/me/route.ts`                                    |

**Capabilities:**
- [x] Return user id + email
- [x] Return all workspaces the caller belongs to (RLS-filtered)
- [x] Return active workspace id (from `bk_active_ws` cookie)
- [x] Return auth source: `cookie | bearer` + token scopes

---

#### Feature: Active Workspace Selection

| Aspect           | Value                                                       |
|------------------|-------------------------------------------------------------|
| **ID**           | FEAT-009                                                    |
| **Status**       | Stable                                                      |
| **Endpoints**    | `POST /api/v1/me/active-workspace`                          |
| **UI**           | Workspace switcher (implicit — no dedicated page found)     |
| **Users**        | Any authenticated user                                      |
| **Dependencies** | `workspace_members`, `bk_active_ws` httpOnly cookie         |
| **Evidence**     | `app/api/v1/me/active-workspace/route.ts`                   |

**Capabilities:**
- [x] Validate caller is an active member of the target workspace
- [x] Set `bk_active_ws` httpOnly cookie (does not modify JWT)
- [x] 403 if caller is not a member

---

### Domain: Workspace Management

#### Feature: Workspace Creation (Onboarding)

| Aspect           | Value                                                       |
|------------------|-------------------------------------------------------------|
| **ID**           | FEAT-010                                                    |
| **Status**       | Stable                                                      |
| **Endpoints**    | `POST /api/v1/workspaces` (backed by `bunkai_bootstrap_workspace` RPC) |
| **UI**           | `/onboarding` — `OnboardingForm` component                  |
| **Users**        | Authenticated users who have no workspace yet               |
| **Dependencies** | `workspaces`, `workspace_members`, `bunkai_bootstrap_workspace` (SECURITY DEFINER) |
| **Evidence**     | `app/api/v1/workspaces/route.ts`, `app/(app)/onboarding/`  |

**Capabilities:**
- [x] Create workspace with `{ slug, name }`
- [x] Atomic RPC: workspace + owner membership in one transaction
- [x] Slug validation: `^[a-z0-9][a-z0-9-]{1,38}[a-z0-9]$`
- [x] Reserved slug block: `api`, `auth`, `admin`, `login`, etc.
- [x] 409 on slug collision with human-readable error
- [x] Plans: `community | cloud | enterprise` (default `community`)
- [ ] Slug rotation (post-MVP — name-only is mutable today)

---

#### Feature: Workspace Listing

| Aspect           | Value                                                       |
|------------------|-------------------------------------------------------------|
| **ID**           | FEAT-011                                                    |
| **Status**       | Stable                                                      |
| **Endpoints**    | `GET /api/v1/workspaces`                                    |
| **UI**           | Navigation sidebar (implicit)                               |
| **Users**        | Any authenticated member                                    |
| **Dependencies** | `workspace_members` (RLS-filtered)                          |
| **Evidence**     | `app/api/v1/workspaces/route.ts`                            |

**Capabilities:**
- [x] Return all workspaces where caller has active membership

---

#### Feature: Workspace Detail

| Aspect           | Value                                                       |
|------------------|-------------------------------------------------------------|
| **ID**           | FEAT-012                                                    |
| **Status**       | Stable                                                      |
| **Endpoints**    | `GET /api/v1/workspaces/{id}`                               |
| **UI**           | None dedicated (data used by member pages)                  |
| **Users**        | Any active member                                           |
| **Dependencies** | `workspaces`, RLS                                           |
| **Evidence**     | `app/api/v1/workspaces/[id]/route.ts`                       |

**Capabilities:**
- [x] Return single workspace (id, slug, name, plan, owner, created_at)
- [x] 404 if not found or caller is not a member

---

#### Feature: Workspace Metadata Update

| Aspect           | Value                                                       |
|------------------|-------------------------------------------------------------|
| **ID**           | FEAT-013                                                    |
| **Status**       | Stable (name-only; slug rotation post-MVP)                  |
| **Endpoints**    | `PATCH /api/v1/workspaces/{id}`                             |
| **UI**           | None found in Phase 1                                       |
| **Users**        | Workspace owner only                                        |
| **Dependencies** | `workspaces` table                                          |
| **Evidence**     | `app/api/v1/workspaces/[id]/route.ts`                       |

**Capabilities:**
- [x] Update workspace `name`
- [ ] Update workspace `slug` (post-MVP)
- [x] 403 if caller is not owner

---

### Domain: Team Management (Invites)

#### Feature: Invite Issuance

| Aspect           | Value                                                       |
|------------------|-------------------------------------------------------------|
| **ID**           | FEAT-014                                                    |
| **Status**       | Stable (email delivery pending Resend integration)          |
| **Endpoints**    | `POST /api/v1/workspaces/{id}/invites`                      |
| **UI**           | `/workspaces/[id]/members` — `MembersClient` form           |
| **Users**        | Workspace admin or owner                                    |
| **Dependencies** | `workspace_invites`, `workspace_invite_secrets`             |
| **Evidence**     | `app/api/v1/workspaces/[id]/invites/route.ts`, `app/(app)/workspaces/[id]/members/` |

**Capabilities:**
- [x] Issue invite to email with role (`viewer | member | admin`)
- [x] Generate one-time token (crypto-random + SHA-256 hash stored)
- [x] Set expiry at `now() + 7 days`
- [x] Return `accept_url` + raw token (shown once)
- [x] UI copies `accept_url` to clipboard automatically
- [ ] Automated email delivery (Resend — planned, not wired)

---

#### Feature: Invite Listing

| Aspect           | Value                                                       |
|------------------|-------------------------------------------------------------|
| **ID**           | FEAT-015                                                    |
| **Status**       | Stable                                                      |
| **Endpoints**    | `GET /api/v1/workspaces/{id}/invites`                       |
| **UI**           | `/workspaces/[id]/members` — pending invites section        |
| **Users**        | Workspace admin or owner                                    |
| **Dependencies** | `workspace_invites` table, RLS                              |
| **Evidence**     | `app/api/v1/workspaces/[id]/invites/route.ts`               |

**Capabilities:**
- [x] List invites with status derived: `pending | accepted | revoked | expired`
- [x] Show email, role, expiry, acceptance/revocation timestamps

---

#### Feature: Invite Acceptance

| Aspect           | Value                                                       |
|------------------|-------------------------------------------------------------|
| **ID**           | FEAT-016                                                    |
| **Status**       | Stable                                                      |
| **Endpoints**    | `POST /api/v1/invites/accept`                               |
| **UI**           | `/invites/accept?token=...` — `AcceptClient` component      |
| **Users**        | Invited user (must be authenticated, email must match)      |
| **Dependencies** | `workspace_invites`, `workspace_invite_secrets`, `workspace_members` |
| **Evidence**     | `app/api/v1/invites/accept/route.ts`, `app/invites/accept/` |

**Capabilities:**
- [x] Hash token → lookup invite
- [x] Validate: not revoked, not accepted, not expired, email match (case-insensitive)
- [x] Upsert `workspace_members (status=active)`
- [x] Stamp `accepted_at` + `accepted_by_user_id`
- [x] Redirect to `/projects` on success
- [x] Distinct error codes: 404 (invalid), 403 (email mismatch), 409 (already accepted/revoked/expired)

---

#### Feature: Invite Revocation

| Aspect           | Value                                                       |
|------------------|-------------------------------------------------------------|
| **ID**           | FEAT-017                                                    |
| **Status**       | Stable                                                      |
| **Endpoints**    | `DELETE /api/v1/workspaces/{id}/invites/{inviteId}`         |
| **UI**           | `/workspaces/[id]/members` — "Revoke" button per invite     |
| **Users**        | Workspace admin or owner                                    |
| **Dependencies** | `workspace_invites`                                         |
| **Evidence**     | `app/api/v1/workspaces/[id]/invites/[inviteId]/route.ts`    |

**Capabilities:**
- [x] Stamp `revoked_at = now()` — token immediately invalid
- [x] UI refreshes invite list after revocation

---

#### Feature: Invite Rotation (Resend)

| Aspect           | Value                                                       |
|------------------|-------------------------------------------------------------|
| **ID**           | FEAT-018                                                    |
| **Status**       | Stable                                                      |
| **Endpoints**    | `POST /api/v1/workspaces/{id}/invites/{inviteId}`           |
| **UI**           | `/workspaces/[id]/members` — "Rotate link" button           |
| **Users**        | Workspace admin or owner                                    |
| **Dependencies** | `workspace_invites`, `workspace_invite_secrets`             |
| **Evidence**     | `app/api/v1/workspaces/[id]/invites/[inviteId]/route.ts`    |

**Capabilities:**
- [x] Revoke old token + generate new secret
- [x] Extend expiry by 7 days from now
- [x] Clear prior acceptance/revocation state
- [x] Return new `accept_url` — UI copies to clipboard

---

### Domain: Project & Module Browsing

#### Feature: Project List View

| Aspect           | Value                                                       |
|------------------|-------------------------------------------------------------|
| **ID**           | FEAT-019                                                    |
| **Status**       | Stable (read-only; no API creation endpoint in Phase 1)     |
| **Endpoints**    | None (server-rendered via Supabase client direct query)     |
| **UI**           | `/projects` page                                            |
| **Users**        | Any authenticated workspace member                          |
| **Dependencies** | `projects` table, active workspace context                  |
| **Evidence**     | `app/(app)/projects/page.tsx`                               |

**Capabilities:**
- [x] List projects belonging to the active workspace
- [ ] Create project via UI (no API endpoint — discovery gap)

---

#### Feature: Project Detail / ATC Table

| Aspect           | Value                                                       |
|------------------|-------------------------------------------------------------|
| **ID**           | FEAT-020                                                    |
| **Status**       | Stable                                                      |
| **Endpoints**    | None (server-rendered)                                      |
| **UI**           | `/projects/[projectSlug]` — `AtcTable` component            |
| **Users**        | Any authenticated workspace member                          |
| **Dependencies** | `atcs`, `modules`, `user_stories` tables                    |
| **Evidence**     | `app/(app)/projects/[projectSlug]/page.tsx`, `components/atcs/AtcTable.tsx` |

**Capabilities:**
- [x] Display ATCs for a project in table view
- [x] Show ATC status with color coding
- [x] Navigate to ATC editor per row

---

#### Feature: Module Tree Navigation

| Aspect           | Value                                                       |
|------------------|-------------------------------------------------------------|
| **ID**           | FEAT-021                                                    |
| **Status**       | Stable (read-only; no CRUD API endpoints in Phase 1)        |
| **Endpoints**    | None (via `lib/tree.ts` + direct Supabase queries)          |
| **UI**           | Sidebar on `/projects/[projectSlug]`                        |
| **Users**        | Any authenticated workspace member                          |
| **Dependencies** | `modules` table (self-referential tree, max depth 6)        |
| **Evidence**     | `lib/tree.ts`, `app/(app)/projects/[projectSlug]/page.tsx`  |

**Capabilities:**
- [x] Render module hierarchy as collapsible tree
- [x] Sort siblings by `position`
- [x] Show materialized path (`Feature/Login/UI`)
- [ ] Create / rename / delete modules (no API — discovery gap)

---

#### Feature: ATC Full-Text Search

| Aspect           | Value                                                       |
|------------------|-------------------------------------------------------------|
| **ID**           | FEAT-022                                                    |
| **Status**       | Stable (DB infrastructure ready; UI surface TBD)            |
| **Endpoints**    | None dedicated — available via Supabase `ts_query` on `atcs.tsv` |
| **UI**           | Not found in Phase 1                                        |
| **Users**        | Any workspace member                                        |
| **Dependencies** | `atcs.tsv` tsvector, GIN index, `atcs_refresh_tsv` trigger |
| **Evidence**     | `supabase/migrations/0004_atcs.sql`                         |

**Capabilities:**
- [x] Full-text index on ATC title + tags (auto-refreshed by trigger)
- [x] GIN index for sub-millisecond search
- [ ] UI search box (not implemented in Phase 1)

---

### Domain: ATC Management

#### Feature: ATC Editor (Create / Edit)

| Aspect           | Value                                                       |
|------------------|-------------------------------------------------------------|
| **ID**           | FEAT-023                                                    |
| **Status**       | Stable                                                      |
| **Endpoints**    | Server Action `saveAtcAction` → `bunkai_save_atc` RPC       |
| **UI**           | `/projects/[projectSlug]/atcs/[atcId]` — `AtcEditor` component |
| **Users**        | Authenticated workspace member                              |
| **Dependencies** | `atcs`, `atc_steps`, `atc_assertions`, `atc_acceptance_criteria`, `bunkai_save_atc` RPC |
| **Evidence**     | `app/(app)/projects/[projectSlug]/atcs/[atcId]/`, `components/atcs/AtcEditor.tsx` |

**Capabilities:**
- [x] Edit ATC title, layer (`UI | API | Unit`), tags
- [x] Write steps in Markdown (parsed by `lib/atc-parse`)
- [x] Write assertions in YAML (parsed by `lib/atc-parse`)
- [x] Atomic save: delete-then-reinsert steps + assertions + AC bindings in one RPC transaction
- [x] Version counter incremented on every save (optimistic lock prep)
- [x] Validation: title required, user story required, ≥1 AC binding required
- [x] `revalidatePath` triggers SSR rerender after save

---

#### Feature: ATC AC Anchoring

| Aspect           | Value                                                       |
|------------------|-------------------------------------------------------------|
| **ID**           | FEAT-024                                                    |
| **Status**       | Stable                                                      |
| **Endpoints**    | Part of `saveAtcAction` → `bunkai_save_atc` RPC             |
| **UI**           | `AnchoringPanel` component inside `AtcEditor`               |
| **Users**        | Authenticated workspace member                              |
| **Dependencies** | `atc_acceptance_criteria` (M:N join), `user_stories`, `acceptance_criteria` |
| **Evidence**     | `components/atcs/AnchoringPanel.tsx`, `app/(app)/projects/[projectSlug]/atcs/[atcId]/page.tsx` |

**Capabilities:**
- [x] Browse all user stories + their ACs within the project
- [x] Select one user story to anchor the ATC
- [x] Bind multiple ACs from that story to the ATC
- [x] AC bindings replaced atomically on save (no partial updates)

---

#### Feature: ATC Step Authoring

| Aspect           | Value                                                       |
|------------------|-------------------------------------------------------------|
| **ID**           | FEAT-025                                                    |
| **Status**       | Stable                                                      |
| **Endpoints**    | Part of `saveAtcAction`                                     |
| **UI**           | `StepEditor` + Monaco editor inside `AtcEditor`             |
| **Users**        | Authenticated workspace member                              |
| **Dependencies** | `atc_steps` table, `lib/atc-parse` (Markdown parser)       |
| **Evidence**     | `components/atcs/StepEditor.tsx`, `lib/atc-parse.ts`        |

**Capabilities:**
- [x] Author steps in Markdown with Monaco editor
- [x] Parse `content | input_data | expected` from Markdown tables
- [x] Steps assigned positions 0..N on save

---

#### Feature: ATC Status Lifecycle

| Aspect           | Value                                                       |
|------------------|-------------------------------------------------------------|
| **ID**           | FEAT-026                                                    |
| **Status**       | Stable (DB infrastructure); no dedicated run-execution API  |
| **Endpoints**    | None in Phase 1 (status field writable via editor save)     |
| **UI**           | Status badges in `AtcTable` + `AtcEditor`                   |
| **Users**        | Authenticated workspace member                              |
| **Dependencies** | `atcs.status` enum: `unrun|running|pass|fail|blocked|skipped` |
| **Evidence**     | `supabase/migrations/0004_atcs.sql`, `business-data-map.md` |

**Capabilities:**
- [x] Display status with color coding (`pass=#2fb673`, `fail=#e5484d`, `blocked=#e8a838`, `skipped=#8a91a0`, `running=#4f8cf7`)
- [x] Manual status transitions (app-layer, no DB constraint on order)
- [x] Reset to `unrun` for re-runs
- [ ] Automated status update via run-execution API (planned — FEAT-033)

---

### Domain: API & Developer Surface

#### Feature: OpenAPI Spec (Live Generation)

| Aspect           | Value                                                       |
|------------------|-------------------------------------------------------------|
| **ID**           | FEAT-027                                                    |
| **Status**       | Stable                                                      |
| **Endpoints**    | `GET /api/openapi`                                          |
| **UI**           | None (JSON response)                                        |
| **Users**        | API clients, OpenAPI tools, CI pipelines                    |
| **Dependencies** | `@asteasolutions/zod-to-openapi`, all `route.openapi.ts` files |
| **Evidence**     | `app/api/openapi/route.ts`, `lib/openapi/registry.ts`       |

**Capabilities:**
- [x] Generate OpenAPI 3.0 JSON at runtime from Zod schemas
- [x] All routes registered via `route.openapi.ts` sidecar pattern
- [x] No build step required — always reflects deployed code

---

#### Feature: Interactive API Docs

| Aspect           | Value                                                       |
|------------------|-------------------------------------------------------------|
| **ID**           | FEAT-028                                                    |
| **Status**       | Stable                                                      |
| **Endpoints**    | `GET /api/docs`                                             |
| **UI**           | Scalar interactive docs (React component)                   |
| **Users**        | Developers, QA engineers                                    |
| **Dependencies** | `@scalar/api-reference-react`, FEAT-027                     |
| **Evidence**     | `app/api/docs/page.tsx`                                     |

**Capabilities:**
- [x] Render live interactive API reference (try-it-out)
- [x] No auth required to access docs UI

---

#### Feature: Health Check

| Aspect           | Value                                                       |
|------------------|-------------------------------------------------------------|
| **ID**           | FEAT-029                                                    |
| **Status**       | Stable                                                      |
| **Endpoints**    | `GET /api/v1/health`                                        |
| **UI**           | None                                                        |
| **Users**        | Monitoring tools, CI, uptime checks                         |
| **Dependencies** | None (static response)                                      |
| **Evidence**     | `app/api/v1/health/route.ts`                                |

**Capabilities:**
- [x] Return `{ ok: true, service, environment, timestamp }`
- [x] Public (no auth required)

---

### Domain: Observability & Platform

#### Feature: Activity Log (Audit Trail)

| Aspect           | Value                                                       |
|------------------|-------------------------------------------------------------|
| **ID**           | FEAT-030                                                    |
| **Status**       | Stable (DB infrastructure); no read API in Phase 1          |
| **Endpoints**    | None (writes happen internally on mutations)                |
| **UI**           | None in Phase 1                                             |
| **Users**        | Internal (audit); workspace admins eventually               |
| **Dependencies** | `activity_log` table                                        |
| **Evidence**     | `supabase/migrations/0009_cross_cutting.sql`                |

**Capabilities:**
- [x] Log `entity_type`, `entity_id`, `action` (create/update/delete), `payload` (JSONB diff), `actor_user_id`
- [ ] Read API for activity history (not in Phase 1)

---

#### Feature: Idempotency Key Support

| Aspect           | Value                                                       |
|------------------|-------------------------------------------------------------|
| **ID**           | FEAT-031                                                    |
| **Status**       | Stable (infrastructure ready; client adoption varies)       |
| **Endpoints**    | POST endpoints accept `Idempotency-Key` header              |
| **UI**           | None                                                        |
| **Users**        | API clients wanting safe retries                            |
| **Dependencies** | `idempotency_keys` table (24h TTL, no cleanup job yet)      |
| **Evidence**     | `supabase/migrations/0009_cross_cutting.sql`                |

**Capabilities:**
- [x] UNIQUE on `(user_id, endpoint, key)` — deduplicates replayed requests
- [x] Cache `response_snapshot` (JSONB) + `response_status` for replay
- [ ] Background TTL cleanup job (gap — rows accumulate indefinitely)

---

#### Feature: Feature Flags

| Aspect           | Value                                                       |
|------------------|-------------------------------------------------------------|
| **ID**           | FEAT-032                                                    |
| **Status**       | Stable (infrastructure); no management UI or API            |
| **Endpoints**    | None                                                        |
| **UI**           | None in Phase 1                                             |
| **Users**        | Internal (DB-managed today)                                 |
| **Dependencies** | `feature_flags` table                                       |
| **Evidence**     | `supabase/migrations/0009_cross_cutting.sql`                |

**Capabilities:**
- [x] Global scope (`workspace_id IS NULL`) and workspace scope
- [x] `key`, `enabled` (bool), `payload` (JSONB for rich config)
- [ ] Read/write API for feature flags (not in Phase 1)
- [ ] UI for flag management (not in Phase 1)

---

### Domain: Planned / WIP

#### Feature: ATC Run Execution

| Aspect           | Value                                                       |
|------------------|-------------------------------------------------------------|
| **ID**           | FEAT-033                                                    |
| **Status**       | Planned                                                     |
| **Evidence**     | `atcs.status` field exists with `running|pass|fail|blocked|skipped`; `run:execute` PAT scope defined but no endpoint found |

**Gap**: `run:execute` is a named scope with no corresponding endpoint yet. Run recording and result ingestion are implied by the data model but absent from Phase 1 code.

---

#### Feature: Transactional Email (Resend)

| Aspect           | Value                                                       |
|------------------|-------------------------------------------------------------|
| **ID**           | FEAT-034                                                    |
| **Status**       | Planned (package not yet installed)                         |
| **Evidence**     | `business-data-map.md` integration map; `N8N_API_URL` and Resend references in env; UI comment: "MVP does not send email — link copied to clipboard" |

**Gap**: Invite email delivery is manual (clipboard copy). Resend (`resend` package) not in `package.json` — integration is entirely future work.

---

#### Feature: Workflow Automation (n8n)

| Aspect           | Value                                                       |
|------------------|-------------------------------------------------------------|
| **ID**           | FEAT-035                                                    |
| **Status**       | Planned                                                     |
| **Evidence**     | `N8N_API_URL` environment variable referenced; `business-data-map.md` integration map |

**Gap**: No n8n client code found. Infrastructure variable exists but feature not implemented.

---

#### Feature: Project & Module CRUD (API)

| Aspect           | Value                                                       |
|------------------|-------------------------------------------------------------|
| **ID**           | FEAT-036                                                    |
| **Status**       | Planned / Discovery Gap                                     |
| **Evidence**     | `projects` and `modules` tables exist with full schema; UI pages for listing exist; no API route files under `app/api/v1/projects/` or `app/api/v1/modules/` |

**Gap**: Projects and modules are read from DB directly in server components. No REST API endpoints for create/update/delete. This limits headless agent and CI/CD integration.

---

## 3. CRUD Matrix

| Entity                  | Create          | Read   | Update              | Delete          | Evidence                              |
|-------------------------|-----------------|--------|---------------------|-----------------|---------------------------------------|
| Workspace               | ✅ POST         | ✅ GET | ⚠️ Name only (PATCH) | ❌              | `api/v1/workspaces`                   |
| Workspace Member        | ✅ (via invite) | ✅     | ⚠️ Suspend only (admin action, no API found) | ❌ | `workspace_members` |
| Workspace Invite        | ✅ POST         | ✅ GET | ⚠️ Rotate (POST)    | ✅ Soft (DELETE) | `api/v1/workspaces/{id}/invites`      |
| PAT                     | ✅ POST         | ✅ GET | ❌                  | ✅ Soft (DELETE) | `api/v1/tokens`                       |
| Project                 | ❌ No API       | ✅ (server) | ❌            | ❌              | `app/(app)/projects/`                 |
| Module                  | ❌ No API       | ✅ (server) | ❌            | ❌              | `modules` table, `lib/tree.ts`        |
| User Story              | ❌ No API       | ✅ (server) | ❌            | ❌              | `user_stories` table                  |
| Acceptance Criterion    | ❌ No API       | ✅ (server) | ❌            | ❌              | `acceptance_criteria` table           |
| ATC                     | ✅ (RPC)        | ✅     | ✅ (RPC save)       | ❌ No API       | `api/(app)/projects/[slug]/atcs/`     |
| ATC Steps               | ✅ (via RPC)    | ✅     | ✅ (via RPC, full replace) | ✅ (via RPC) | `atc_steps`                     |
| ATC Assertions          | ✅ (via RPC)    | ✅     | ✅ (via RPC, full replace) | ✅ (via RPC) | `atc_assertions`                |
| ATC–AC Binding          | ✅ (via RPC)    | ✅     | ✅ (via RPC, full replace) | ✅ (via RPC) | `atc_acceptance_criteria`       |
| Activity Log            | ✅ (auto)       | ❌ No API | ❌              | ❌              | `activity_log`                        |
| Idempotency Key         | ✅ (auto)       | ❌      | ❌                  | ❌ (TTL-based)  | `idempotency_keys`                    |
| Feature Flag            | ❌ (DB only)    | ❌ No API | ❌ No API       | ❌              | `feature_flags`                       |

**Legend**: ✅ Implemented · ⚠️ Partial/conditional · ❌ Not available / no endpoint

---

## 4. API Endpoint Inventory

### Auth

| Method | Endpoint                        | Purpose                                | Auth         |
|--------|---------------------------------|----------------------------------------|--------------|
| POST   | `/api/v1/auth/magic-link`       | Send OTP magic-link email              | Public       |
| POST   | `/api/v1/auth/signin`           | Headless sign-in + auto-mint PAT       | Public       |
| POST   | `/api/v1/auth/signup`           | Headless sign-up + auto-mint PAT (QA) | Public       |
| GET    | `/auth/callback`                | OTP code exchange → set session cookie | Public       |

### Identity

| Method | Endpoint                        | Purpose                                | Auth         |
|--------|---------------------------------|----------------------------------------|--------------|
| GET    | `/api/v1/me`                    | Introspect current principal           | Cookie/Bearer |
| POST   | `/api/v1/me/active-workspace`   | Set active workspace cookie            | Cookie/Bearer |

### Workspaces

| Method | Endpoint                        | Purpose                                | Auth         |
|--------|---------------------------------|----------------------------------------|--------------|
| POST   | `/api/v1/workspaces`            | Create workspace (onboarding)          | Cookie/Bearer |
| GET    | `/api/v1/workspaces`            | List caller's workspaces               | Cookie/Bearer |
| GET    | `/api/v1/workspaces/{id}`       | Get single workspace                   | Cookie/Bearer |
| PATCH  | `/api/v1/workspaces/{id}`       | Update workspace name (owner only)     | Cookie/Bearer |

### Invites

| Method | Endpoint                                              | Purpose                                | Auth         |
|--------|-------------------------------------------------------|----------------------------------------|--------------|
| POST   | `/api/v1/workspaces/{id}/invites`                     | Issue workspace invite                 | Cookie/Bearer (admin+) |
| GET    | `/api/v1/workspaces/{id}/invites`                     | List workspace invites                 | Cookie/Bearer (admin+) |
| POST   | `/api/v1/workspaces/{id}/invites/{inviteId}`          | Rotate invite token                    | Cookie/Bearer (admin+) |
| DELETE | `/api/v1/workspaces/{id}/invites/{inviteId}`          | Revoke invite                          | Cookie/Bearer (admin+) |
| POST   | `/api/v1/invites/accept`                              | Redeem invite token                    | Cookie/Bearer |

### Tokens (PATs)

| Method | Endpoint                        | Purpose                                | Auth         |
|--------|---------------------------------|----------------------------------------|--------------|
| POST   | `/api/v1/tokens`                | Mint PAT                               | Cookie only  |
| GET    | `/api/v1/tokens`                | List caller's PATs                     | Cookie only  |
| DELETE | `/api/v1/tokens/{id}`           | Revoke PAT (soft-delete)               | Cookie only  |

### Developer Surface

| Method | Endpoint                        | Purpose                                | Auth         |
|--------|---------------------------------|----------------------------------------|--------------|
| GET    | `/api/v1/health`                | Health check                           | Public       |
| GET    | `/api/openapi`                  | Live OpenAPI 3.0 JSON spec             | Public       |
| GET    | `/api/docs`                     | Interactive Scalar API docs UI         | Public       |
| GET    | `/api/v1`                       | API root info                          | Public       |

---

## 5. UI Component Inventory

### Pages

| Page                                   | Route                                    | Purpose                                         |
|----------------------------------------|------------------------------------------|-------------------------------------------------|
| Login                                  | `/login`                                 | Magic-link email submission form                |
| Onboarding                             | `/onboarding`                            | First-time workspace creation                   |
| Project List                           | `/projects`                              | Browse all projects in active workspace         |
| Project Detail / ATC Table             | `/projects/[projectSlug]`               | ATC list with module tree sidebar               |
| ATC Editor                             | `/projects/[projectSlug]/atcs/[atcId]`  | Full ATC authoring (steps, assertions, AC bind) |
| Team Members                           | `/workspaces/[id]/members`              | Invite management + active member list          |
| Invite Acceptance                      | `/invites/accept`                        | Token redemption for invitees                   |
| API Docs                               | `/api/docs`                              | Scalar interactive API reference                |
| Design Tokens                          | `/design-tokens`                         | Internal design system reference                |
| QA Reference                           | `/qa`                                    | QA onboarding documentation page               |

### Key Components

| Component         | Location                                | Role                                                        |
|-------------------|-----------------------------------------|-------------------------------------------------------------|
| `MagicLinkForm`   | `app/(auth)/login/magic-link-form.tsx`  | Email input + submit for OTP flow                           |
| `OnboardingForm`  | `app/(app)/onboarding/onboarding-form.tsx` | Workspace slug/name creation form                        |
| `AtcEditor`       | `components/atcs/AtcEditor.tsx`         | Main ATC authoring surface (title, layer, tags, editors, anchoring) |
| `AtcTable`        | `components/atcs/AtcTable.tsx`          | Project ATC list with status badges                         |
| `AnchoringPanel`  | `components/atcs/AnchoringPanel.tsx`    | Story + AC picker for ATC binding                           |
| `StepEditor`      | `components/atcs/StepEditor.tsx`        | Markdown editor for ATC steps (Monaco)                      |
| `MembersClient`   | `app/(app)/workspaces/[id]/members/members-client.tsx` | Invite form + member/invite list           |
| `AcceptClient`    | `app/invites/accept/accept-client.tsx`  | Token redemption UI for invitees                            |

---

## 6. Third-Party Integrations

| Service           | Purpose                                           | Package                         | Status        | Features using it          |
|-------------------|---------------------------------------------------|---------------------------------|---------------|----------------------------|
| **Supabase Auth** | Magic-link OTP, session management, `auth.users`  | `@supabase/ssr`, `@supabase/supabase-js` | Active | FEAT-001, 002, 003, 004 |
| **Supabase PostgreSQL** | All application data, RLS enforcement      | `@supabase/supabase-js`         | Active        | All features               |
| **Vercel**        | Hosting, edge functions, CI/CD deploy target      | (deploy target, no package)     | Active        | All features               |
| **Jira (BK project)** | External issue ID linkage on User Stories   | `acli` (CLI tool, not package)  | Active (read) | FEAT-019, 020 (story.external_id) |
| **Resend**        | Transactional email for invite delivery           | Not in `package.json` yet       | Planned       | FEAT-014, 034              |
| **n8n**           | Workflow automation                               | Not in `package.json` yet       | Planned       | FEAT-035                   |
| **Monaco Editor** | In-browser code editor for steps/assertions       | `@monaco-editor/react`          | Active        | FEAT-023, 025              |
| **Scalar**        | Interactive API documentation renderer            | `@scalar/api-reference-react`   | Active        | FEAT-028                   |
| **Sonner**        | Toast notifications                               | `sonner`                        | Active        | FEAT-014, 016, 017, 018    |
| **TanStack Table**| Sortable/filterable ATC table                     | `@tanstack/react-table`         | Active        | FEAT-020                   |

---

## 7. Feature Flags and WIP

### Feature Flags (DB-managed)

| Flag scope | Key pattern     | Description                                 | Default | Environment  |
|------------|-----------------|---------------------------------------------|---------|--------------|
| global     | (any)           | System-wide toggles (no entries seeded yet) | —       | All          |
| workspace  | (any)           | Per-workspace toggles                       | —       | All          |

No flags have been seeded in Phase 1. Infrastructure exists for runtime feature gating.

### Planned Features (TODOs, stubs, implied by data model)

| Planned Feature              | Evidence                                                     | Estimated Status |
|------------------------------|--------------------------------------------------------------|------------------|
| ATC Run Execution API        | `run:execute` PAT scope defined; `atcs.status` enum; no endpoint | Not started  |
| Invite email via Resend      | UI comment "MVP does not send email"; Resend not in package.json | Planned      |
| n8n workflow automation      | `N8N_API_URL` env var; integration map in docs               | Planned          |
| Project CRUD API             | UI lists projects; no `POST /api/v1/projects`                | Planned          |
| Module CRUD API              | Module tree exists; no `POST /api/v1/modules`                | Planned          |
| Workspace slug rotation      | Comment "slug rotation post-MVP" in `route.openapi.ts`       | Post-MVP         |
| Activity log read API        | Table exists; no `GET /api/v1/activity`                      | Not started      |
| Feature flag management API  | Table exists; no CRUD endpoints                              | Not started      |
| Idempotency key TTL cleanup  | `expires_at` column exists; no background job                | Not started      |
| PAT management UI            | API complete; no UI pages found                              | Not started      |
| ATC full-text search UI      | GIN index + tsv ready; no search box in Phase 1              | Not started      |
| Member suspension / removal  | `status=suspended` in schema; no API                         | Planned          |
| Workspace deletion           | No endpoint                                                  | Not planned (MVP)|
| ATC deletion                 | No endpoint                                                  | Not planned (MVP)|

---

## 8. QA Relevance

### Feature Test Coverage Matrix

| Feature ID | Feature                         | Unit | Integration | E2E | Coverage Status        |
|------------|---------------------------------|------|-------------|-----|------------------------|
| FEAT-001   | Magic-Link Login                | —    | ⚠️          | ⚠️  | Partial — no full E2E  |
| FEAT-002   | Headless Sign-in + PAT          | —    | ✅          | —   | API covered (auth spec)|
| FEAT-003   | Headless Sign-up + PAT          | —    | ⚠️          | —   | Needs QA coverage      |
| FEAT-004   | OAuth Callback                  | —    | ⚠️          | —   | Edge cases missing     |
| FEAT-005   | PAT Issuance                    | —    | ✅          | —   | API covered            |
| FEAT-006   | PAT Listing                     | —    | ⚠️          | —   | Needs coverage         |
| FEAT-007   | PAT Revocation                  | —    | ⚠️          | —   | Needs coverage         |
| FEAT-008   | Principal Introspection         | —    | ⚠️          | —   | Needs coverage         |
| FEAT-009   | Active Workspace Selection      | —    | ⚠️          | —   | Needs coverage         |
| FEAT-010   | Workspace Creation              | —    | ✅          | ⚠️  | API covered; E2E partial |
| FEAT-011   | Workspace Listing               | —    | ⚠️          | —   | Needs coverage         |
| FEAT-012   | Workspace Detail                | —    | ⚠️          | —   | Needs coverage         |
| FEAT-013   | Workspace Name Update           | —    | ❌          | —   | Not covered            |
| FEAT-014   | Invite Issuance                 | —    | ⚠️          | ⚠️  | Partial                |
| FEAT-015   | Invite Listing                  | —    | ⚠️          | —   | Needs coverage         |
| FEAT-016   | Invite Acceptance               | —    | ⚠️          | ⚠️  | Critical — partial     |
| FEAT-017   | Invite Revocation               | —    | ❌          | —   | Not covered            |
| FEAT-018   | Invite Rotation                 | —    | ❌          | —   | Not covered            |
| FEAT-019   | Project List View               | —    | —           | ⚠️  | UI not tested          |
| FEAT-020   | Project Detail / ATC Table      | —    | —           | ⚠️  | UI not tested          |
| FEAT-021   | Module Tree Navigation          | —    | —           | ⚠️  | UI not tested          |
| FEAT-022   | ATC Full-Text Search            | —    | ❌          | —   | DB ready, no test      |
| FEAT-023   | ATC Editor                      | —    | ⚠️          | ⚠️  | Core feature — needs E2E |
| FEAT-024   | ATC AC Anchoring                | —    | ⚠️          | ⚠️  | Critical — needs E2E   |
| FEAT-025   | ATC Step Authoring              | ⚠️   | —           | ⚠️  | Parser tested?         |
| FEAT-026   | ATC Status Lifecycle            | —    | ❌          | —   | No transition tests    |
| FEAT-027   | OpenAPI Spec                    | —    | ⚠️          | —   | Structure tested?      |
| FEAT-028   | Interactive API Docs            | —    | —           | ⚠️  | Render test needed     |
| FEAT-029   | Health Check                    | —    | ✅          | —   | Covered                |
| FEAT-030   | Activity Log                    | —    | ❌          | —   | Not testable via API   |
| FEAT-031   | Idempotency Keys                | —    | ❌          | —   | Not covered            |
| FEAT-032   | Feature Flags                   | —    | ❌          | —   | No test surface        |

**Legend**: ✅ Covered · ⚠️ Partial / needs attention · ❌ Not covered · — Not applicable

### High-Risk Features (Prioritize for Testing)

| Feature  | Risk Level | Reason                                                                                    |
|----------|------------|-------------------------------------------------------------------------------------------|
| FEAT-016 | **CRITICAL** | Invite acceptance — multi-step validation chain; email mismatch + expiry edge cases are security-sensitive |
| FEAT-023 | **CRITICAL** | ATC save (RPC) — delete-then-reinsert pattern; bad positions or partial writes corrupt test data |
| FEAT-002 | **HIGH**   | Headless auth + PAT minting — single request mints permanent credential; scope enforcement is a security boundary |
| FEAT-001 | **HIGH**   | Magic-link auth — open-redirect guard; OTP rate limiting (4/hour); session cookie handling |
| FEAT-010 | **HIGH**   | Workspace creation — atomic RPC with reserved slug block; 409 collision handling          |
| FEAT-024 | **HIGH**   | AC anchoring — M:N replace-all pattern; empty AC binding breaks ATC integrity             |
| FEAT-007 | **HIGH**   | PAT revocation — soft-delete must be reflected immediately by bearer middleware            |
| FEAT-021 | **MEDIUM** | Module tree max depth (6 levels CHECK constraint) — off-by-one failures corrupt hierarchy |
| FEAT-014 | **MEDIUM** | Invite issuance without email — link must be manually shared; raw token single-show        |
| FEAT-031 | **MEDIUM** | Idempotency — replayed POST must return cached response, not re-execute mutation          |

---

## 9. Discovery Gaps

These features or behaviors could not be fully verified from Phase 1 source code:

| Gap | Impact | Recommended Action |
|-----|--------|--------------------|
| **Project CRUD API missing** — no `POST /api/v1/projects` endpoint; projects exist in DB but no creation flow found in routes | Blocks headless project setup for CI/QA | Confirm with dev team — may be seeded via migration or RPC |
| **Module CRUD API missing** — no API for create/rename/delete module; tree is read-only from API perspective | Limits test data setup flexibility | Same as above — RPC or direct DB seed likely |
| **ATC run execution** — `run:execute` PAT scope and status enum exist with no endpoint | Cannot test end-to-end test result recording | Track as FEAT-033; verify roadmap date |
| **Member suspension/removal UI** — `workspace_members.status=suspended` exists; no API action found | Incomplete team management surface | Confirm if out of scope for Phase 1 |
| **ATC deletion** — no delete endpoint or button visible in UI | ATCs can only be abandoned, not removed | Intentional? Verify with product team |
| **Idempotency client header** — infrastructure in DB but unclear which endpoints enforce/check it | Cannot test replay behavior without knowing which routes opt in | Read `lib/api/` middleware to confirm |
| **Activity log write triggers** — table exists; unclear which mutations write to it automatically vs. manually | Cannot verify audit coverage | Grep for `activity_log` inserts in route handlers |
| **`bk_active_ws` cookie behavior on logout** — does logout clear this cookie? | Could leak workspace context to next session | Trace `auth/callback` + logout route |

---

*Cross-reference with `business-data-map.md` for entity state machines and integration flows.*  
*Update this file after each sprint that adds routes, pages, or DB migrations.*
