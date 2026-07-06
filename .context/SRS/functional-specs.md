# SRS — Functional Specifications

> **Discovery date**: 2026-05-31  
> **Method**: Reverse-engineered from API route handlers, page components, DB migrations, and RLS policies  
> **Convention**: FR-N = Functional Requirement, BR-N = Business Rule, ERR-N = Error Behavior

---

## FR-1 — User Authentication

### FR-1.1 — Magic-Link Login (Browser)
Users authenticate via passwordless OTP email.

**Preconditions**: User has an email address; Supabase email provider configured.

**Flow**:
1. User submits email to `POST /api/v1/auth/magic-link`
2. System calls `supabase.auth.signInWithOtp({ email })`
3. Supabase sends OTP link to user's email
4. User clicks link → `GET /auth/callback?code=...&next=...`
5. System calls `supabase.auth.exchangeCodeForSession(code)`
6. Session cookie set; user redirected to `next` (default: `/projects`)

**BR-1.1**: `next` parameter must be root-relative (starts with `/`, does not start with `//`). Invalid `next` defaults to `/projects`.  
**ERR-1.1**: If code exchange fails → redirect to `/login?error=otp_exchange_failed`.

Source: `app/api/v1/auth/magic-link/route.ts`, `app/auth/callback/route.ts`

---

### FR-1.2 — Headless Authentication (CLI / Agent)
Non-browser clients authenticate with email + password and receive a PAT in the same request.

**Preconditions**: User has a password-authenticated Supabase account.

**Flow**:
1. Client sends `POST /api/v1/auth/signin` with `{ email, password, pat_name?, pat_scopes?, workspace_id?, expires_in_days? }`
2. System calls `supabase.auth.signInWithPassword()`
3. System mints a PAT: stores in `access_tokens` + SHA-256 hash in `access_token_secrets`
4. Returns `{ session: {...}, pat: { token: "bk_pat_<prefix>.<secret>", scopes, expires_at } }`

**BR-1.2**: Raw PAT token returned exactly once. Not recoverable after initial response — client must cache it.  
**ERR-1.2**: Invalid credentials → 401. Email already exists (signup) → 409.

Source: `app/api/v1/auth/signin/route.ts`, `lib/api/pat.ts`

---

## FR-2 — Personal Access Tokens (PATs)

### FR-2.1 — Token Creation
**Endpoint**: `POST /api/v1/tokens`  
**Auth**: Cookie session only (a PAT cannot create another PAT).

**Inputs**: `{ name?, workspace_id?, scopes: string[], expires_in_days? }`  
**Scopes allowed**: `atc:read | atc:write | run:execute | workspace:admin`  
**Token format**: `bk_pat_<12-char-prefix>.<32-byte-secret-base64url>`

**BR-2.1**: `scopes` array must be non-empty. Values must be a subset of the 4 allowed scopes.

### FR-2.2 — Token Revocation
**Endpoint**: `DELETE /api/v1/tokens/{id}`  
**Auth**: Owner only (user_id = auth.uid()).  
**Action**: Soft-delete — stamps `revoked_at`, preserves record for audit.

### FR-2.3 — Token Listing
**Endpoint**: `GET /api/v1/tokens`  
Returns metadata only — raw secret never returned after creation.

Source: `supabase/migrations/0008_access_tokens.sql`, `lib/api/pat.ts`

---

## FR-3 — Workspace Management

### FR-3.1 — Workspace Creation
**Endpoint**: `POST /api/v1/workspaces` (also via onboarding form → `bunkai_bootstrap_workspace` RPC)

**Inputs**: `{ slug, name }`  
**BR-3.1a**: Slug must match `^[a-z0-9][a-z0-9-]{1,38}[a-z0-9]$` (3–40 chars, alphanumeric + hyphens, no leading/trailing hyphens).  
**BR-3.1b**: Reserved slugs rejected: `api`, `auth`, `admin`, `login`, and other system paths.  
**BR-3.1c**: Workspace creation is atomic — workspace + owner member created in one `bunkai_bootstrap_workspace` RPC transaction.  
**ERR-3.1**: Slug already taken → 409 Conflict, message includes the slug.

