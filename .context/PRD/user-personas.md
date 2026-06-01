# PRD — User Personas

> **Discovery date**: 2026-05-31  
> **Method**: Reverse-engineered from role enums, page components, DESIGN.md, and login page copy  
> **Source files**: `lib/types.ts` (MemberRole, MemberStatus), `DESIGN.md`, `app/(auth)/login/page.tsx`, `app/(app)/workspaces/[id]/members/members-client.tsx`

---

## Role Hierarchy

```
owner > admin > member > viewer
```

| Role | Permissions |
|---|---|
| `owner` | Full control; workspace creator; cannot be removed |
| `admin` | Manage members and invites; edit workspace metadata |
| `member` | Create/edit ATCs, steps, assertions, modules, user stories |
| `viewer` | Read-only access to all workspace data |

Source: `lib/types.ts` `MemberRole`, `supabase/migrations/0001_tenancy.sql` RLS policies

---

## Persona 1 — QA Lead / Test Manager

**Archetype**: Sarah  
**Role in Bunkai**: `owner` or `admin`  
**Primary screen time**: 2–3 hours/day

### Profile
Leads QA for a product team. Responsible for test strategy, coverage quality, and team coordination. Manages 200–500 ATCs across 5–10 modules. Reports to Product or Engineering.

### Goals
- Maintain ATC coverage aligned with acceptance criteria across all modules
- Onboard and manage QA team members (invite, role assignment, revoke)
- Navigate large ATC inventories quickly without losing context
- Ensure every user story has at least one ATC bound to each AC

### Pain Points
- Spreadsheet-based test management: no traceability, no status tracking
- Team members working on overlapping or duplicate ATCs
- Manually tracking who has access to what test assets
- Context switching between test tool and issue tracker

### Primary Workflows
1. **Workspace creation** → onboarding form → create first project
2. **Team management** → invite teammates, assign roles, rotate/revoke links
3. **ATC inventory review** → browse module tree, scan ATC table, check coverage
4. **Command palette navigation** → Cmd+K for fast jumping between modules/ATCs

### Bunkai Features Used
- Workspace onboarding form
- Members page (invite, role select, revoke, rotate)
- Project tree sidebar (module navigation)
- ATC table (list view, all ATCs with module_path)
- Workspace switcher (multi-project oversight)

---

## Persona 2 — Test Automation Engineer / SDET

**Archetype**: James  
**Role in Bunkai**: `member`  
**Primary screen time**: 4+ hours/day

### Profile
Writes and maintains automated test cases. Deep familiarity with the system under test. Uses Bunkai as the source of truth for ATC definitions, then implements them in code (Playwright/API tests). Cares about IDs, layers (UI/API/Unit), step precision, and AC binding correctness.

### Goals
- Write precise ATCs with clear steps (content, input_data, expected) and assertions
- Bind ATCs to the right acceptance criteria (anchoring moat)
- Identify ATCs that need automation (unrun, fail, blocked)
- Work fast with keyboard shortcuts and command palette — no mouse dependency

### Pain Points
- Ambiguous test steps with no expected output defined
- ATCs not linked to user stories — can't tell what requirement they cover
- Switching between editor and list view breaks focus
- Monolithic test scripts that can't be reused across stories

### Primary Workflows
1. **ATC editing** → navigate tree → open ATC → edit steps/assertions → bind to ACs → save
2. **New ATC creation** → "N" shortcut or "New ATC" button → editor pre-filled with module context
3. **Layer selection** → choose UI / API / Unit per ATC
4. **Coverage check** → filter table by module/story, scan AC bindings

### Bunkai Features Used
- ATC editor (steps, assertions, anchoring panel)
- Layer selector (UI/API/Unit chip)
- AC binding picker (story grouping + AC list)
- Keyboard shortcut "N" for new ATC
- Command palette for fast navigation
- Module tree sidebar for context

---

## Persona 3 — QA Analyst / Manual Tester

**Archetype**: Priya  
**Role in Bunkai**: `viewer` or `member`  
**Primary screen time**: 1–2 hours/day

### Profile
Executes manual tests during sprint QA cycles. Reads ATCs to understand test intent, follows steps, records results. Less technical than James; needs clear, readable descriptions over dense code-like notation.

### Goals
- Read ATC steps and understand what to do and what to expect
- Track which tests map to which user story
- See test status (pass/fail/blocked) to coordinate with the team
- Join the workspace when invited without friction

### Pain Points
- Overcrowded test management UIs with too many buttons/options
- ATCs with missing `expected` outcomes — can't determine pass/fail
- No clarity on which test covers which requirement
- Difficult onboarding (creating accounts, figuring out where to start)

### Primary Workflows
1. **Invite acceptance** → click link → join workspace → land on /projects
2. **Module browsing** → sidebar tree → open story → read its ATCs
3. **ATC detail reading** → open ATC, read steps + expected outcomes
4. **Status tracking** → ATC table status chips (pass/fail/blocked/skipped)

### Bunkai Features Used
- Invite acceptance flow (link-based, no account creation friction if magic-link)
- Project tree sidebar (read-only browsing)
- ATC table (status chips, module_path column)
- ATC detail view (steps, assertions, bound ACs)

---

## Persona 4 — AI Agent / CI System

**Archetype**: Automated executor (Claude/OpenCode, GitHub Actions)  
**Role in Bunkai**: Headless consumer via Bearer PAT  
**Interaction**: API-only (no browser)

### Profile
A non-human actor that queries ATCs, posts execution results, and manages runs. Uses Personal Access Tokens (PATs) with explicit scopes. The same ATC schema that humans author is what the agent executes against.

### Goals
- Fetch ATCs for a project/module via API (`atc:read`)
- Execute test steps and post results (`run:execute`)
- Update ATC status post-run (`atc:write`)
- Operate without a browser session (Bearer token only)

### Pain Points
- Scope creep: agent should only access what its PAT permits
- Token expiry handling: need refresh or long-lived PAT
- Rate limits and idempotency: duplicate run posts must not corrupt results

### Primary Workflows
1. `POST /api/v1/auth/signin` → get session + mint PAT in one call
2. `GET /api/v1/workspaces` → resolve workspace_id
3. `GET /api/v1/projects?workspace_id=...` → resolve project_id
4. Fetch ATCs via project → execute → post results
5. `DELETE /api/v1/tokens/{id}` → revoke PAT when done

### Bunkai Features Used
- `/api/v1/auth/signin` (headless auth)
- `/api/v1/tokens` (PAT lifecycle)
- `/api/v1/workspaces` (workspace discovery)
- All ATC-related endpoints (read/write per scopes)
- Idempotency keys (POST replay protection)

---

## Discovery Gaps

- [ ] **Viewer role UI gating**: Source code shows role enum but no explicit in-component role checks on edit actions were found — assumes server-side RLS handles it. Confirm no client-side edit buttons visible to viewers.
- [ ] **Guest / unauthenticated user**: No public-facing project view found. All `/projects/*` routes require auth. Confirm if there's a planned public share-link feature.
- [ ] **Enterprise admin persona**: `workspace:admin` PAT scope exists; enterprise plan mentioned in schema. No enterprise-specific UI features found in Phase 1.
