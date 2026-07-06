# Master Test Plan — Bunkai TMS

```
+----------------------------------------------------------+
|  Bunkai TMS — What to Test, and Why                      |
|  Generated: 2026-05-31  |  Source: full Phase 1 trilogy  |
|  "Decompose stories. Execute trust."                     |
+----------------------------------------------------------+
```

> This is a **test-strategy document**, not a test-case list. It tells you *where to focus*, *why it matters*, and *what experienced QA engineers check first*. Flow diagrams live in `business-data-map.md`. Feature catalog lives in `business-feature-map.md`. Test cases and traceability live in the TMS.

---

## 1. Visual Header

Bunkai TMS is a multi-tenant, API-first Test Management System. Its value proposition — IQL traceability from User Story → AC → ATC → Run — depends entirely on data integrity. Every critical mutation is atomic (RPC-backed), every identity decision runs through a dual-mode auth gate (`requireAuth()`), and every piece of business data is protected by Row Level Security. The system is narrow in Phase 1: read-only project/module hierarchy, no run execution API, no automated email. That narrowness is a gift — fewer moving parts means the ones that exist are load-bearing.

**The fragile zones**: authentication, the ATC save RPC, workspace onboarding, and RLS enforcement. These four areas carry nearly all the blast radius in the system. If any one of them breaks, the product does not just degrade — it either locks users out, destroys test data, or silently exposes cross-tenant data.

---

## 2. Executive Risk Map

The system's most fragile area is the combination of two independent subsystems: the **auth gate** (which everything else depends on) and the **`bunkai_save_atc` RPC** (which is the core product mutation). Neither is particularly complex in isolation, but both carry an outsized blast radius — auth because it gates every other feature, and the RPC because its delete-then-reinsert pattern means a partial failure silently wipes test data. A third hidden risk is **RLS correctness**: Supabase RLS is the last line of defense against cross-tenant data leakage, and four SECURITY DEFINER functions sit in that boundary. Finally, the **workspace onboarding RPC** creates a catastrophic state if it partially fails — a user can be fully authenticated but permanently locked out with no self-recovery.

| Priority | Flow / Area | Why it matters | Depends on / Affects |
|----------|-------------|----------------|---------------------|
| CRITICAL | Magic-Link Authentication | Security + 100% browser user lockout; silent failure mode (API returns 200 when email provider is down) | Everything gated by session cookie |
| CRITICAL | ATC Save RPC (`bunkai_save_atc`) | Delete-then-reinsert: a partial failure mid-transaction wipes steps, assertions, and AC bindings permanently | ATC integrity → IQL traceability |
| CRITICAL | RLS Enforcement | Cross-tenant data exposure; 4 SECURITY DEFINER functions sit in this boundary | All workspace-scoped data |
| CRITICAL | Workspace Onboarding Bootstrap | RPC partial failure = authenticated user locked out of all content, no self-recovery | All content access (workspace is the root container) |
| HIGH | Headless Auth + PAT Mint | Single request mints a permanent scoped credential; scope enforcement is a security boundary | All CI/agent integrations |
| HIGH | Team Invite + Acceptance | 5-condition validation chain; email mismatch and expiry are security-sensitive | Workspace membership → access to all content |
| HIGH | PAT Revocation | Zero-delay enforcement is the contract; any caching of validity state creates a window with a live-but-revoked token | Bearer auth on all protected endpoints |
| HIGH | ATC AC Anchoring (M:N) | Full replace-all on AC bindings: empty selection silently removes all traceability | IQL methodology (ATC → AC → Story traceability) |
| HIGH | Scope Enforcement Completeness | `requireAuth()` vs `requireScopeOrCookie()` — endpoints that only call `requireAuth()` are reachable by any Bearer token regardless of scope | All API endpoints |
| MEDIUM | Module Tree Max Depth | CHECK constraint at 6 levels; off-by-one failures corrupt the hierarchy | Project navigation; ATC context |
| MEDIUM | Idempotency Key Deduplication | Replayed POSTs should return cached response; which routes opt in is undocumented | Workspace creation, invite issuance |
| MEDIUM | ATC Full-Text Search (TSV trigger) | `atcs_refresh_tsv` trigger fires on title/tag change — stale index silently breaks search | Future search UI (FEAT-022) |
| MEDIUM | Activity Log Write Coverage | Audit trail; unclear which mutations write to it — silent audit gaps are invisible | Compliance, debugging |
| LOW | Project / Module Read-Only Display | Server-rendered via direct Supabase query; no API surface to break | UX polish |

