# Shift-Left Refinement: BK-20 — TMS-ATC Search | Search and autocomplete ATCs

**Status**: Refined — Awaiting PO Estimation
**Mode**: Shift-Left (pre-sprint, single-story batch)
**Refined on**: 2026-06-01
**Refined by**: QA — Shift-Left batch session
**Modality**: Jira-native

---

## Phase 1 — Critical Analysis

### Business context

- **Primary persona affected**: Senior QA Engineer (Elena) who composes Tests from reusable ATCs
- **Secondary personas**: Any workspace member who consumes the ATC picker in the Test composition UI (EPIC-BK-5)
- **Business value proposition**: Eliminates manual scrolling through deep Module trees to locate ATCs. Faster ATC reuse → faster test composition → higher IQL coverage.
- **KPIs influenced**: ATC reuse rate, time-to-test-composition per story, duplicate ATC creation rate (should fall)
- **User journey position**: BK-20 is the discovery surface of the ATC Library. It sits between ATC creation (BK-18/BK-19) and Test composition (EPIC-BK-5). The picker autocomplete is the primary consumption point.

### Technical context

- **Frontend**: ATC search box / picker UI — the rendering component is referenced in EPIC-BK-5 (Test composition). BK-20 is the backend contract; UI is consumed by the picker. No dedicated frontend Story for the search box itself has been identified yet — **this is a scope gap to surface**.
- **Backend**: `GET /api/v1/atcs/search` (NEW endpoint). Query params: `query` (required, ≥1 char), `module_id` (optional UUID), `layer` (optional enum: UI|API|Unit), `limit` (optional int, default 20, cap 50). Response: `{ items: [{ atc_id, slug, title, module_path, layer, status_dot }] }`.
- **DB layer**: `atcs` table gains `search_tsv tsvector` column + GIN index (`atcs_search_tsv_idx`) + trigger `atcs_tsv_trg` (fires BEFORE INSERT OR UPDATE OF title, tags). Ranking SQL: `ts_rank(search_tsv, plainto_tsquery('simple', $1)) * exp(-EXTRACT(EPOCH FROM (now() - updated_at)) / $decay)` where `$decay = 604800` (7 days in seconds). Module subtree filter via recursive CTE or materialized `module_paths`.
- **External services**: PostgreSQL FTS (`tsvector`, `to_tsvector`, `ts_rank`, `plainto_tsquery`, GIN index). No external APIs.
- **Integration points specific to this Story**: consumes `atcs.title`, `atcs.tags`, `atcs.updated_at`, `atcs.module_id`, `atcs.workspace_id` from BK-18's schema. Workspace scoping applied at service layer (defense in depth even with RLS). Module subtree reuses Wave 1 `module_paths` mechanism.

### Story complexity

| Axis | Rating | Why |
|------|--------|-----|
| Business logic | High | Ranking algorithm: FTS relevance × recency decay. Module subtree recursion. Workspace isolation as security boundary. |
| Integration | Medium | Depends on BK-18 (atcs table + columns), Wave 1 module tree mechanism, PostgreSQL FTS infrastructure. |
| Data validation | Medium | Query ≥1 char, limit 1–50, layer enum, module_id UUID format. |
| UI | Low | API-only Story. UI picker lives in EPIC-BK-5. |

**Estimated test effort**: 18–22 outlines (HIGH complexity — ranking logic, security boundary, FTS semantics, subtree filter each require dedicated scenarios).

### Epic-level inheritance (BK-13 — ATC Library)

- **Inherited risk**: Workspace isolation — any breach exposes cross-tenant test data. Master test plan classifies RLS enforcement as CRITICAL. BK-20 adds defense in depth (`WHERE workspace_id = $session.workspace_id`) on top of RLS.
- **Integration points inherited**: `atcs` table schema from BK-18, `module_paths`/recursive CTE from Wave 1 module Stories.
- **PO/Dev answers already given at BK-18 level (reused)**:
  - `atc:read` scope covers GET operations (not explicitly stated in BK-18, but implied by the scope model — **confirm in Critical Questions**).
  - `test_steps` table does not yet exist (EPIC-BK-5) — `used_in` expansion is out of scope for BK-20.
  - Events `atc.created`/`atc.updated` will update the search index in Wave 2 (BK-20 is listed as consumer).
- **Test strategy inherited**: every search result must go through workspace scope gate. Security tests are non-negotiable regardless of risk score.

