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
- **DBHub** (re-diagnosed 2026-06-29): credentials VALID — probed real PG connection (Bun.SQL, sslmode=require) → connects as `qa_inspector_ro` (read-only), sees 5 atc* tables. `.mcp.json` + `dbhub.toml` (${VAR} interpolation) are correct. MCP simply did not spawn this session → **needs a session RESTART only** (no config/cred change).
- **AUTH is PASSWORDLESS** (re-diagnosed 2026-06-29): prior "STAGING_USER_PASSWORD invalid" was a MISDIAGNOSIS. Bunkai login = Supabase magic-link / email OTP via Resend; the account has NO password, so `POST /auth/signin` (email+password) cannot auth it. Headless path = OTP: `POST /auth/magic-link` → read 6–8 digit code from Resend inbound → `POST /auth/confirm {email, token, pat_scopes}` → session + auto-minted PAT (scopes requestable inline). `confirm`/`signin` auto-mint a PAT; `POST /tokens` is cookie-session-only.
- **Resend inbound VALIDATED**: `resend emails receiving list/get` reads the inbox. Correct domain = `@vexaakarii.resend.app` (X), a Resend sandbox inbox (receives by default; not in `domains list`). `.env` STAGING_USER_EMAIL had a typo `vezaakarii` (Z) → **FIXED to vexaakarii (X)** 2026-06-29. Account `bunkai-dojo3-staging@vexaakarii.resend.app` exists+confirmed.
- **Auth currently RATE-LIMITED**: `POST /auth/magic-link` → 429 `email rate limit exceeded` (Supabase free tier, several signups burned it today). Retry after ~1h. Path is proven; only the limit blocks it.
- **TC22 (403) path**: mint a `["run:execute"]`-only PAT (no atc:read) → 403 on search. Get a session via the OTP flow above, then either pass `pat_scopes:["run:execute"]` to `confirm`, or `POST /tokens` with the cookie. **DEFERRED until rate-limit clears** — all other TCs run without a session.
- **TC17 P_OUT gap**: token user owns BOTH W1+W2 → every seeded project is within memberships. "project outside memberships" needs a foreign project_id → source via DBHub after restart, OR a 2nd signup user (email OTP via resend).
- **TC08 recency**: 30-day-old `updated_at` ideally backdated via DBHub; otherwise approximate via creation-order delta.

## Stage state
- Session Start: COMPLETE (user confirmed story explanation).
- Stage 1 Planning: COMPLETE (ATP written to BK-20 customfield_10120; 24 TCs).
- Stage 2 Execution: COMPLETE (2026-06-30). Smoke GO. 23/24 PASS, 1 FAIL (TC01 response-shape). Tenant isolation VERIFIED (API+SQL). Evidence in evidence/stage2-evidence.md + stage2-api-results.json. Verdict orchestrator-verified against raw API dump.
- **DEFECT (F1)** — `GET /api/v1/atcs/search` item returns `status="unrun"` (run-status enum) and field `id`, NOT the PO-decided `status_dot` lifecycle enum {draft,ready,automated,deprecated} / `atc_id`. Breaks the BK-5 picker's reuse signal. User (PO) classified as Defect → file in Stage 3, block BK-20. F2/F3 = notes only (correct behavior / unconstructible backdate).
- Stage 3 Reporting: COMPLETE (2026-06-30). Defect **BK-187** filed (type Defect, Severity Mayor → Priority High, parent QA Defect Management epic **BK-183**, component "ATC Library (Acceptance Test Cases)", qa_assignee Facu Barea). ATR written to BK-20 (real ATR field = customfield_10147, NOT catalog's 10284). QA comment posted (FAILED, id 11841). BK-20 transitioned Ready For QA → In Test → **BLOCKED**. **OPEN MANUAL STEP:** the `BK-187 blocks BK-20` issuelink could NOT be created — token lacks "Link Issues" permission on BK-20 (HTTP 401). Create manually: BK-187 "blocks" BK-20 (BK-20 "is blocked by" BK-187).
- **CATALOG DRIFT FOUND:** `.agents/jira-fields.json` field IDs do NOT match the live BK project (Defect screen severity=10143 not 10177, ATR=10147 not 10284, qa_assignee=10188, etc.). All Defect/ATR field IDs were resolved via live createmeta/editmeta. Recommend `bun run jira:sync-fields --force` then `bun run jira:check`. Sync cache `acceptance-test-results.md` did NOT materialize because the sync script reads the stale catalog (10284, absent on BK); the ATR itself is correctly in Jira on 10147.

## Session log — 2026-06-29 (resume)
- Resumed at Stage 2. Re-diagnosed both blockers: DBHub = restart-only (creds valid); auth = passwordless (signin N/A), OTP path validated via Resend inbound but rate-limited now.
- Fixed `.env` STAGING_USER_EMAIL domain typo (vezaakarii→vexaakarii). User approved full-auto Resend OTP path.
- NEXT ACTION: user restarts session → DBHub spawns → run Stage 2 (TC22 deferred).