---

## 3. What to Test First and Why

---

### CRITICAL — Magic-Link Authentication

**Why it matters**: Magic-link is the only login method for browser users in Phase 1. If it breaks — or if Supabase's email service has an outage — 100% of browser users are silently locked out. The word "silently" is what makes this especially dangerous: `POST /api/v1/auth/magic-link` returns `{ ok: true }` regardless of whether Supabase actually queued the email. From the API response alone, you cannot tell if the user will receive a link. The only way to know is to verify the email arrived, which requires either a real inbox or a Supabase email log.

**What commonly breaks**:
- Supabase email provider is rate-limited or down — API returns 200, user waits forever
- Open-redirect bypass: `next` parameter with `//attacker.com` path (the guard blocks `//` prefix but variations should be tested)
- OTP code expired before user clicks — the `/auth/callback` must return a user-friendly error, not a 500
- Callback sets cookie but redirect points to a wrong `next` value — user ends up on the right page but with a broken session
- `magic_link_tokens` accumulate (no cleanup job) — under high volume, does this affect lookup performance?

**Dependencies**: Supabase Auth (external, cannot be mocked in production testing). Every other feature in the system requires a valid session. If magic-link is broken, the entire product is unavailable to browser users.

**What an experienced QA checks**:
- Happy path: email submitted → callback with valid code → session cookie set → redirected to `next`
- Verify `/auth/callback?code=INVALID` redirects to `/login?error=otp_exchange_failed`, not a 500
- Test `next` parameter guard: `?next=/` (valid), `?next=//evil.com` (must reject), `?next=http://external.com` (must reject)
- Verify that a second click on the same magic-link (already consumed) fails gracefully
- Verify that after successful auth, the session cookie is httpOnly and Secure

---

### CRITICAL — ATC Save RPC (`bunkai_save_atc`)

**Why it matters**: This is the most dangerous mutation in the product. The RPC's delete-then-reinsert strategy for steps, assertions, and AC bindings means there is no "diff" — on every save, the old children are deleted and new ones are inserted in one transaction. If any INSERT in that chain fails (e.g., a position uniqueness violation, a constraint, a network timeout mid-RPC), PostgreSQL rolls back the transaction — but the ATC's metadata (`title`, `layer`, `tags`) may or may not be in a consistent state depending on where the failure occurred. The version counter increments on every save, which is the only signal that a save succeeded. From the UI, the only feedback is success or a generic error — there is no partial-state indicator.

**What commonly breaks**:
- Empty steps array after a "clear all" action — does the RPC correctly handle zero steps/assertions?
- Position sequence: after save, positions must be 0..N contiguous. If the client sends positions out of order or with gaps, what happens?
- AC binding with zero selected ACs — the feature map says "≥1 AC binding required" for validation, but what if the validation is frontend-only and a direct API call bypasses it?
- Concurrent saves from the same browser tab (double-click on save button) — the version counter provides an optimistic lock hint, but actual enforcement is "planned Phase E"
- Saving an ATC that belongs to a different workspace (cross-tenant RPC call) — is the RPC `SECURITY DEFINER`? If so, does it enforce workspace membership?

**Dependencies**: `atcs`, `atc_steps`, `atc_assertions`, `atc_acceptance_criteria`. Both triggers fire on the `atcs` table. `revalidatePath` must invalidate the SSR cache. The `saveAtcAction` is a Next.js Server Action, not a REST endpoint — it cannot be tested via `curl` or Playwright API mocking; it requires either a browser-driven test or a direct Supabase client call.

**What an experienced QA checks**:
- Happy path: save with 3 steps, 2 assertions, 2 AC bindings → verify DB state is exactly that
- Verify `version` counter increments on each save (read `atcs.version` before and after)
- Verify full-text search index (`tsv`) is updated after a tag change (query `atcs` table for the new tags)
- Verify that saving with an empty steps array leaves the ATC with zero steps (not leftover steps from previous save)
- Attempt save with a malformed step (missing `content`) — does it roll back cleanly or leave a corrupt state?
- Verify the ATC belongs to the caller's workspace before allowing the save

---

### CRITICAL — RLS Enforcement

**Why it matters**: Row Level Security is the last line of defense. Every query that reaches Supabase goes through RLS policies. If a policy is misconfigured — even for one table — a caller can read or write data from another tenant's workspace. There are four SECURITY DEFINER functions in the system (`bunkai_can_write_workspace`, `bunkai_is_workspace_admin`, `bunkai_is_workspace_owner`, `bunkai_bootstrap_workspace`). SECURITY DEFINER functions run with elevated privileges; a bug in their logic can silently bypass the policies they're supposed to enforce.

