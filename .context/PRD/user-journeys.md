# PRD — User Journeys

> **Discovery date**: 2026-05-31  
> **Method**: Reverse-engineered from page components, middleware, API routes, and auth callback handlers  
> **Source files**: `middleware.ts`, `app/(app)/*/page.tsx`, `app/(auth)/login/`, `app/auth/callback/route.ts`, `app/invites/accept/`

---

## Journey 1 — First-Time Onboarding (Magic-Link)

**Actor**: New QA Lead (Sarah)  
**Entry point**: Direct URL or referral link  
**Goal**: Create a workspace and reach the projects view

```
[1] Visit https://upexbunkai.vercel.app
        ↓ (middleware: no session → redirect)
[2] GET /login
        ↓ Left panel: brand story + IQL/ATC/KATA explanation
        ↓ Right panel: email input
[3] Enter work email → click "Send magic link"
        → POST /api/v1/auth/magic-link { email, next: "/projects" }
        ↓ API sends OTP email via Supabase
[4] Open email → click magic link
        → GET /auth/callback?code=<otp>&next=/projects
        ↓ exchangeCodeForSession(code) → Supabase sets cookie
        → Redirect to /projects
[5] GET /projects
        ↓ Server: no workspace membership found
        → Redirect to /onboarding
[6] GET /onboarding
        ↓ OnboardingForm renders (2 fields: name, slug)
[7] Type "Acme QA" → slug auto-fills "acme-qa"
        → Click "Create workspace"
        → POST /api/v1/workspaces { name: "Acme QA", slug: "acme-qa" }
        ↓ bunkai_bootstrap_workspace RPC: workspace + owner member created atomically
[8] Toast: "Workspace created" → redirect /projects
```

**Key touch points**: email form validation, magic-link delivery, OTP exchange, onboarding form slug auto-fill, 409 conflict on duplicate slug

---

## Journey 2 — Daily ATC Management (Main Workflow)

**Actor**: Test Automation Engineer (James)  
**Entry point**: Already authenticated, workspace active  
**Goal**: Find, edit, and save an ATC

```
[1] GET /projects/acme-qa
        ↓ Server loads: project, workspace, modules tree, all ATCs
        ↓ Sidebar: hierarchical module tree
        ↓ Main: AtcTable (all ATCs with status chips + module_path column)
[2] Scan ATC table → find target ATC
        → Option A: Click row → GET /projects/acme-qa/atcs/[atcId]
        → Option B: Cmd+K → type ATC name → select from palette
        → Option C: Sidebar tree → navigate to module → story → click ATC
[3] GET /projects/acme-qa/atcs/[atcId]
        ↓ Server loads: ATC + steps + assertions + AC bindings + all stories/ACs for picker
        ↓ AtcEditor renders with initialSteps, initialAssertions, initialAcIds
[4] Edit steps:
        → Add step: "When I submit the login form with valid credentials"
        → Set input_data: '{ "email": "qa@acme.com", "password": "..." }'
        → Set expected: "Dashboard renders, user menu shows email"
[5] Edit assertions:
        → Add assertion: "Response status is 200"
        → Add assertion: "Cookie bk_session is set"
[6] Bind to Acceptance Criterion:
        → Open AnchoringPanel → browse stories → select story → select AC
        → AC bound (shown as chip/tag)
[7] Click "Save"
        → saveAtcAction (server action) → bunkai_save_atc RPC
        ↓ Steps/assertions deleted + re-inserted atomically; AC bindings updated
        ↓ atcs.updated_at auto-updated by trigger
[8] Success feedback → continue editing or navigate back
```

**Key touch points**: command palette navigation, step/assertion ordering, AC picker (story grouping), atomic save RPC, optimistic version lock

---

## Journey 3 — Team Member Invitation

**Actor**: QA Lead (Sarah, admin/owner)  
**Entry point**: /workspaces/[id]/members  
**Goal**: Invite James to the workspace as `member`

```
[1] Navigate to workspace settings → GET /workspaces/[id]/members
        ↓ MembersClient loads: members list + pending invites list
[2] Enter email: "james@acme.com"
        → Select role: "member" (default)
        → Click "Generate invite link"
        → POST /api/v1/workspaces/[id]/invites { email, role }
        ↓ API: generates raw token, stores SHA-256 hash in workspace_invite_secrets
        ↓ Returns: { accept_url: "https://upexbunkai.vercel.app/invites/accept?token=..." }
[3] Link auto-copied to clipboard
        ↓ Toast: "Invite link for james@acme.com copied to clipboard."
        ↓ [MVP: no email sent — Sarah shares link manually]
[4] James appears in "Pending invites" section with:
        - email: james@acme.com
        - role: member
        - expires_at: now + 7 days
        - Actions: [Resend link] [Revoke]

Optional: Rotate link
        → Click "Resend link" → POST /api/v1/workspaces/[id]/invites/[inviteId]
        ↓ New token generated, old invalidated
        ↓ New link auto-copied to clipboard

Optional: Revoke
        → Click "Revoke" → DELETE /api/v1/workspaces/[id]/invites/[inviteId]
        ↓ invite.revoked_at stamped
        ↓ Toast: "Invite revoked."
```

