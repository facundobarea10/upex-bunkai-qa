# Domain Glossary — Bunkai TMS

> **Discovery date**: 2026-05-31  
> **Sources**: `lib/types.ts`, `app/qa/qa-config.ts`, `app/(auth)/login/page.tsx`, `supabase/migrations/`, `DESIGN.md`  
> **Rule**: UI label ↔ code identifier mapping is listed explicitly. When in doubt, use the code identifier in tests and the UI label in user-facing docs.

---

## Methodology Terms

### IQL — Integrated Quality Lifecycle
The methodology Bunkai is built around. Maps the complete quality chain:

```
UserStory → AcceptanceCriterion → ATC → Run → Bug
```

Every entity in Bunkai maps to exactly one IQL stage. No freeform steps — quality is continuous from requirement to release.  
Source: `app/(auth)/login/page.tsx` FEATURE_TICKS[0], `app/qa/qa-config.ts`

### KATA — Komponent Action Test Architecture
The architecture that describes how ATCs assemble into automated tests. ATCs are the reusable "katas"; KATA defines how they compose into test suites without copy-paste.  
Source: `app/(auth)/login/page.tsx` FEATURE_TICKS[2]

### ATC — Acceptance Test Case
The lowest executable unit in Bunkai. An ATC maps to one or more AcceptanceCriteria and is:
- Layer-aware (UI / API / Unit)
- Executable by humans, AI agents, or CI
- Composed of ordered Steps and Assertions

UI label: **ATC**  
Code identifier: `Atc` / `atc`  
ID format: slug (e.g. `my-feature-happy-path`); visual display: `ATC-001` style  
Source: `lib/types.ts` `Atc` interface, `DESIGN.md` §Chips

---

## Core Entities

### Workspace
Top-level tenant container. All data in Bunkai is scoped to a Workspace. Users can belong to multiple Workspaces with different roles.

| Field | Type | Description |
|---|---|---|
| `id` | Uuid | Primary key |
| `slug` | string | URL-safe identifier (3–40 chars, `^[a-z0-9][a-z0-9-]{1,38}[a-z0-9]$`) |
| `name` | string | Display name |
| `owner_user_id` | Uuid | FK → auth.users |
| `plan` | WorkspacePlan | Tier: community / cloud / enterprise |
| `created_at` | Timestamp | ISO 8601 |

Source: `lib/types.ts`, `supabase/migrations/0001_tenancy.sql`

---

### WorkspaceMember
Join table that represents a user's membership in a Workspace, including their role and status.

| Field | Type | Description |
|---|---|---|
| `workspace_id` | Uuid | FK → workspaces |
| `user_id` | Uuid | FK → auth.users |
| `role` | MemberRole | viewer / member / admin / owner |
| `status` | MemberStatus | active / invited / suspended |
| `joined_at` | Timestamp | When they joined |

Source: `lib/types.ts`, `supabase/migrations/0001_tenancy.sql`

---

### Project
Test scope within a Workspace. Organizes Modules, UserStories, and ATCs. Has a URL-safe slug unique within the workspace.

| Field | Type | Description |
|---|---|---|
| `id` | Uuid | Primary key |
| `workspace_id` | Uuid | FK → workspaces |
| `slug` | string | URL identifier (unique within workspace) |
| `name` | string | Display name |
| `description` | string? | Optional description |
| `created_at` | Timestamp | ISO 8601 |

Source: `lib/types.ts`, `supabase/migrations/0002_projects_modules.sql`

---

### Module
Hierarchical folder that organizes UserStories and ATCs within a Project. Supports nesting up to 6 levels deep via a materialized path.

| Field | Type | Description |
|---|---|---|
| `id` | Uuid | Primary key |
| `project_id` | Uuid | FK → projects |
| `parent_module_id` | Uuid? | Self-referential FK (nullable = root module) |
| `path` | string | Slash-separated materialized path (e.g. `Feature/Login/UI`) |
| `name` | string | Display name |
| `position` | number | Sort order among siblings |
| `created_at` | Timestamp | ISO 8601 |

**Constraint**: max depth 6 (`array_length(string_to_array(path, '/')) BETWEEN 1 AND 6`).  
Source: `lib/types.ts`, `supabase/migrations/0002_projects_modules.sql`

---

### UserStory
A feature or requirement being tested. Corresponds to a Jira story or equivalent issue tracker item. Contains AcceptanceCriteria and anchors ATCs.