**What commonly breaks**:
- Cross-tenant read: caller with valid session for Workspace A queries `atcs` with a known ID from Workspace B — must return 0 rows or 404, never the data
- Cross-tenant write: `saveAtcAction` called with an `atcId` from a different workspace — must fail at RLS, not at application validation
- `bunkai_bootstrap_workspace` called by a user who is not the requesting user — SECURITY DEFINER means the DB function runs as its definer; can it be called with an arbitrary `user_id`?
- Suspended workspace members: a suspended member's session cookie should not grant data access — does the RLS policy check `status=active`?
- Bearer PAT cross-tenant: a PAT minted in Workspace A, used to query data from Workspace B (where the user also has membership) — should the PAT's `workspace_id` scope limit this?

**Dependencies**: Every feature in the system. RLS is not a single test — it's a test lens applied to every endpoint.

**What an experienced QA checks**:
- After creating two separate workspaces with two users, verify neither can read the other's ATCs, projects, or members
- Verify that a `workspace:admin` scoped PAT from Workspace A cannot perform admin actions in Workspace B (even if the same user is a member of both)
- Call `bunkai_save_atc` directly via Supabase client with an `atcId` that belongs to a different workspace — confirm RLS rejects it
- Verify that `suspended` member status blocks all data reads, not just the UI

---

### CRITICAL — Workspace Onboarding Bootstrap

**Why it matters**: The `bunkai_bootstrap_workspace` RPC creates the workspace and the owner membership in one atomic transaction. If the RPC succeeds for the workspace INSERT but fails for the membership INSERT, the user ends up with a workspace they cannot access — authenticated, but with no membership record, no RBAC, and no content. There is no self-service recovery for this state. The feature map notes this explicitly: "If the RPC partial-fails (workspace created, membership not), the user is authenticated but locked out of all content." Additionally, reserved slug blocking is a security boundary — if `api`, `auth`, or `admin` slugs are not blocked, a user could create a workspace that shadows system routes.

**What commonly breaks**:
- Reserved slug bypass: variations like `API` (uppercase), `api-team` (starts with reserved word but not exact match), `aPI` (mixed case)
- Slug collision race condition: two users submit the same slug simultaneously — only one 409 should succeed; both cannot create the workspace
- Slug format edge cases: exactly 2 characters (`ab`), exactly 40 characters (max), starting with a hyphen (`-acme`), ending with a hyphen (`acme-`)
- Post-RPC state after 409: does the form correctly display "Slug taken" without losing the user's other form data?
- What happens if the user navigates back to `/onboarding` after successfully creating a workspace? (Redirect guard)

**Dependencies**: `workspaces` table, `workspace_members` table, `bunkai_bootstrap_workspace` RPC. Once created, workspace feeds every content access path.

**What an experienced QA checks**:
- Happy path: submit valid `{ slug, name }` → workspace + owner membership created atomically → redirect to `/projects`
- All reserved slugs rejected with 409 + clear error message
- Slug regex edge cases: 1-char slug (must fail), 41-char slug (must fail), valid 2-char slug (must pass)
- 409 collision: create workspace with slug "acme-qa", attempt again with same slug → second returns 409, no second workspace created
- Verify workspace member record has `role=owner` and `status=active` immediately after creation

---

### HIGH — Headless Auth + PAT Mint

**Why it matters**: A single `POST /api/v1/auth/signin` request authenticates the user AND mints a permanent (up to 365-day) scoped credential. The raw token is returned once and never again. If a CI pipeline loses the token between the response and storage, it cannot recover without generating a new one. The scope enforcement is a hard security boundary — a token with `atc:read` scope must not be able to write.

**Dependencies**: Supabase Auth (password path), `access_tokens`, `access_token_secrets`. The PAT prefix index is the O(1) lookup for every subsequent authenticated call.

**What an experienced QA checks**:
- Mint a PAT with `atc:read` scope, then attempt `saveAtcAction` → must return 403
- Verify the raw token (`bk_pat_<12>.<32>`) is not stored in `access_tokens` — only the SHA-256 hash should be in `access_token_secrets`
- Verify `GET /api/v1/tokens` (list) never exposes the raw token in any response field
- Mint a PAT with `expires_in_days=1`, wait for expiry, attempt API call → must return 401
- Attempt to mint a PAT using an existing Bearer token (not a cookie) → must fail with 401 (chicken-and-egg guard)

