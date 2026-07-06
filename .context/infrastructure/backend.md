# Infrastructure — Backend

> **Discovery date**: 2026-05-31  
> **Sources**: `next.config.ts`, `tsconfig.json`, `package.json`, `lib/supabase/`, `supabase/migrations/`, `middleware.ts`, `.env.example`

---

## Runtime

| Property | Value |
|---|---|
| **Language** | TypeScript 5.8 |
| **Runtime** | Bun ≥ 1.0.0 |
| **Framework** | Next.js 15 (App Router) |
| **API style** | REST — `/api/v1/` route handlers (Next.js Route Handlers) |
| **Database** | Supabase PostgreSQL 16 (managed cloud) |
| **ORM** | None — raw SQL via Supabase JS client |
| **Auth** | Supabase Auth (magic-link OTP + Bearer PAT, dual-mode) |

---

## Build Configuration

```bash
# Install
bun install

# Development (hot reload, port 3000)
bun run dev              # → next dev

# Production build
bun run build            # → next build
bun run start            # → next start

# Type check (CI gate)
bun run typecheck        # → tsc --noEmit

# Generate Supabase TypeScript types from live schema
bun run types:gen        # → bun scripts/gen-supabase-types.ts

# Sync OpenAPI spec → TypeScript types
bun run api:sync         # → bun scripts/sync-openapi.ts

# Full repo gate (format + lint + types + vars + skills)
bun run repo:check
bun run repo:fix         # auto-fix where possible
```

---

## Key Path Aliases (`tsconfig.json`)

| Alias | Resolves to | Usage |
|---|---|---|
| `@/*` | `./` | Root-level imports |
| `@app/*` | `./app/` | App Router pages + layouts |
| `@components/*` | `./components/` | Shared UI components |
| `@lib/*` | `./lib/` | Utilities, helpers, types |

---

## Environment Variables (Required)

From `.env.example` — copy to `.env` and fill before running.

```bash
# Supabase (REQUIRED — app won't start without these)
NEXT_PUBLIC_SUPABASE_URL=https://fmbpikzpkafptqximhxn.supabase.co
SUPABASE_PUBLISHABLE_KEY=          # anon/public key
SUPABASE_SECRET_KEY=               # service_role key (server-only)
SUPABASE_JWT_SECRET=               # custom JWT signing

# App URL (REQUIRED — auth redirects won't work without this)
NEXT_PUBLIC_APP_URL=http://localhost:3000    # dev
# NEXT_PUBLIC_APP_URL=https://upexbunkai.vercel.app  # prod

# PostgreSQL direct (optional — only if using Prisma or raw SQL outside Supabase client)
POSTGRES_HOST=db.fmbpikzpkafptqximhxn.supabase.co
POSTGRES_USER=postgres
POSTGRES_PASSWORD=
POSTGRES_DATABASE=postgres
POSTGRES_URL=                      # pooled (port 6543)
POSTGRES_URL_NON_POOLING=          # direct (port 5432)
POSTGRES_PRISMA_URL=               # pooled + pgbouncer=true

# Integrations (optional)
RESEND_API_KEY=                    # transactional email
SUPABASE_ACCESS_TOKEN=             # Supabase MCP admin
ATLASSIAN_URL=                     # Jira instance
ATLASSIAN_EMAIL=
ATLASSIAN_API_TOKEN=
TAVILY_API_KEY=                    # web search MCP
N8N_API_URL=
N8N_API_KEY=
```

**Loading**: Use `bun run claude` or `bun run opencode` (wraps CLI with `dotenv-cli`). For shell: `direnv` via `.envrc`.

---

## Auth Flow

### Browser — Magic-Link OTP
```
POST /api/v1/auth/magic-link { email, next }
  → supabase.auth.signInWithOtp({ email })
  → Supabase sends OTP email

User clicks link → GET /auth/callback?code=...&next=/projects
  → supabase.auth.exchangeCodeForSession(code)
  → Session cookie set (httpOnly)
  → Redirect to `next`
```