| Field | Type | Description |
|---|---|---|
| `id` | Uuid | Primary key |
| `module_id` | Uuid | FK → modules |
| `title` | string | Story title |
| `description` | string? | Optional description |
| `external_id` | string? | Jira issue key (e.g. `BK-42`) |
| `external_url` | string? | Link to Jira/external tracker |
| `created_at` | Timestamp | ISO 8601 |

Source: `lib/types.ts`, `supabase/migrations/0003_authoring.sql`

---

### AcceptanceCriterion (AC)
A single acceptance criterion within a UserStory. ATCs must be anchored to at least one AC (the "anchoring moat"). Ordered by position.

UI label: **AC** / **Acceptance Criterion**  
Code identifier: `AcceptanceCriterion` / `acceptance_criterion`

| Field | Type | Description |
|---|---|---|
| `id` | Uuid | Primary key |
| `user_story_id` | Uuid | FK → user_stories |
| `title` | string | AC text |
| `description` | string? | Optional extended description |
| `position` | number | Order within story |
| `created_at` | Timestamp | ISO 8601 |

Source: `lib/types.ts`, `supabase/migrations/0003_authoring.sql`

---

### ATC (Acceptance Test Case)
The primary executable entity in Bunkai. Maps to one or more AcceptanceCriteria and is composed of ordered Steps and Assertions. Must always be anchored to a UserStory.

UI label: **ATC**  
Code identifier: `Atc` / `atc`

| Field | Type | Description |
|---|---|---|
| `id` | Uuid | Primary key |
| `project_id` | Uuid | FK → projects |
| `module_id` | Uuid | FK → modules |
| `user_story_id` | Uuid | FK → user_stories (RESTRICT — must have story) |
| `slug` | string | URL identifier (unique within project) |
| `title` | string | ATC title |
| `layer` | AtcLayer | UI / API / Unit |
| `version` | number | Optimistic lock version counter |
| `status` | AtcStatus | Execution state (see enumerations) |
| `tags` | string[] | Searchable tags array |
| `created_at` | Timestamp | ISO 8601 |
| `updated_at` | Timestamp | Auto-updated by DB trigger |

Source: `lib/types.ts`, `supabase/migrations/0004_atcs.sql`

---

### AtcStep
An individual test step within an ATC. Ordered by position. Describes what to do, what input to use, and what to expect.

| Field | Type | Description |
|---|---|---|
| `id` | Uuid | Primary key |
| `atc_id` | Uuid | FK → atcs |
| `position` | number | Order within ATC |
| `content` | string | Step description (what to do) |
| `input_data` | string? | Test data / payload |
| `expected` | string? | Expected outcome |

Source: `lib/types.ts`, `supabase/migrations/0004_atcs.sql`

---

### AtcAssertion
A verification statement within an ATC. Ordered by position.

| Field | Type | Description |
|---|---|---|
| `id` | Uuid | Primary key |
| `atc_id` | Uuid | FK → atcs |
| `position` | number | Order within ATC |
| `content` | string | Assertion description |

Source: `lib/types.ts`, `supabase/migrations/0004_atcs.sql`

---

### AtcAcceptanceCriterion
Many-to-many join table linking ATCs to AcceptanceCriteria. An ATC must be linked to at least one AC (enforced at app layer).

| Field | Type | Description |
|---|---|---|
| `atc_id` | Uuid | FK → atcs |
| `acceptance_criterion_id` | Uuid | FK → acceptance_criteria |

Source: `lib/types.ts`, `supabase/migrations/0004_atcs.sql`

---

### ModuleTreeNode
UI-only composite type. A Module hydrated with its children (recursively), user_stories, and atcs for the sidebar tree view.

Source: `lib/types.ts` — not persisted, computed at query time via `lib/tree.ts`

---

### UserStoryWithChildren
UI-only composite type. A UserStory hydrated with its acceptance_criteria and atcs.

Source: `lib/types.ts` — not persisted, computed at query time

---

## Enumerations

### AtcStatus
Execution state of an ATC. Only one status at a time. Color-coded in the UI.