### FR-3.2 — Workspace Listing
**Endpoint**: `GET /api/v1/workspaces`  
Returns workspaces where `workspace_members.user_id = auth.uid() AND status = 'active'`.

### FR-3.3 — Active Workspace Selection
**Endpoint**: `POST /api/v1/me/active-workspace`  
Sets `bk_active_ws` httpOnly cookie. Does not modify JWT.

Source: `supabase/migrations/0001_tenancy.sql`, `supabase/migrations/0006_bootstrap_workspace.sql`

---

## FR-4 — Team Invitations

### FR-4.1 — Invite Creation
**Endpoint**: `POST /api/v1/workspaces/{id}/invites`  
**Auth**: admin or owner role required.  
**Inputs**: `{ email, role }` where `role ∈ { viewer, member, admin }`

**Flow**:
1. Generate raw token (crypto-random)
2. Store SHA-256 hash in `workspace_invite_secrets`
3. Set `expires_at = now() + 7 days`
4. Return `{ invite_id, accept_url, raw_token }` — raw token returned once only

**BR-4.1**: Invite expires in 7 days. `MVP: email not sent automatically — caller copies accept_url`.

### FR-4.2 — Invite Acceptance
**Endpoint**: `POST /api/v1/invites/accept`  
**Auth**: Authenticated user required.  
**Inputs**: `{ token }`

**Validations**:
1. Hash token → lookup `workspace_invite_secrets.token_hash`
2. `invite.email` must match `auth.uid()` email (case-insensitive)
3. `invite.revoked_at IS NULL`
4. `invite.accepted_at IS NULL`
5. `invite.expires_at > now()`

**Action**: Upsert `workspace_members (workspace_id, user_id, role, status='active')`.  
**Stamps**: `invite.accepted_at`, `invite.accepted_by_user_id`.

**ERR-4.2**: Token not found → 404. Email mismatch → 403. Expired → 410. Already accepted → 409.

### FR-4.3 — Invite Revocation
**Endpoint**: `PATCH /api/v1/workspaces/{id}/invites/{inviteId}`  
**Auth**: admin or owner role.  
**Action**: Stamp `invite.revoked_at = now()`. Token immediately invalid.

### FR-4.4 — Invite Rotation (Resend)
**Endpoint**: `POST /api/v1/workspaces/{id}/invites/{inviteId}` (same as revoke but re-generates token)  
**Action**: Revoke old token → generate new token → return new `accept_url`.

Source: `supabase/migrations/0010_workspace_invites.sql`, `supabase/migrations/0011_split_token_secrets.sql`

---

## FR-5 — Project Management

### FR-5.1 — Project Scoping
Projects are scoped to a workspace. `slug` is unique within the workspace.  
**BR-5.1**: UNIQUE constraint on `(workspace_id, slug)`.

Source: `supabase/migrations/0002_projects_modules.sql`

---

## FR-6 — Module Tree

### FR-6.1 — Hierarchical Modules
Modules form a tree within a project using a materialized path and a self-referential FK.

**BR-6.1a**: Max depth 6 (`array_length(string_to_array(path, '/'), 1) BETWEEN 1 AND 6`).  
**BR-6.1b**: `path` is unique within a project (`UNIQUE (project_id, path)`).  
**BR-6.1c**: Deleting a parent module cascades to all children, user stories, ATCs.

### FR-6.2 — Module Tree Rendering
`lib/tree.ts` builds `ModuleTreeNode[]` from flat module list for sidebar display.  
**BR-6.2**: Nodes sorted by `position` (integer) within siblings.

Source: `supabase/migrations/0002_projects_modules.sql`, `lib/tree.ts`

---

## FR-7 — User Stories & Acceptance Criteria

### FR-7.1 — External Linkage
UserStories optionally link to external issue tracker items.  
**BR-7.1**: `external_id` (e.g. `BK-42`) and `external_url` stored but not validated — any string accepted.

### FR-7.2 — AC Ordering
AcceptanceCriteria are ordered by `position` within a UserStory.  
**BR-7.2**: UNIQUE constraint on `(user_story_id, position)`.

