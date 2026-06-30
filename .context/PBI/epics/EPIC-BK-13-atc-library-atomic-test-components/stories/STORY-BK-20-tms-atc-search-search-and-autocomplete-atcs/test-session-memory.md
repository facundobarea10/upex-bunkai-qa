# BK-20 — Test Session Memory (cross-stage shared state)

> Hand-authored. Load-bearing across the 4 sub-agent dispatches (Session Start → Stage 1 → Stage 2 → Stage 3). NON-Jira file.

## Ticket
- **Key:** BK-20 — TMS-ATC Search | Search and autocomplete ATCs
- **Epic:** BK-13 (ATC Library) · **Type:** Story · **Status at entry:** Ready For QA · **Points:** 5
- **Surface:** API-only (search box UI lives in EPIC-BK-5). Testing = API + DB. No browser UI for this story.
- **Shift-Left:** `shift-left-reviewed` label, dated 2026-06-01 (23 days old < 30 → valid). Stage 1 short-circuits ATP Phases 1–3, starts at Phase 4 and reconciles against real contract.

## TMS modality
- **Jira-native** (no Xray Test Plan / Test Execution issues). ATP → Story custom field `{{jira.acceptance_test_plan}}` (customfield_10120). ATR → `{{jira.acceptance_test_results}}`. TCs = Jira `Test` issues. Writes via `/acli`.

## Environment
- **Active env:** staging (default). No override.
- WEB_URL: https://staging-upexbunkai.vercel.app (307 redirect = reachable)
- API_URL: https://staging-upexbunkai.vercel.app/api
- Creds: `.env` → `STAGING_USER_EMAIL` / `STAGING_USER_PASSWORD`. API_TOKEN present + valid (OpenAPI MCP returned 53 endpoints).
- DB: `DBHUB_*` keys present in `.env`. DBHub MCP must be probed before Stage 2 DB validation.

## REAL implemented contract — `GET /api/v1/atcs/search` (source: OpenAPI MCP, authoritative)
- **Auth:** Bearer `atc:read` OR cookie session.
- **Params:**
  - `query` (string, **required**, minLength 1) — single token = prefix-aware (autocomplete); multi-word = AND semantics.
  - `project_id` (uuid, **REQUIRED**) — scopes to one project; project outside caller's active workspaces → no rows.
  - `module_id` (uuid, optional) — narrows to module + descendant subtree.
  - `layer` (enum, optional) — `UI | API | Unit`.
  - `limit` (int, optional) — 1..50, default 20.
- **Responses:** 200 `{items: AtcSearchResult[]}` (possibly empty, never 404 on zero matches) · 401 not authenticated · 403 missing `atc:read` scope · **422** validation failed (empty/missing query, missing/invalid project_id, bad limit, bad layer).

## Contract deltas vs Shift-Left ATP (MUST reconcile in Stage 1)
1. Empty/absent/whitespace query → ATP said **400**; real = **422**. Fix all S5.* expected codes.
2. `project_id` is **REQUIRED** (ATP omitted it; scoped only by workspace). Adds: missing project_id → 422; project outside memberships → empty items. Reframes S3 (subtree) and S6 (isolation): isolation is now project-scoped AND workspace-scoped.
3. **403** (PAT without `atc:read` scope) is a NEW dimension absent from the ATP. Add a negative TC.
4. Prefix matching = CONFIRMED (no longer "NEEDS PO/DEV CONFIRMATION"). Single-token prefix, multi-word AND.
5. `limit` bounds enforced by schema (1..50). limit=0 / limit=51 → 422 (was "open item").
6. Zero matches → 200 `{items:[]}` CONFIRMED (never 404).

## Open data questions for Stage 1
- Workspace isolation TC needs 2 workspaces (W1, W2) the same or different user belongs to, each with a matching ATC. Determine available test tenants on staging.
- `project_id` test data: need a known project with seeded ATCs (title + tags + module subtree + varying `updated_at` + layers). May need to seed via POST /api/v1/atcs.
- `status_dot` response field values: `draft / ready / automated / deprecated` (per PO decision). Verify against `AtcSearchResult` schema.

## Stage 2 readiness (probed 2026-06-24)
- **Auth identity**: API_TOKEN = `mail-api-postman@vexaakarii.resend.app`. Owns 2 workspaces: W1 `qa-api` (1bcc4bd5-3135-4712-8321-3d77c56fa28a, active) + W2 `qa-api-2` (a4628b6a-72f6-45a3-a52e-e31cb3760659). Scopes: atc:read, atc:write, run:execute. NOT the same user as STAGING_USER.
- **No project discovery via API**: no GET /projects endpoint; search requires `project_id`. Must seed via API (atc:write present) or source via DB.
- **DBHub**: was down — `.mcp.json` dbhub block lacked `env` → `${DBHUB_*}` unresolved. FIXED `.mcp.json` (added env block mirroring openapi). DBHUB_* values present in .env. **Requires session restart to take effect.**
- **STAGING_USER_PASSWORD invalid**: 5 chars; API requires ≥6 → POST /auth/signin returns 422 `too_small`. Blocks headless session → blocks session-minted read-less PAT for TC22. **Needs correct password in .env.**
- **TC22 (403) path CONFIRMED feasible**: CreateTokenBody.scopes enum = [atc:read, atc:write, run:execute, workspace:admin], minItems 1. Mint a `["run:execute"]`-only PAT via a session → 403 on search. Requires a valid session (fix STAGING_USER_PASSWORD).
- **TC17 P_OUT gap**: token user owns BOTH W1+W2 → every seeded project is within memberships. "project outside memberships" needs a foreign project_id → source via DBHub after restart, OR a 2nd signup user (email OTP via resend).
- **TC08 recency**: 30-day-old `updated_at` ideally backdated via DBHub; otherwise approximate via creation-order delta.

## Stage state
- Session Start: COMPLETE (user confirmed story explanation).
- Stage 1 Planning: COMPLETE (ATP written to BK-20 customfield_10120; 24 TCs).
- Stage 2 Execution: BLOCKED pending — DBHub restart + STAGING_USER_PASSWORD fix. User approved seeding + DBHub fix+restart.
- Stage 3 Reporting: pending.