| Value | Color | Hex | Meaning |
|---|---|---|---|
| `unrun` | neutral | — | Default; never executed |
| `running` | blue (pulsing) | `#4f8cf7` | Currently executing |
| `pass` | green | `#2fb673` | Executed and passed |
| `fail` | red | `#e5484d` | Executed and failed |
| `blocked` | amber | `#e8a838` | Cannot execute (dependency/blocker) |
| `skipped` | gray | `#8a91a0` | Intentionally skipped |

Source: `lib/types.ts` `AtcStatus`, `DESIGN.md` §Signal palette

---

### AtcLayer
Testing scope. Determines which part of the stack an ATC tests.

| Value | Color token | Description |
|---|---|---|
| `UI` | `--layer-ui` (purple) | User interface / browser-level test |
| `API` | `--layer-api` (blue) | HTTP API / service-level test |
| `Unit` | `--layer-unit` (green) | Code unit / function-level test |

Source: `lib/types.ts` `AtcLayer`, `DESIGN.md` §Layer chips

---

### MemberRole
User permission level within a Workspace. Hierarchical.

| Value | Permissions |
|---|---|
| `owner` | Full control; workspace creator |
| `admin` | Manage members, invites, metadata |
| `member` | Create/edit ATCs, stories, modules |
| `viewer` | Read-only access |

Source: `lib/types.ts` `MemberRole`, `supabase/migrations/0001_tenancy.sql`

---

### MemberStatus
Lifecycle state of a WorkspaceMember.

| Value | Meaning |
|---|---|
| `active` | Member has joined and can access workspace |
| `invited` | Invitation sent; pending acceptance |
| `suspended` | Access revoked |

Source: `lib/types.ts` `MemberStatus`, `supabase/migrations/0010_workspace_invites.sql`

---

### WorkspacePlan
Workspace tier / billing level.

| Value | Deployment | Description |
|---|---|---|
| `community` | Self-hosted | OSS, Docker Compose |
| `cloud` | SaaS | Managed cloud (Vercel + Supabase) |
| `enterprise` | Custom | Custom deployment |

Source: `lib/types.ts` `WorkspacePlan`, `supabase/migrations/0001_tenancy.sql`

---

## PAT Scopes (Personal Access Token)

Scopes granted to Bearer tokens used by CLI tools and AI agents.

| Scope | What it allows |
|---|---|
| `atc:read` | Read ATCs, steps, assertions, modules, user stories, ACs |
| `atc:write` | Create / update / delete ATCs |
| `run:execute` | Start runs, post step results |
| `workspace:admin` | Manage members, invites, workspace metadata |

Source: `supabase/migrations/0008_access_tokens.sql`, `app/qa/qa-config.ts`

---

## UI/UX Vocabulary

| UI Term | Code Term | Description |
|---|---|---|
| Chip | — | Small badge displaying status or layer |
| Status chip | — | Chip for `AtcStatus` (pass/fail/blocked/skipped/running) |
| Layer chip | — | Chip for `AtcLayer` (UI/API/Unit) |
| Command palette | `CommandPalette` | Cmd+K quick action/search overlay |
| Module tree | `ModuleTreeNode[]` | Hierarchical sidebar navigator |
| Workspace switcher | `WorkspaceSwitcher` | Dropdown to switch active workspace |
| Topbar | `Topbar` | Header with breadcrumbs + workspace/project context |
| ATC editor | `AtcEditor` | Full ATC authoring component (steps, assertions, anchoring) |
| Anchoring panel | `AnchoringPanel` | Picker for linking ATC to AcceptanceCriteria |

Source: `DESIGN.md`, `components/` directory

---

## Typography Conventions

| Context | Font | Token |
|---|---|---|
| IDs (`ATC-001`, `RUN-451`, `MOD-014`) | JetBrains Mono | `font-mono` |
| Body text / prose | Inter | `font-sans` |
| Brand wordmark (kanji) | Noto Serif JP | — |

Source: `DESIGN.md` §Design principles, `app/layout.tsx` font imports

---

## Discovery Gaps

- [ ] **Run entity**: `AtcStatus` has a `running` state and the login page mentions "Run" as an IQL stage, but no `runs` or `run_results` tables were found in migrations. The full Run execution model is deferred to a future phase.
- [ ] **Bug entity**: IQL mentions "Story → Case → Run → Bug" but no `bugs` table exists in the current schema. Bugs likely tracked in external Jira (via `external_id` on UserStory).
- [ ] **Module naming conventions**: No documented convention for Module path segments (e.g. whether to use kebab-case or Title Case for path components).