**Key touch points**: role dropdown (viewer/member/admin), clipboard copy, pending invite display, 7-day expiry, rotate/revoke actions

---

## Journey 4 — Invite Acceptance

**Actor**: New team member (James)  
**Entry point**: Invite link received from Sarah  
**Goal**: Join the workspace and reach the projects view

```
[1] James clicks link:
        GET /invites/accept?token=<raw_token>&next=/projects
        ↓ Page server component reads token from query param
[2] AcceptClient renders:
        ↓ Checks auth state
        → Case A: Not authenticated
                → Shows "Sign in to accept this invite"
                → Button: "Sign in" → /login?next=/invites/accept?token=...
                → James signs in via magic-link (Journey 1, steps 3-4)
                → Returns to /invites/accept with token still in URL
        → Case B: Authenticated
                → Shows "Accept invite" button
[3] James clicks "Accept invite"
        → POST /api/v1/invites/accept { token: "<raw_token>" }
        ↓ API: hashes token → lookup invite_id in workspace_invite_secrets
        ↓ Validates: email matches auth.uid() email, not expired, not revoked, not accepted
        ↓ Upserts workspace_members (workspace_id, user_id, role='member', status='active')
        ↓ Stamps invite.accepted_at + accepted_by_user_id
[4] Redirect to /projects (or `next` param)
        ↓ James now sees Acme QA workspace projects
```

**Key touch points**: unauthenticated redirect chain (preserves token in `next` param), token hash validation, email ownership check, expiry check, idempotent upsert

---

## Journey 5 — Headless Agent Execution (Bearer PAT)

**Actor**: AI agent (Claude/OpenCode) or CI script  
**Entry point**: API call (no browser)  
**Goal**: Fetch ATCs and post execution results

```
[1] POST /api/v1/auth/signin
        { email, password, pat_name: "ci-runner", pat_scopes: ["atc:read","run:execute"] }
        ↓ Supabase: verifyPassword → sign in
        ↓ Mint PAT: store in access_tokens + access_token_secrets
        → Returns: { session: {...}, pat: { token: "bk_pat_<prefix>.<secret>", ... } }
[2] Cache PAT locally — used for all subsequent calls
[3] GET /api/v1/me
        Authorization: Bearer bk_pat_<prefix>.<secret>
        → Returns: { user_id, email, workspaces, active_workspace_id }
[4] GET /api/v1/workspaces
        → Resolve workspace_id for "acme-qa"
[5] Fetch ATCs for project (via project endpoints — not yet specified in migrations)
        → Filter by module/story for targeted execution
[6] Execute each ATC step → record result
[7] POST results (run execution endpoint — planned Phase 2)
        → Update atc.status to 'pass' / 'fail' / 'blocked'
[8] DELETE /api/v1/tokens/[id]  (optional cleanup)
        → Revoke PAT after run
```

**Key touch points**: headless sign-in + PAT mint in one call, Bearer token precedence over cookie, scope enforcement, idempotency keys for POST replay protection

---

## Route Map

```
/ → /login (redirect, no UI)

/login
  → POST /api/v1/auth/magic-link
  → /auth/callback?code=...&next=...
    → /projects (authenticated) or /login?error=... (failed)

/projects
  → /onboarding (no workspace membership)
  → /projects/[projectSlug] (has projects)

/projects/[projectSlug]
  → /projects/[projectSlug]/atcs/[atcId]

/onboarding
  → POST /api/v1/workspaces
  → /projects

/workspaces/[id]/members
  → POST /api/v1/workspaces/[id]/invites
  → PATCH /api/v1/workspaces/[id]/invites/[inviteId]
  → DELETE /api/v1/workspaces/[id]/invites/[inviteId]

/invites/accept?token=...
  → /login?next=/invites/accept?token=... (unauthenticated)
  → POST /api/v1/invites/accept
  → /projects

/qa          (public — QA testing guide)
/design-tokens (public — design system reference)
/api/docs    (public — Scalar interactive API docs)
/api/openapi (public — OpenAPI JSON spec)
```

---

## Discovery Gaps

- [ ] **Project creation journey**: No "Create project" page or form found in source. After workspace creation the user lands on `/projects` — but how they create the first project is not in Phase 1 source.
- [ ] **ATC creation form**: The "New ATC" button exists in the topbar with keyboard shortcut "N", but the creation form/page itself was not traced in the explore phase.
- [ ] **Module creation**: Module tree is displayed in sidebar but no "Add module" or "Rename module" page was found.
- [ ] **Password-based auth for agents**: `POST /api/v1/auth/signin` accepts email+password, but Supabase magic-link is the only browser auth method. Confirm password-based auth is agent-only.