---

### HIGH — Team Invite + Acceptance

**Why it matters**: The invite is the only way to grow a workspace. The 5-condition validation chain on acceptance (`exists → not revoked → not accepted → not expired → email match`) is a security gate. If any condition is incorrectly evaluated — especially the email case-insensitive match — an unauthorized user could join a workspace. The MVP has no automated email delivery, which means test data setup requires the tester to manually handle the token.

**Dependencies**: `workspace_invites`, `workspace_invite_secrets`, `workspace_members`. Token acceptance feeds workspace membership, which gates all content access.

**What an experienced QA checks**:
- Accept invite with wrong email (case-sensitive and case-insensitive variations) → must return 403
- Accept same token twice → first succeeds (200), second returns 409
- Accept an expired token (manipulate `expires_at` in DB or wait 7 days in test) → must return 410, not 404
- Revoke an invite → attempt acceptance → must return 409 with revoked status
- Rotate an invite (new token) → attempt acceptance with old token → must fail; new token must succeed
- Invite acceptance requires the invitee to be authenticated — attempt acceptance without a session → must redirect to login

---

### HIGH — PAT Revocation

**Why it matters**: The gap between a revoke action and its enforcement must be zero. Any caching of token validity state creates a window where a compromised-but-revoked token is still valid. The API map explicitly flags this: "tests must verify enforcement is synchronous, not eventually consistent."