**Entry files**: `app/api/v1/auth/magic-link/route.ts`, `app/auth/callback/route.ts`

### CLI/Agent — Bearer PAT
```
POST /api/v1/auth/signin { email, password, pat_scopes? }
  → supabase.auth.signInWithPassword()
  → Mint PAT: access_tokens + access_token_secrets (SHA-256 hash)
  → Returns { session, pat: { token: "bk_pat_<prefix>.<secret>" } }

Subsequent calls:
  Authorization: Bearer bk_pat_<prefix>.<secret>
  → lib/api/auth.ts: requireAuth() → Bearer path
    → prefix lookup → hash verify → userId + scopes
```

**Entry files**: `app/api/v1/auth/signin/route.ts`, `lib/api/auth.ts`, `lib/api/pat.ts`

### Session Refresh (Middleware)
Every request to protected routes (`/projects/*`, `/onboarding/*`) goes through `middleware.ts`:
1. `createServerClient()` with request cookies
2. `supabase.auth.getUser()` — refreshes session if expired
3. Updated cookies written back to response
4. No user → redirect to `/login?next=[pathname]`

---

## Database

### Supabase Project
| Property | Value |
|---|---|
| Project ref | `fmbpikzpkafptqximhxn` |
| Cloud URL | `https://fmbpikzpkafptqximhxn.supabase.co` |
| DB engine | PostgreSQL 16 |
| Local dev config | `supabase/config.toml` — **NOT FOUND** (supabase init not run) |

### Client Initialization
| Context | File | Client type |
|---|---|---|
| Browser (RSC / Client components) | `lib/supabase/client.ts` | `createBrowserClient()` from `@supabase/ssr` |
| Server (Route Handlers, Server Actions) | `lib/supabase/server.ts` | `createServerClient()` with cookie management |
| Admin ops (service role) | `lib/supabase/admin.ts` | Service-role client (bypasses RLS) |
| RPC calls | `lib/supabase/rpc.ts` | Helper wrappers for stored procedures |

### Key Stored Procedures (RPC)
| Function | Purpose |
|---|---|
| `bunkai_bootstrap_workspace(slug, name)` | Atomic workspace + owner member creation (used by onboarding) |
| `bunkai_save_atc(atc_id, title, layer, tags, user_story_id, steps, assertions, ac_ids)` | Atomic ATC save: delete-then-insert steps/assertions/bindings |

### Migrations
12 migration files in `supabase/migrations/` — applied sequentially. See `.context/SRS/architecture.md` for full table list.

### RLS Security Model
All 18 tables have RLS. Four SECURITY DEFINER helper functions prevent recursion:
- `bunkai_is_workspace_member(ws_id)` — active membership check
- `bunkai_can_write_workspace(ws_id)` — role ≥ member
- `bunkai_is_workspace_admin(ws_id)` — role ∈ {admin, owner}
- `bunkai_is_workspace_owner(ws_id)` — role = owner

---

## API Surface

| Base | Format | Docs |
|---|---|---|
| `/api/v1/` | REST JSON | `/api/docs` (Scalar UI) |
| `/api/openapi` | OpenAPI 3.0 JSON | Runtime-generated via zod-to-openapi |

Full endpoint list: `.context/SRS/functional-specs.md` §FR-10

---

## Discovery Gaps

- [ ] **Supabase local dev**: No `supabase/config.toml` found. Local Supabase (`supabase start`) not initialized — tests against cloud project.
- [ ] **Migrations applied status**: No migration tracking visible in source (Supabase manages this internally).
- [ ] **No test framework**: Zero test runner dependencies in `package.json` — no unit or integration tests exist.
- [ ] **CI/CD secrets injection**: How Vercel receives `SUPABASE_SECRET_KEY` and other secrets is not documented in source.