---

## Phase 2 — Story Quality Analysis

### Ambiguities

| # | Location in Story | Question for PO/Dev | Impact on testing | Suggested clarification |
|---|-------------------|---------------------|-------------------|------------------------|
| A1 | Workflow: "suggestions appear and refine as she types" | The `plainto_tsquery('simple', $1)` used in the architect's SQL matches complete words only. If Elena types "expir", FTS will NOT match "expired". Does MVP need prefix matching (`to_tsquery` with `:*` modifier)? Or does the UI debounce until a full word is entered? | Cannot design autocomplete test cases without knowing match semantics | Clarify: exact-word FTS only (search on word-boundary) OR prefix-aware FTS. If prefix, architect SQL needs `to_tsquery('simple', $1 || ':*')`. |
| A2 | AC3: "filter to the Payment Module" | Does the module filter apply to the exact selected module only, OR to the selected module AND all its descendant sub-modules (recursive subtree)? Architect note says recursive CTE — but AC text is ambiguous. | Changes assertion: flat filter → 1 module; recursive → N modules | Confirm: filter is recursive subtree (module + all descendants), not just the selected module. |
| A3 | AC4: "two equally relevant ATCs" | "Equally relevant" implies identical `ts_rank` score. In practice, creating two ATCs with provably identical FTS rank requires careful test data control. Is there a simpler way to express this expectation — e.g., "when both ATCs contain identical tokens, the more recently updated one appears first"? | Makes the ranking test brittle if the ranking formula changes | Restate AC4 as: "When two ATCs produce identical text-relevance scores for a query, the one with a later `updated_at` ranks higher." |
| A4 | API contract: auth model | The story does not specify authentication. Does `GET /atcs/search` require a Bearer PAT (like BK-18's mutating endpoints), a session cookie (for browser users), or both? The session `workspace_id` is derived from the caller's identity — auth is load-bearing. | Changes auth test cases entirely | Confirm: `requireAuth()` (cookie OR Bearer) or `requireBearerToken()` only. |
| A5 | Response field `status_dot` | Architect specifies `status_dot` in the response shape. What values does this field take? Is it the ATC's workflow status? A color enum? Not defined in the ACs or business rules. | Cannot assert response shape without knowing field contract | Define: `status_dot` values and their semantics (e.g., `{draft, ready, automated, deprecated}`). |

### Gaps (missing info)

| # | Type | Why critical | What to add | Risk if omitted |
|---|------|--------------|-------------|-----------------|
| G1 | Missing AC | No AC covers authentication/authorization. `workspace_id` is derived from the session — unauthenticated calls have no workspace, so either a 401 or a panic results. | Add negative AC: "Unauthenticated request to GET /atcs/search returns 401" | Security gap: no test coverage on auth gate. Production 500 if auth middleware is not wired. |
| G2 | Missing AC | No AC covers the `limit` parameter behavior (default 20, cap 50). The business rule exists but no AC tests it. | Add boundary ACs: "Default limit is 20 when not specified" and "Limit is capped at 50 when 51+ is requested" | Unverified API contract — cap could be off-by-one or not enforced. |
| G3 | Missing AC | No AC covers the `layer` filter despite it being in the API contract (`layer` optional param). | Add AC: "When layer filter is applied, only ATCs of that layer appear in results" | Layer filter is an advertised feature with no test coverage. |
| G4 | Missing AC | No AC covers the response structure (`module_path` format, `status_dot`, `slug`). ACs only describe WHICH items appear, not WHAT the items look like. | Add response-shape validation AC | False confidence — API may return items but with malformed fields (e.g., `module_path` missing separator). |
| G5 | Scope gap | No frontend Story identified for the ATC search box / picker UI. The workflow describes a UI experience but no UI Story is linked. | Clarify: is the search box included in this Story, in EPIC-BK-5, or in a separate BK-? | Test scope ambiguity — QA cannot test UI if no UI Story exists. |
| G6 | Missing AC | No AC for the case where a valid query matches ZERO ATCs in the workspace. Should return `{items: []}` (200) or a 404? | Add AC: "A query with no matches returns 200 with an empty items list" | Ambiguous error handling — empty results are a normal state, not an error. |

### Edge cases not in Story

| # | Scenario | Expected behavior (best guess) | Criticality | Action |
|---|----------|-------------------------------|-------------|--------|
| E1 | Query with multiple words: "expired token" | FTS `plainto_tsquery` treats as AND of tokens — returns ATCs containing BOTH "expired" AND "token" in title/tags | High | Add as positive test case |
| E2 | Query with PostgreSQL FTS stop words ("the login") | `simple` dictionary does not suppress stop words — "the" is indexed. Behavior differs from `english` dictionary. | Medium | Test only — document behavior |
| E3 | `module_id` for a module that does not exist | Return 404 or return empty `{items: []}`? | High | **NEEDS PO/DEV CONFIRMATION** — 404 vs empty behavior undefined |
| E4 | `layer` param with invalid enum value (e.g., "Mobile") | Return 400 `validation_failed` | Medium | **NEEDS PO/DEV CONFIRMATION** — error response not specified |
| E5 | Whitespace-only query (" ") | Should be treated as empty (no search) OR as a validation error (400) | Medium | **NEEDS PO/DEV CONFIRMATION** |
| E6 | Query param absent entirely (URL has no `?query=`) | 400 `validation_failed` (query is required) | High | **NEEDS PO/DEV CONFIRMATION** |
| E7 | Workspace with 10,000+ ATCs — response time | Architect budget: < 100ms p95. GIN index is load-bearing. | High | Document as performance test outline — verify index exists before load test |
| E8 | ATC with null `updated_at` (if possible) | Ranking formula breaks (EXTRACT EPOCH from null). Should default to epoch 0 or be excluded | Medium | **NEEDS PO/DEV CONFIRMATION** — confirm `updated_at` has NOT NULL constraint |
| E9 | Two ATCs with identical `updated_at` timestamp | Tie-breaking order is undefined — non-deterministic ranking | Low | Document: no tie-break guarantee; test should not assert order for timestamp ties |
| E10 | Injected workspace_id query param (e.g., `?workspace_id=other-tenant-id`) | API must ignore it; always use session workspace_id | Critical | **NEEDS PO/DEV CONFIRMATION** — confirm param is not accepted or silently ignored |
| E11 | Query with SQL injection characters (`'; DROP TABLE atcs; --`) | Parameterized query via Supabase client — safe by default. But confirm `plainto_tsquery` sanitization. | High | Security test only — document mitigation |
| E12 | `limit=0` | Return 400 or default to 20? | Low | **NEEDS PO/DEV CONFIRMATION** |

### Contradictions

- **Workflow vs FTS semantics**: The Workflow section says "suggestions appear and refine as she types" (implying prefix/autocomplete behavior). The architect's SQL uses `plainto_tsquery('simple', $1)` which is exact-word matching — it does NOT match partial tokens. These two are contradictory: either the workflow description is aspirational and MVP is exact-word only, or the SQL needs `to_tsquery` with `:*` prefix. This must be resolved before estimation (see A1).

### Testability validation

**Verdict**: Partial

Issues:
- AC4 ("equally relevant") is vague — two ATCs being provably text-equivalent requires test-data control. Without knowing the ranking formula's precision, equality cannot be guaranteed in real tests. **Mitigable** by creating ATCs with identical titles and tags.
- AC3 ("sub-modules are shown") does not define the recursive depth (1 level? all descendants?). Partially testable as written but assertions depend on recursion depth.
- The response field `status_dot` is undefined — cannot assert it.
- Authentication mechanism unspecified — cannot design auth test cases.

---

## Phase 3 — Refined Acceptance Criteria

### Original AC1 — Find an ATC by a word in its title

#### Scenario 1.1: Should return ATC when query matches a word in its title (Positive, High)
- **Given**: Workspace W1 has an ATC titled "Login with expired token" (all fields indexed in `search_tsv`)
- **When**: User calls `GET /atcs/search?query=expired` with valid auth
- **Then**:
  - API returns 200
  - `items` array contains the ATC with `title = "Login with expired token"`
  - Response includes `atc_id`, `slug`, `title`, `module_path`, `layer`, `status_dot` fields

#### Scenario 1.2: Should return multiple ATCs when query matches a word in multiple titles (Positive, Medium)
- **Given**: Workspace W1 has two ATCs: "Login with expired token" and "Expired session redirect"
- **When**: User calls `GET /atcs/search?query=expired`
- **Then**: Both ATCs appear in `items`; ranked by relevance + recency

#### Scenario 1.3: Should NOT return ATC when query does not match any word in title or tags (Negative, High)
- **Given**: Workspace W1 has only ATCs with titles unrelated to "xyznotfound"
- **When**: User calls `GET /atcs/search?query=xyznotfound`
- **Then**: API returns 200 with `items: []`

---

### Original AC2 — Find an ATC by one of its tags

#### Scenario 2.1: Should return ATC when query matches a tag exactly (Positive, High)
- **Given**: Workspace W1 has an ATC tagged `["smoke", "login"]`
- **When**: User calls `GET /atcs/search?query=smoke`
- **Then**: The ATC appears in `items`

#### Scenario 2.2: Should return ATC when query matches a tag but not the title (Positive, Medium)
- **Given**: ATC has title "Navigate to homepage" and tags `["smoke"]`
- **When**: User calls `GET /atcs/search?query=smoke`
- **Then**: The ATC appears (tag match, weight B)

---

### Original AC3 — Narrow results to a Module subtree

#### Scenario 3.1: Should include ATCs from selected Module AND all descendant sub-modules (Positive, High)
- **Given**: Module "Payment" has sub-modules "Payment/Checkout" and "Payment/Refunds". Each sub-module has one ATC matching "flow"
- **When**: User calls `GET /atcs/search?query=flow&module_id=<Payment-id>`
- **Then**: All ATCs from "Payment", "Payment/Checkout", and "Payment/Refunds" appear in `items` — **NEEDS PO/DEV CONFIRMATION** (recursive behavior assumed from architect note)

#### Scenario 3.2: Should exclude ATCs from sibling Modules outside the selected subtree (Negative, High)
- **Given**: "Login" Module has an ATC matching "flow". "Payment" Module has a separate ATC matching "flow"
- **When**: User calls `GET /atcs/search?query=flow&module_id=<Payment-id>`
- **Then**: Only the Payment-subtree ATC appears; "Login" ATC is excluded

---

### Original AC4 — More recently updated ATCs rank higher among equal matches

#### Scenario 4.1: Should rank more recently updated ATC above an equally text-relevant older ATC (Positive, High)
- **Given**: Two ATCs with identical titles ("Validate login") and identical tags, one updated today and one updated 30 days ago
- **When**: User calls `GET /atcs/search?query=validate`
- **Then**: The ATC updated today appears first in `items`

#### Scenario 4.2: Should rank by text relevance first when relevance differs (NEEDS PO/DEV CONFIRMATION) (Edge, Medium)
- **NEEDS PO/DEV CONFIRMATION**: ranking formula implies text relevance × recency decay. If text relevance is higher for ATC-A, it should rank above ATC-B even if ATC-B is more recent.
- **Given**: ATC-A has "login" in both title AND tags (higher FTS weight). ATC-B has "login" only in tags. ATC-B is 1 day newer.
- **When**: User calls `GET /atcs/search?query=login`
- **Then**: ATC-A ranks above ATC-B (relevance beats recency when relevance differs significantly)

---

### Original AC5 — An empty query runs no search

#### Scenario 5.1: Should return empty result (no search) when query is empty string (Negative, High)
- **Given**: Any authenticated workspace with ATCs
- **When**: User calls `GET /atcs/search?query=`
- **Then**: API returns 200 with `items: []` — OR returns 400 validation error — **NEEDS PO/DEV CONFIRMATION** (behavior for empty string vs absent query)

#### Scenario 5.2: Should return 400 when `query` param is absent entirely (Negative, High) — NEEDS PO/DEV CONFIRMATION
- **NEEDS PO/DEV CONFIRMATION**: undefined whether absent param = 400 or treated as empty
- **Given**: Authenticated user calls `GET /atcs/search` with no `query` param
- **Then**: API returns 400 `validation_failed` (query is required per business rules)

#### Scenario 5.3: Should not search when query is whitespace-only (Negative, Medium) — NEEDS PO/DEV CONFIRMATION
- **NEEDS PO/DEV CONFIRMATION**: whitespace trimming behavior not specified
- **Given**: User calls `GET /atcs/search?query=%20%20%20` (3 spaces URL-encoded)
- **Then**: No search performed; return empty or 400

---

### Original AC6 — Results never include ATCs from another workspace

#### Scenario 6.1: Should not return ATCs from another workspace even when title matches (Security, Critical)
- **Given**: Workspace W1 has ATC "Login with expired token". Workspace W2 (different tenant) also has ATC "Login with expired token"
- **When**: User from W1 calls `GET /atcs/search?query=expired`
- **Then**: Only the W1 ATC appears. W2's ATC is absent.
- **Verify at**: API response (items only contain W1 atc_id) AND DB (workspace_id filter applied in query)

#### Scenario 6.2: Should use session workspace_id and ignore injected workspace_id param — NEEDS PO/DEV CONFIRMATION (Security, Critical)
- **NEEDS PO/DEV CONFIRMATION**: does the endpoint accept `workspace_id` as a query param? If it does, it must be silently ignored.
- **Given**: User from W1 calls `GET /atcs/search?query=expired&workspace_id=<W2-id>`
- **Then**: Results are still scoped to W1; W2 data does not appear

---

### New scenarios surfaced from gaps — NEEDS PO/DEV CONFIRMATION

#### Scenario G1.1: Should return 401 when called without authentication (Security, Critical) — NEEDS PO/DEV CONFIRMATION
- **NEEDS PO/DEV CONFIRMATION**: auth model not specified in original story
- **Given**: No Authorization header and no session cookie
- **When**: Caller hits `GET /atcs/search?query=login`
- **Then**: API returns 401

#### Scenario G2.1: Should cap results at 50 when limit exceeds max (Boundary, Medium)
- **Given**: Workspace has 100+ ATCs matching the query
- **When**: User calls `GET /atcs/search?query=login&limit=100`
- **Then**: Response contains at most 50 items

#### Scenario G2.2: Should return 20 results by default when limit is not specified (Boundary, Low)
- **Given**: Workspace has 50+ ATCs matching the query
- **When**: User calls `GET /atcs/search?query=login` (no limit param)
- **Then**: `items` array has exactly 20 entries

#### Scenario G3.1: Should filter results to specified layer only (Functional, Medium) — NEEDS PO/DEV CONFIRMATION
- **NEEDS PO/DEV CONFIRMATION**: layer filter not in original ACs despite being in the architect's API spec
- **Given**: Workspace has ATCs of layers UI, API, and Unit — all matching query "validate"
- **When**: User calls `GET /atcs/search?query=validate&layer=UI`
- **Then**: Only layer=UI ATCs appear in `items`

#### Scenario G6.1: Should return 200 with empty items list when query has no matches (Functional, Medium)
- **Given**: No ATCs in the workspace match "xyznotfound"
- **When**: User calls `GET /atcs/search?query=xyznotfound`
- **Then**: API returns 200 with `{items: []}`; NOT a 404

---

## Phase 4 — Test Outlines (DRAFT — outline names only)

### Coverage estimate

| Type | Count | Notes |
|------|-------|-------|
| Positive | 5 | Title match, tag match, multi-word, combined filter, recency ranking |
| Negative | 7 | Empty query, absent query, whitespace, no match, unauth, wrong scope, workspace isolation |
| Boundary | 4 | Min query length, limit default, limit cap, limit=0 |
| Integration | 3 | TSV trigger fires, module subtree recursive CTE, workspace scope at service layer |
| Security | 2 | Workspace injection attempt, SQL injection via query param |
| **Total** | **21** | HIGH complexity — ranking + security boundary + FTS semantics each require dedicated coverage |

**Rationale**: This Story combines a non-trivial ranking algorithm (text relevance × recency decay), a recursive data structure (module subtree), and a security perimeter (workspace isolation). Each of these three axes independently drives a test cluster. The security cluster alone requires 4–5 scenarios. The integration cluster requires trigger + CTE verification that cannot be inferred from the API surface alone — DB cross-validation is mandatory.

### Outline list (NAMES ONLY)

#### Positive
- **Should return matching ATC when query exactly matches a word in its title** — Pre: ATC "Login with expired token" exists in workspace. Expected: ATC appears in items, all response fields populated.
- **Should return matching ATC when query exactly matches one of its tags** — Pre: ATC tagged "smoke" exists. Expected: ATC appears; tag match confirmed (weight B).
- **Should return all ATCs matching a multi-word query (AND semantics)** — Pre: ATCs with "login" AND "token" in title/tags. Expected: only ATCs containing both tokens; FTS AND behavior documented.
- **Should include ATCs from selected module AND all recursive sub-modules when module filter applied** — Pre: module hierarchy with 3 levels. Expected: ATCs from all 3 levels appear; sibling module ATCs excluded. [**NEEDS PO/DEV CONFIRMATION** on recursive behavior]
- **Should rank more recently updated ATC above equally text-relevant older ATC** — Pre: two ATCs, identical FTS tokens, different updated_at. Expected: newer ATC first.

#### Negative
- **Should return 200 with empty items list when query matches no ATCs** — Pre: query with unique string not in any ATC. Expected: `{items: []}`, 200.
- **Should return 400 when query param is empty string** — Pre: `?query=`. Expected: 400 or empty — [**NEEDS PO/DEV CONFIRMATION**]
- **Should return 400 when query param is absent entirely** — Pre: no `?query=` at all. Expected: 400 `validation_failed`. [**NEEDS PO/DEV CONFIRMATION**]
- **Should return 400 or empty when query is whitespace only** — Pre: `?query=%20`. Expected: no search executed. [**NEEDS PO/DEV CONFIRMATION**]
- **Should return 401 when called without authentication** — Pre: no auth header/cookie. Expected: 401. [**NEEDS PO/DEV CONFIRMATION** on auth model]
- **Should not return ATCs from other workspaces even when title matches** — Pre: identical ATC title in two workspaces. Expected: only caller's workspace ATC in response.
- **Should ignore workspace_id query param injection and scope to session workspace** — Pre: valid ATC in W2; caller is W1 and passes `?workspace_id=<W2>`. Expected: W2 ATC not in results. [**NEEDS PO/DEV CONFIRMATION** on param acceptance]

#### Boundary
- **Should accept single-character query as valid minimum input** — Pre: `?query=l`. Expected: 200, results may be empty or not, no validation error.
- **Should return default 20 results when limit param is not specified** — Pre: 50+ matching ATCs. Expected: exactly 20 items.
- **Should cap results at 50 when limit=100 is requested** — Pre: 100+ matching ATCs. Expected: at most 50 items.
- **Should return 400 or default when limit=0 is requested** — Pre: valid query, limit=0. Expected: 400 or default 20. [**NEEDS PO/DEV CONFIRMATION**]

#### Integration
- **Should reflect updated title in search results after trigger fires on title change** — Pre: ATC exists, title updated via PATCH. Expected: `search_tsv` updated; new title is searchable; old title no longer matches if not present.
- **Should apply module subtree filter using recursive CTE correctly at DB level** — Pre: 3-level module hierarchy. Expected: verify via DB query that only subtree module_ids are included.
- **Should apply workspace scoping at service layer independently of RLS** — Pre: two tenants, same Supabase instance. Expected: `workspace_id = $session.workspace_id` clause present in query regardless of RLS policy state.

#### Security
- **Should not leak atc_id or content from another workspace via query param injection** — Pre: valid ATC in W2, caller from W1 attempts `workspace_id=<W2>`. Expected: W2 data absent, no server error.
- **Should safely handle query param containing SQL injection patterns** — Pre: `?query='; DROP TABLE atcs;--`. Expected: 200 or empty results; no DB error; parameterized query confirmed.

---

## Phase 5 — Edge Cases (DRAFT)

| # | Edge case | In original Story? | Criticality | Action |
|---|-----------|-------------------|-------------|--------|
| E1 | Partial-word query ("expir" instead of "expired") — FTS `plainto_tsquery` is exact-word only | No | **Critical** | **NEEDS PO/DEV CONFIRMATION** — MVP supports prefix matching or exact-word only? Determines the entire autocomplete UX |
| E2 | Multi-word query ("expired token") — FTS AND semantics | No | High | Add as positive test case — document AND behavior |
| E3 | `module_id` for a non-existent module UUID | No | High | **NEEDS PO/DEV CONFIRMATION** — 404 or empty results? |
| E4 | `layer` param with invalid enum value ("Mobile") | No | Medium | **NEEDS PO/DEV CONFIRMATION** — 400 or silently ignored? |
| E5 | Workspace with 0 ATCs — any query returns empty | No | Low | Test only — 200 `{items:[]}` |
| E6 | Workspace with 10,000+ ATCs — p95 response time < 100ms | No | High | Performance outline — verify GIN index present before load |
| E7 | ATC with `updated_at = null` (if column allows nulls) | No | High | **NEEDS PO/DEV CONFIRMATION** — confirm NOT NULL constraint on `updated_at`. Null breaks ranking formula. |
| E8 | Two ATCs with identical `updated_at` timestamp — tie in recency | No | Low | Non-deterministic order expected; test should not assert relative order for timestamp ties |
| E9 | ATC with 10 tags (max) — all tags indexed in tsvector | No | Medium | Test only — verify all 10 tags are searchable |
| E10 | Query matching only tag of ATC in another workspace | No | Critical | Extends AC6 — workspace isolation must hold for tag matches, not just title |
| E11 | `limit=51` (one over cap) vs `limit=50` (at cap) | No | Medium | Boundary test — cap enforcement must be ≥50, not >50 |
| E12 | Response `module_path` for a top-level module (no parent) | No | Low | Expected: just the module name with no "/" separator |

---

## Story Quality Assessment

**Verdict**: Needs Improvement

**Key findings**:
- **Critical ambiguity (A1/E1)**: The FTS implementation (`plainto_tsquery`) does not support prefix matching, but the Workflow describes an autocomplete UX ("suggestions refine as she types"). This is a contradiction that must be resolved before Dev starts. If prefix search is required, the SQL needs to change; if exact-word is acceptable, the UX description needs correction.
- **Security gap (G1)**: No AC covers authentication. `GET /atcs/search` derives `workspace_id` from the caller's session — if there is no session, the endpoint either panics or returns no results. A 401 scenario must be added.
- **Missing functional ACs (G2, G3)**: The `limit` boundary behavior and `layer` filter are documented in the business rules and architect note but have no corresponding ACs. Without them, Dev has no contract to implement against.

---

## Critical Questions for PO

> These BLOCK sprint planning until answered.

1. **FTS match semantics: exact-word or prefix (autocomplete)?**
   - **Context**: `plainto_tsquery('simple', $1)` matches whole tokens only. "expir" does NOT match "expired". The Workflow section says suggestions refine as she types — implying prefix behavior.
   - **Impact if unanswered**: Dev implements exact-word FTS; UX team builds an autocomplete that doesn't work until the user types a complete word. Misaligned expectations cause rework.
   - **Suggested answer**: If prefix search is required for MVP, use `to_tsquery('simple', $1 || ':*')` for single-token queries (and `plainto_tsquery` for multi-word). Document the hybrid approach.

2. **Authentication model for GET /atcs/search**
   - **Context**: The endpoint needs a session to derive `workspace_id`. No auth AC exists.
   - **Impact if unanswered**: Security test gap; potential 500 on unauthenticated calls.
   - **Suggested answer**: `requireAuth()` (accepts session cookie OR Bearer PAT) — consistent with the pattern used in other GET routes.

3. **Layer filter: is it IN scope for BK-20 or a separate Story?**
   - **Context**: Architect spec includes `layer` as an optional filter. No AC covers it. Scope doc does not list it.
   - **Impact if unanswered**: Dev may implement it (following architect spec) but QA has no AC to verify against. Or Dev skips it and the picker has no layer filter at all.
   - **Suggested answer**: Confirm IN or OUT of BK-20. If IN, add AC. If OUT, create a follow-up Story.

---

## Technical Questions for Dev

> These do not block PO but block implementation.

1. **`search_tsv` column: `GENERATED ALWAYS AS STORED` or trigger-maintained?** The architect note says "decided at impl plan". The trigger approach fires on INSERT/UPDATE, which is correct for MVP. Generated column requires an immutable function — `array_to_string(tags, ' ')` may not qualify. Confirm choice so QA can write the integration outline (trigger fires vs column is computed).

2. **`module_id` filter: what happens when the UUID is valid but the module does not belong to the caller's workspace?** Should the module subtree CTE silently return empty (module not found → no subtree → no results) or return a 404? Empty is safer and avoids information leakage, but needs an explicit decision.

3. **`updated_at` null safety**: Confirm `updated_at` has a NOT NULL constraint (or defaults to `now()` on insert). A null `updated_at` causes EXTRACT to return null, which causes the ranking score to be null, which sorts unpredictably.

4. **`status_dot` field in response**: What values can this take? Is it the ATC's `status` column from the `atcs` table? What are the valid values? This field is in the architect spec but undocumented in ACs or business rules.

5. **Empty/absent/whitespace query handling**: Confirm the exact 400 response shape for each: empty string, absent param, whitespace-only. Should all return the same `validation_failed` code or differ?

---

## Suggested Story Improvements

| # | Current state | Suggested change | Benefit |
|---|---------------|------------------|---------|
| 1 | AC4: "two equally relevant ATCs" | "When two ATCs produce identical text-relevance scores, the one updated more recently ranks higher" | Removes ambiguity about what "equally relevant" means; makes test data setup deterministic |
| 2 | No AC for auth | Add: "Unauthenticated request returns 401" | Closes security gap; aligns with all other protected endpoints |
| 3 | No AC for limit behavior | Add: "Limit defaults to 20 when not specified; limit is capped at 50" | Closes contract gap; prevents silent unbounded queries |
| 4 | Workflow says "refines as she types" | Qualify: "refines as she types complete words" OR implement prefix FTS | Prevents UX misalignment with FTS implementation |
| 5 | `status_dot` undefined | Define `status_dot` values in business rules | QA can assert response shape; Frontend can render the field |

---

## Data feasibility flags

DATA-FEASIBILITY-RISK was raised in Phase 1 Selection and is now **cleared** based on BK-18 context:

- **BK-18 Shift-Left QA completed 2026-05-27**: schema for `atcs` table is contractually defined. Columns `title`, `tags`, `updated_at`, `module_id`, `workspace_id`, `id`, `slug`, `layer` are confirmed in the BK-18 ATP DRAFT.
- **BK-20 adds its own migration**: `search_tsv tsvector` column + `atcs_search_tsv_idx` GIN index + `atcs_tsv_trg` trigger. This migration is scoped to BK-20, not BK-18.
- **Remaining pre-condition**: Wave 1 `module_paths` mechanism (recursive CTE or materialized view) must exist when BK-20 is implemented. QA should verify this during sprint planning if BK-20 starts before Wave 1 module work is complete.
- **`test_steps` table**: NOT YET present (EPIC-BK-5). BK-20 search does not depend on it — confirmed out of scope.

---

## Recommended testing strategy

### Pre-implementation (NOW — Shift-Left)
- Resolve Critical PO Questions (especially FTS semantics and auth model) before Dev starts
- Confirm `layer` filter scope (IN or OUT of BK-20)
- Confirm `status_dot` field values
- Verify Wave 1 module subtree mechanism is available when BK-20 enters sprint

### During implementation
- Dev confirms `search_tsv` approach (generated column vs trigger) — QA updates the integration outline accordingly
- Dev registers the new endpoint in OpenAPI spec; QA validates the spec against the architect's documented shape
- QA reviews the DB migration for: NOT NULL on `updated_at`, correct tsvector weights (A for title, B for tags), GIN index creation

### Post-implementation (in-sprint by /sprint-testing)
- Smoke: `GET /atcs/search?query=login` returns 200 with valid items — confirms endpoint is live and auth is wired
- UI exploration: verify `module_path` format (`"Module A / Submodule B"`) in actual response
- API exploration: verify ranking (recency + relevance) with controlled test data
- DB cross-validation: confirm `workspace_id` filter is in the SQL query (defense in depth), verify GIN index is used (EXPLAIN ANALYZE)
- Security: workspace isolation test with two tenants

---

## Risks & mitigation

| # | Risk | Likelihood | Impact | Mitigated by which outlines |
|---|------|-----------|--------|-----------------------------|
| 1 | FTS semantics mismatch (exact-word vs prefix) breaks autocomplete UX | Medium | High | Positive-1, Negative-3 (document behavior explicitly) |
| 2 | Workspace isolation breach via query param injection | Low | Critical | Security-1, Security-2 (mandatory regardless of risk) |
| 3 | Ranking formula breaks on null `updated_at` | Low | High | E7 (confirm constraint before sprint) |
| 4 | Module subtree filter returns flat results instead of recursive | Medium | Medium | Positive-4, Integration-2 (recursive CTE must be verified at DB level) |
| 5 | `search_tsv` trigger not fired after PATCH — stale search index | Low | Medium | Integration-1 (trigger firing test after title update) |
| 6 | Performance degradation on large workspaces (>10k ATCs) | Low | High | E6 (performance outline — GIN index must be present) |

---

## Next steps

- [ ] PO answers Critical Questions (1, 2, 3) before sprint planning
- [ ] Dev answers Technical Questions (1–5) before estimation
- [ ] Story enters sprint at status `Ready For Dev` once estimated
- [ ] When Story reaches `Ready For QA`, `/sprint-testing` will short-circuit refinement (label `shift-left-reviewed` detected) and proceed to Stage 2 execution