**What an experienced QA checks**:
- Revoke a PAT, then immediately make a protected API call with that token → must return 401 on the very next request (no delay, no grace window)
- Verify that only the token owner can revoke it (forged `{id}` from another user's token returns 404, not 403 — indistinguishable from "not found")
- After revocation, verify `revoked_at` is stamped but the token record is NOT deleted (audit trail preserved)
- Attempt to revoke a token using another Bearer token (not a cookie) → must fail (cookie-only endpoint)

---

## 4. State Machines That Matter

---

### ATC Status Machine

The ATC status machine has no DB-level transition constraints — any status can be set to any other status at the application layer. This means the state machine is only as reliable as the UI enforcement. In Phase 1 there is no run-execution API, so status is set manually. The risk here is not invalid transitions (the DB won't reject them) — it's orphaned states.

**Transitions most likely to break**:
- `running` state with no active run session: since there is no run-execution API (FEAT-033 is planned), an ATC can get stuck in `running` permanently with no way to transition it except manual intervention
- `unrun` reset: the only way to re-test an ATC is to reset to `unrun` — if this action fails or is not surfaced in the UI, the ATC stays permanently in a terminal state

**Terminal / forbidden states to guard**:
- There are no formally forbidden transitions at the DB level. This is intentional, but it means your tests must verify the UI enforces the intended flow.
- `pass → running`: this transition makes sense for re-runs, but going `pass → fail` without going through `running` should probably not be allowed (business logic question — flag as discovery gap)

**How corruption would be detected**: No DB constraint rejects invalid transitions. The only signal is the visual status badge in `AtcTable` + `AtcEditor`. A corrupted status is invisible until a QA engineer notices the wrong color.

---

### Workspace Invite Lifecycle

This is the highest-stakes state machine in the system because it directly controls team access. The validation order on acceptance (`exists → not revoked → not accepted → not expired → email match`) is the security contract. If the order were changed — say, email match checked before expiry — the error codes would change, and a user with the wrong email would receive a different signal.

**Transitions most likely to break**:
- `issued → expired`: time-based, not event-driven. An expired invite must return 410 (Gone), not 404 — the distinction tells the invitee that the link existed but is no longer valid, vs. the token never existed
- `revoked → accepted`: should be impossible. The validation chain checks `not revoked` before attempting the upsert. Race condition: simultaneous revoke and accept could bypass this — test it

**Terminal states to guard**:
- `expired` and `revoked` are terminal. No recovery. Admin must issue a new invite (or rotate the existing one — rotation clears revocation state and resets expiry)
- `accepted` is also terminal — attempting a second acceptance must return 409, not re-upsert the member record

---

### PAT Lifecycle

**Transitions most likely to break**:
- `active → revoked`: the soft-delete path (`revoked_at = now()`) must be reflected in the `pat.ts` validation immediately
- `active → expired`: time-based. The check is `expires_at > now()` at lookup time — no background job transitions the status. If a token's `expires_at` passes, the next API call must return 401

**Terminal states to guard**: Both `revoked` and `expired` are terminal with no recovery. A revoked-then-deleted record (if the row were hard-deleted) would look identical to a non-existent token from the API perspective. Soft-delete is load-bearing for the audit trail.

---

### Workspace Member Status

**Transitions most likely to break**:
- `invited → active`: requires invite acceptance. There is no direct "activate member" API in Phase 1. The only path is through the invite flow
- `active → suspended`: no API endpoint confirmed in Phase 1 (listed as a discovery gap). If this transition exists at the DB level, test that suspended members cannot read any workspace data
- `suspended → active`: same gap — re-activation path is undocumented

---

## 5. Silent Killers — Automated Processes

This is the most undertested area in Bunkai TMS. While there are no traditional cron jobs in Phase 1, there are three automatic processes that run without any UI feedback path: two DB triggers and two TTL-based expiry columns with no cleanup workers.

---

### Silent Killer 1 — `atcs_refresh_tsv` Trigger

**What it does**: Fires `BEFORE INSERT / UPDATE` of `title` or `tags` on the `atcs` table. Refreshes the `tsv` tsvector column used for full-text search.

**What breaks if it misfires**:
- If the trigger fails (e.g., a schema mismatch after a migration), it will cause the INSERT/UPDATE to fail entirely — all ATC saves will fail with a cryptic DB error
- If the trigger succeeds but the `tsv` refresh lags (not applicable here — it's BEFORE, not AFTER), search results would be stale
- If a future migration changes the `title` or `tags` column type, the trigger body may silently compute a wrong `tsvector`

**How failure is detected today**: Only by observing stale search results or failed saves. There is no health check for this trigger.

**QA strategy**: After saving an ATC with a unique tag (e.g., `silent-killer-probe-TIMESTAMP`), query the `atcs` table directly via Supabase and verify `tsv` contains that tag. This is a DB-level assertion, not a UI check.

---

### Silent Killer 2 — `idempotency_keys` TTL Accumulation

**What it does**: `idempotency_keys` has an `expires_at` column set to `now() + 24h`. There is **no background job** that purges expired rows. Rows accumulate indefinitely.

**What breaks if left unchecked**:
- Under high API traffic, the `idempotency_keys` table grows without bound. PostgreSQL does not automatically clean up rows past their `expires_at` — only a `DELETE WHERE expires_at < now()` job would do that
- A very large `idempotency_keys` table will slow down the UNIQUE constraint check on `(user_id, endpoint, key)`, which is on the hot path for all POST endpoints that enforce idempotency

**How failure is detected today**: No alerts, no monitoring. The only signal is eventually degraded API response times.

**QA strategy**: Synthetic audit — query `SELECT COUNT(*) FROM idempotency_keys WHERE expires_at < now()` after running tests. A non-zero count should be flagged. Recommend this as a health-check assertion in CI.

---

### Silent Killer 3 — `magic_link_tokens` Expiry Accumulation

**What it does**: Same pattern as idempotency keys. `magic_link_token_secrets` rows have `expires_at` and `consumed_at`, but no background cleanup job was found in Phase 1.

**What breaks if left unchecked**:
- Table grows indefinitely. For a production system with many login attempts, this creates unbounded storage growth
- A consumed but never-cleaned token could theoretically be replayed if the `consumed_at` check has a bug

**QA strategy**: Verify that a consumed magic-link token (one that's been used for login) cannot be replayed — `GET /auth/callback?code=already-used` must return an error, not re-issue a session.

---

### Silent Killer 4 — `bk_active_ws` Cookie Lifecycle

**What it does**: `POST /api/v1/me/active-workspace` sets an httpOnly `bk_active_ws` cookie indicating which workspace is "active". This cookie is used by `GET /api/v1/me` to return the active workspace.

**What breaks if uncontrolled**:
- If logout does not clear `bk_active_ws`, the next user on the same browser would have a stale active-workspace pointer. Combined with a new session, this could cause the new user to load content from a workspace they don't belong to — until the RLS rejects the query. The UI experience would be confusing (flash of wrong workspace before redirect)
- If a user is suspended from a workspace, their `bk_active_ws` still points to that workspace. The RLS will block the data query, but the UI may show an error state rather than gracefully redirecting to a valid workspace

**How failure is detected today**: No monitoring. Only visible when a user reports wrong workspace context on login.

**QA strategy**: Log in, set active workspace to Workspace A, log out, log in as a different user — verify `bk_active_ws` cookie is absent or overwritten with the new user's workspace context.

---

## 6. External Integrations — Failure Points

| Integration | Which journey stops | Critical timeouts | Acceptable degradation | Known quirks |
|-------------|--------------------|--------------------|----------------------|--------------|
| **Supabase Auth (OTP email)** | Journey 1 — Magic-Link Login | None specified; fire-and-forget from API perspective | None — if Supabase Auth is down, there is no fallback login method for browser users | API returns `{ ok: true }` regardless of email delivery; silent failure is undetectable from API response alone |
| **Supabase Auth (password sign-in)** | Journey 2 — Headless Sign-in + PAT | No timeout configured in source | Agents without a PAT are blocked | Wrong credentials return 401; service down returns 500; distinguish these in tests |
| **Supabase PostgreSQL (all tables)** | All journeys — 100% data unavailable | No DB timeout configuration found in Phase 1 | Nothing works without the DB | RLS errors surface as 0 rows, not explicit 403 — consumers must handle empty response |
| **Supabase PostgreSQL (RPCs)** | Journey 3 (workspace), Journey 5 (ATC save) | RPC timeout not configured | None — RPC failure rolls back completely | `bunkai_save_atc` is the most complex RPC; `bunkai_bootstrap_workspace` is the most consequential if it partial-fails |
| **Vercel** | All journeys — deployment failure | No `vercel.json` found; timeout limits unverified | None — a failed deploy serves stale code | No rollback automation confirmed; stale edge cache behavior after deploy is unverified |
| **Jira (BK project)** | FEAT-019, FEAT-020 — story external_id linkage | No live API call — Jira key stored as string only | Jira being down has no real-time impact | No sync failure possible since there is no active API call; linkage is display-only |
| **Resend (planned FEAT-034)** | Journey 4 — invite email delivery | Not wired; not in `package.json` | MVP degrades to clipboard-copy (already the current behavior) | Do not test Resend integration — it is not implemented. Test the clipboard path and the token acceptance path only |
| **n8n (planned FEAT-035)** | No current impact | Not wired | Not applicable | Do not test n8n — not implemented |

---

## 7. Dependency Cascade Between Flows

```
Supabase Auth (OTP)
    |
    +── Magic-Link Login ──────────────────────────────────────────────────────────────+
    |   (Flow 1)                                                                        |
    |   └── Session Cookie set                                                          |
    |            |                                                                      |
    |            +── Workspace Onboarding (Flow 3) ────────────────────────────────+   |
    |            |   bunkai_bootstrap_workspace RPC                                 |   |
    |            |   └── workspace_members (owner, active)                          |   |
    |            |            |                                                      |   |
    |            |            +── Team Invite (Flow 5) ─────────────────────────+  |   |
    |            |            |   workspace_invites + secrets                    |  |   |
    |            |            |   └── Invite Acceptance ───────────────────────>|  |   |
    |            |            |       workspace_members UPSERT (new member)      |  |   |
    |            |            |                                                   |  |   |
    |            |            +── ATC Save (Flow 4) ─────────────────────────+  |  |   |
    |            |                bunkai_save_atc RPC                          |  |  |   |
    |            |                └── atc_steps + assertions + AC bindings     |  |  |   |
    |            |                         |                                    |  |  |   |
    |            |                         +── atcs_refresh_tsv (trigger) ────>|  |  |   |
    |            |                                                               |  |  |   |
RLS enforcement <──────────────────────────────────────────────────────────────>+  |   |
(last defense on all DB queries)                                                    |   |
                                                                                    v   |
                                                              Workspace membership ─────+
                                                              (the access gate for everything)
```

**The three critical chains**:

1. **Auth → Everything**: Magic-link is the only browser login. Supabase Auth outage = 100% of browser users locked out. No fallback.

2. **Workspace RPC → Content access**: `bunkai_bootstrap_workspace` runs once per user. A partial failure creates an authenticated user with no workspace membership — every content endpoint returns empty or 403 (RLS). No self-service recovery path exists in Phase 1.

3. **ATC Save → IQL integrity**: `bunkai_save_atc` replaces ALL child records. A failed save mid-RPC rolls back to the state before the save was triggered — meaning the user's edits are lost, but the ATC is intact. However, a bug in the RPC that causes the transaction to partially commit before rolling back (which should be impossible in PostgreSQL ACID, but is worth verifying) could leave an ATC with no steps, no assertions, and no AC bindings — a test case with no content.

---

## 8. Edge Cases Developers Commonly Forget

**Concurrency**:
- `saveAtcAction` double-click: React's `useOptimistic` or disabled-button debounce are UI mitigations, but if two saves reach the server within milliseconds of each other, the second one clobbers the first. The version counter exists but optimistic lock enforcement is "planned Phase E" — there is no guard today.
- Workspace slug uniqueness race: two users simultaneously submit the same slug to `POST /api/v1/workspaces`. The UNIQUE constraint on `workspaces.slug` will reject the second — but only if the DB constraint fires correctly. The 409 response body should be identical in both paths.

**Data limits**:
- Module tree depth: the CHECK constraint fires at depth > 6. Test depth exactly 6 (valid) and depth 7 (must fail). Off-by-one errors in `parent_module_id` resolution are the most common source.
- PAT `expires_in_days`: max documented as 365. What happens at `expires_in_days=366`? At `expires_in_days=0`? At `expires_in_days=-1`?
- ATC tags: the `tags[]` column has a GIN index. There is no documented max tag count. A tag array with hundreds of entries would bloat the `tsv` tsvector — test with a pathological tag count.

**Timezone / DST**:
- Invite expiry `now() + 7 days` is computed in PostgreSQL, which uses the server timezone. If the Supabase instance is in UTC and the test asserts on a local clock, you'll see 1-hour discrepancies during DST transitions.
- PAT `expires_in_days` arithmetic: `created_at + interval '365 days'` — does this respect DST or treat a day as always 24h?

**Permission boundaries**:
- `requireAuth()` vs `requireScopeOrCookie()`: the API map flags that some endpoints may only call `requireAuth()` without a scope check, meaning any valid Bearer token (regardless of scope) can call them. This is an audit finding, not just a test finding.
- `bk_active_ws` cookie not re-validated on every request: if a user is removed from Workspace A but still has `bk_active_ws=A`, what happens on their next request? The RLS will block the data query — but is the middleware handling this gracefully?
- SECURITY DEFINER functions: `bunkai_bootstrap_workspace` and `bunkai_save_atc` run with elevated DB privileges. Can a Supabase user call these directly via Supabase JS client? If yes, they bypass the Next.js application layer (auth, validation, idempotency).

**Orphaned states**:
- ATC stuck in `running`: since there is no run-execution API, once an ATC is set to `running`, only a manual status change can move it. A crashed test run would orphan ATCs in `running` permanently.
- Authenticated user with no workspace: a user who creates an account via `POST /api/v1/auth/signup` but never calls `POST /api/v1/workspaces` has a valid session but no content access. The app should redirect them to `/onboarding` — does it do this consistently for every protected page?

**Idempotency**:
- Retry a `POST /api/v1/workspaces` with the same `Idempotency-Key` header after a 5xx response — must return the cached response without creating a second workspace.
- Simultaneous invite acceptance (same token, two concurrent requests): the upsert should produce one `workspace_members` row and one `accepted_at` stamp. A race condition could produce two successful 200 responses — verify by checking the DB.

---

## 9. Pre-Release Checklist (Priority-Ordered)

1. Verify Magic-Link happy path from real email to authenticated session in staging environment
2. Verify `/auth/callback?code=INVALID` returns 302 to `/login?error=...`, not a 500
3. Verify `next` parameter open-redirect guard: test `?next=//evil.com` returns 302 to `/projects` or similar safe fallback
4. Verify `bunkai_save_atc` produces correct DB state: steps, assertions, and AC bindings match exactly what was submitted
5. Verify ATC save with empty steps/assertions array does not leave orphaned child rows
6. Verify cross-tenant RLS: User A cannot read User B's ATCs even with a valid PAT
7. Verify PAT revocation takes effect synchronously: revoke → immediate next call returns 401
8. Verify `POST /api/v1/workspaces` returns 409 on reserved slug (test at least: `api`, `auth`, `admin`)
9. Verify invite acceptance with wrong email returns 403; with expired token returns 410; with already-accepted token returns 409
10. Verify workspace onboarding creates both workspace and owner membership in one atomic operation — no partial state in DB
11. Verify `atcs_refresh_tsv` trigger updates `tsv` after a tag change — query Supabase directly to confirm
12. Verify that a suspended workspace member's session cookie does not grant data access (RLS check)
13. Verify Bearer token with `atc:read` scope cannot call `saveAtcAction` or any write endpoint
14. Verify `GET /api/v1/health` returns 200 with correct `environment` field (no `undefined` values in response)
15. Verify `bk_active_ws` cookie is absent or overwritten after logout (session cleanup)

---

## 10. What Is NOT in This Plan

- **Flow-level diagrams and state-machine transition tables** → `.context/business/business-data-map.md`
- **Feature catalog, CRUD matrix, feature flags, planned features** → `.context/business/business-feature-map.md`
- **API endpoint inventory, auth tier table, journey narratives** → `.context/business/business-api-map.md`
- **Detailed test case definitions, step-by-step ATCs, traceability links** → TMS (see `/test-documentation`)
- **Sprint-level execution order, ATR reports** → `.context/PBI/` (see `/sprint-testing`)
- **KATA architecture, fixture selection, TypeScript patterns** → `.claude/skills/test-automation/references/`
- **Performance testing, load testing, accessibility audits** → not in scope for Phase 1 (no SLAs documented beyond design-constraint targets)

---

## 11. Discovery Gaps

The following areas could not be fully grounded in Phase 1 evidence. They affect test design decisions and should be verified with the development team before sprint-level testing begins.

| Gap | Impact on Test Design | Recommended Action |
|-----|-----------------------|--------------------|
| **Idempotency enforcement scope** — `lib/api/idempotency.ts` exists but which POST endpoints actually call it is not confirmed | Cannot design replay tests without knowing which routes opt in | Grep `idempotency.ts` usages across all route handlers before designing idempotency tests |
| **Activity log write triggers** — which mutations write to `activity_log` automatically is unclear | Cannot verify audit completeness; audit coverage tests would be guessing | Grep `activity_log` inserts in route handlers + RPC body (`0009_cross_cutting.sql`) |
| **`bk_active_ws` cookie on logout** — whether logout clears this cookie is unconfirmed | Could leak workspace context to next session; test setup may need manual cookie teardown | Trace the logout route (if any) — it was not found in Phase 1 source; may not exist |
| **`bunkai_save_atc` SECURITY DEFINER** — whether this RPC is `SECURITY DEFINER` determines if a Supabase user can call it directly and bypass Next.js auth | Direct-client attacks on the RPC may be possible | Read `supabase/migrations/0007_save_atc.sql` — check `LANGUAGE` and `SECURITY` clauses |
| **ATC status transition business rules** — the DB has no constraint on transition order; implied business rules (e.g., can you set `pass` without going through `running`?) are undocumented | Transition tests may be testing non-contractual behavior | Confirm with product team which transitions are intentionally allowed and which are gaps |
| **Scope enforcement completeness** — which endpoints call `requireScopeOrCookie()` vs plain `requireAuth()` is not traced | Bearer token scope bypass is untestable without the call-site audit | Audit all `requireAuth()` and `requireScopeOrCookie()` call sites across `/api/v1/` handlers |
| **`bunkai_save_atc` RPC privilege model** — if `SECURITY DEFINER`, a low-privilege Supabase user may be able to call the RPC directly and save ATCs to any workspace | Cross-tenant ATC write via direct Supabase client is possible if unguarded | Verify RPC `SECURITY` setting and whether it checks workspace membership internally |
| **PAT timing-safe comparison** — whether `lib/api/pat.ts` uses a constant-time hash comparison is unverified | Timing attack on token validation (low probability, but relevant for a security-centric product) | Read `lib/api/pat.ts` — look for `timingSafeEqual` or equivalent; flag if missing |
| **ATC deletion path** — there is no delete endpoint; ATCs can only be abandoned | Test teardown in automation cannot remove created ATCs; tests must use isolated workspaces or seed data management | Confirm with product team if this is intentional; recommend a `DELETE /api/v1/atcs/{id}` for QA environments |
| **Project CRUD gap** — projects exist in DB but have no API creation endpoint | Test setup cannot create projects via API; must rely on DB seed or migrations | Confirm if projects are seeded via migration for testing, or if a `POST /api/v1/projects` is coming |

---

*Cross-reference*:
- `business-data-map.md` — entity map, state machines, system flows, DB triggers
- `business-feature-map.md` — feature catalog, CRUD matrix, QA relevance matrix
- `business-api-map.md` — auth model, journey narratives, integration failure modes, discovery gaps
- `PRD/executive-summary.md` — product scope, Phase 1 constraints, Phase 2 deferrals

*Update triggers*: Re-run `/master-test-plan` after — run execution API (FEAT-033) lands · Resend integration (FEAT-034) wired · Project/Module CRUD API added · New auth scheme (GitHub OAuth / Google SSO) added*