Source: `supabase/migrations/0003_authoring.sql`

---

## FR-8 — ATC (Acceptance Test Case) Management

### FR-8.1 — ATC Creation & Editing
ATCs are created/edited via the `bunkai_save_atc` RPC — atomic update of header + steps + assertions + AC bindings.

**Inputs**: `(atc_id, title, layer, tags[], user_story_id, steps jsonb, assertions jsonb, ac_ids uuid[])`

**BR-8.1a**: ATC must belong to a UserStory (`user_story_id` FK is RESTRICT — deleting a story blocks if ATCs exist).  
**BR-8.1b**: `layer ∈ { UI, API, Unit }`.  
**BR-8.1c**: `slug` is unique within the project.  
**BR-8.1d**: `version` is incremented on every save (optimistic locking, Phase E).

**Save behavior**: Steps and assertions are deleted and re-inserted (not diffed). AC bindings are also replaced atomically.

### FR-8.2 — ATC Status Lifecycle
**BR-8.2a**: Status values: `unrun | running | pass | fail | blocked | skipped`.  
**BR-8.2b**: Default status is `unrun`.  
**BR-8.2c**: Status transitions are not currently enforced by DB constraints — enforced at application layer.

### FR-8.3 — Full-Text Search
ATCs support full-text search on title + tags via a `tsvector` column.  
**BR-8.3**: `atcs_refresh_tsv` trigger refreshes `tsv` on INSERT or UPDATE of `title` or `tags`.  
**BR-8.3b**: GIN index on `tsv` for fast search.

### FR-8.4 — AC Binding (Anchoring Moat)
ATCs can be bound to multiple AcceptanceCriteria via the `atc_acceptance_criteria` join table.  
**BR-8.4**: Minimum binding of 0 ACs allowed in DB (application layer recommends at least 1). The ATC editor provides an AnchoringPanel picker.

Source: `supabase/migrations/0004_atcs.sql`, `supabase/migrations/0007_save_atc.sql`

---

## FR-9 — Observability & Idempotency

### FR-9.1 — Activity Log
All significant entity mutations write to `activity_log`.  
**Fields**: `entity_type`, `entity_id`, `action` (create/update/delete), `payload` (JSONB diff), `actor_user_id`.

### FR-9.2 — Idempotency Keys
POST endpoints support client-supplied idempotency keys (24h TTL).  
**BR-9.2**: UNIQUE on `(user_id, endpoint, key)`. Replays return cached `response_snapshot`.

### FR-9.3 — Feature Flags
`feature_flags` table supports global and workspace-scoped feature toggles.  
**BR-9.3**: `scope = 'global' → workspace_id IS NULL`; `scope = 'workspace' → workspace_id NOT NULL`.

Source: `supabase/migrations/0009_cross_cutting.sql`

---

## FR-10 — API Surface

### FR-10.1 — OpenAPI Spec Generation
`GET /api/openapi` returns a valid OpenAPI 3.0 JSON spec generated at runtime from Zod schemas via `@asteasolutions/zod-to-openapi`.

### FR-10.2 — Interactive Docs
`GET /api/docs` renders the Scalar interactive documentation UI (React component, no auth required).

### FR-10.3 — Health Check
`GET /api/v1/health` returns `{ ok: true, service, environment, timestamp }`.

Source: `app/api/v1/health/route.ts`, `app/api/openapi/route.ts`, `app/api/docs/page.tsx`

---

## Discovery Gaps

- [ ] **Project CRUD**: No `POST /api/v1/projects` endpoint found. How are projects created? May be via RPC or missing from current Phase 1.
- [ ] **Module CRUD**: No module create/rename/delete endpoints found in API routes. Module management may be client-side only or deferred.
- [ ] **ATC creation form**: "New ATC" button exists in UI but the creation flow (form/page) was not traced — may reuse the editor with an empty ATC pre-seeded.
- [ ] **Run execution endpoints**: `atcs.status` exists and IQL references "Run" as a lifecycle stage but no run/result endpoints are in Phase 1.
- [ ] **Bulk operations**: No batch create/update/delete endpoints found.
