# Project Configuration — Bunkai TMS

> **Discovery date**: 2026-05-31  
> **Sources**: `.agents/project.yaml`, `package.json`, `.env.example`, `supabase/migrations/`, `middleware.ts`  
> **Boilerplate**: `bunkai-qa-engineer` (QA automation framework)  
> **Target**: `upex-bunkai-tms` (the application under test)

---

## Repositories

| Role | Path | Type |
|---|---|---|
| **Target app (under test)** | `C:\Users\facun\Desktop\UPEX GALAXY\upex-bunkai-tms` | Monorepo — Next.js 15 handles both FE and BE |
| **QA boilerplate (this repo)** | `C:\Users\facun\Desktop\UPEX GALAXY\bunkai-qa-engineer` | KATA-based test automation framework |

The target is a **monorepo**: a single Next.js 15 repo serves both the frontend (React components, app router pages) and the backend (API routes at `/api/v1/`). No separate BE repo.

---

## Tech Stack

### Frontend
| Technology | Version | Role |
|---|---|---|
| Next.js | 15 (App Router) | Full-stack framework |
| React | 19 | UI library |
| TypeScript | 5.8+ | Type safety |
| Tailwind CSS | 3.4 | Styling |
| shadcn/ui | latest | Component library (Radix UI primitives) |
| Monaco Editor | 4.7 | Code/text editing in ATC editor |
| TanStack Table | 8.21 | ATC list data table |
| Lucide React | 1.16 | Icon library |
| Sonner | 2.0 | Toast notifications |

### Backend
| Technology | Version | Role |
|---|---|---|
| Next.js API routes | 15 | REST API at `/api/v1/` |
| Supabase JS | 2.106 | DB client, auth, RLS |
| Supabase SSR | 0.10 | Server-side session management |
| Zod | 4.4 | Request validation |
| `@asteasolutions/zod-to-openapi` | 8.5 | OpenAPI spec generation from Zod schemas |

### Database
| Technology | Details |
|---|---|
| Engine | Supabase PostgreSQL 16 (managed) |
| Migrations | 12 migration files in `supabase/migrations/` |
| Auth | Supabase Auth (magic-link OTP + password) |
| ORM | None — raw SQL via Supabase client |
| RLS | Enabled on all tables; enforced via SECURITY DEFINER helper functions |

### Infrastructure
| Technology | Role |
|---|---|
| Bun | Package manager + runtime for scripts |
| Vercel | Frontend hosting + API edge functions |
| Supabase Cloud | Database + Auth hosting |

---

## Environments

| Environment | Web URL | API URL | DB Project Ref |
|---|---|---|---|
| **local** | `http://localhost:3000` | `http://localhost:3000/api` | `fmbpikzpkafptqximhxn` |
| **staging** | `https://staging-upexbunkai.vercel.app` | `https://staging-upexbunkai.vercel.app/api` | `fmbpikzpkafptqximhxn` (shared) |
| **production** | `https://upexbunkai.vercel.app` | `https://upexbunkai.vercel.app/api` | `fmbpikzpkafptqximhxn` (shared) |

Source: `.agents/project.yaml`  
Note: Staging and Production share the same Supabase project for the MVP — separate projects are a Phase 2 concern.

---

## Project Keys & Identifiers

| Key | Value |
|---|---|
| **App name** | Bunkai (分解) |
| **Package name** | `upex-bunkai-tms` |
| **Project key (Jira)** | `BK` |
| **Jira site** | `upexgalaxy67` (Atlassian Cloud) |
| **Supabase project ref** | `fmbpikzpkafptqximhxn` |

---

## Required Environment Variables

From `.env.example`:

### Core (required for app to run)
```bash
NEXT_PUBLIC_SUPABASE_URL=          # https://<project-ref>.supabase.co
SUPABASE_PUBLISHABLE_KEY=          # Browser-safe anon key
SUPABASE_SECRET_KEY=               # Server-only service_role key
SUPABASE_JWT_SECRET=               # For custom JWT signing/verification
NEXT_PUBLIC_APP_URL=               # Base URL for auth redirects (http://localhost:3000 local)
```

### Database (if using raw SQL / Prisma)
```bash
POSTGRES_HOST=
POSTGRES_USER=                     # usually postgres
POSTGRES_PASSWORD=
POSTGRES_DATABASE=                 # usually postgres
POSTGRES_URL=                      # pooled (port 6543)
POSTGRES_URL_NON_POOLING=          # direct (port 5432)
POSTGRES_PRISMA_URL=               # pooled with pgbouncer=true
```

### Third-party integrations (optional)
```bash
SUPABASE_ACCESS_TOKEN=             # Supabase MCP control-plane (admin)
RESEND_API_KEY=                    # Transactional email (not yet active in MVP)
TAVILY_API_KEY=                    # Web search MCP
N8N_API_URL=                       # Workflow automation
N8N_API_KEY=
ATLASSIAN_URL=                     # Jira integration
ATLASSIAN_EMAIL=
ATLASSIAN_API_TOKEN=
```

---

## Key Scripts (from `package.json`)

```bash
bun run dev             # Start Next.js dev server
bun run build           # Production build
bun run typecheck       # TypeScript check (tsc --noEmit)
bun run types:gen       # Generate Supabase TypeScript types
bun run api:sync        # Sync OpenAPI → TypeScript types
bun run repo:check      # Full repo lint (format + lint + types + vars + skills)
bun run repo:fix        # Auto-fix where possible
```

---

## Protected Routes & Auth Routing

From `middleware.ts`:

| Route prefix | Protection |
|---|---|
| `/projects/*` | Requires auth session → redirect to `/login?next=...` |
| `/onboarding/*` | Requires auth session → redirect to `/login?next=...` |
| `/login/*` | Public (auth pages) |
| `/auth/*` | Public (OTP callback) |
| `/api/auth/*` | Public (magic-link endpoints) |
| `/api/v1/*` | Auth enforced per-route via `requireAuth()` |
| `/qa` | Public (QA testing guide) |
| `/design-tokens` | Public (design system reference) |

---

## Project Assessment (Phase 1)

### Testing Maturity
| Category | Finding |
|---|---|
| **Unit tests** | No test framework detected in `devDependencies` (no Vitest, Jest, Bun test) |
| **E2E tests** | No Playwright, Cypress, or similar framework found |
| **CI/CD** | No `.github/workflows/` directory found — Vercel deploys likely triggered by git push |
| **Linting** | ESLint 9 + `@antfu/eslint-config` configured |
| **Type checking** | TypeScript 5.8 strict mode; `tsc --noEmit` in repo:check |
| **Formatting** | Prettier configured with `.prettierignore` |
| **Pre-commit hooks** | Husky + lint-staged (ESLint on `.ts/.tsx`, Prettier on `.json/.yml/.css`) |

### Risk Profile
| Risk | Level | Notes |
|---|---|---|
| No automated tests | HIGH | Zero unit or E2E coverage; this discovery run is the first step toward adding it |
| Shared staging/prod DB | MEDIUM | Staging and production share the same Supabase project ref |
| No CI pipeline | MEDIUM | No GitHub Actions; deploys are manual or Vercel auto-deploy on push |
| PAT secret isolation | LOW (handled) | Secrets stored in sibling tables, excluded from QA roles via RLS |
| RLS coverage | LOW (handled) | All tables have RLS; SECURITY DEFINER helpers prevent recursion |

### OpenAPI Status
- OpenAPI spec is generated at runtime via `app/api/openapi/route.ts`
- `bun run api:sync` regenerates `api/openapi-types.ts` from the live spec
- Interactive docs available at `/api/docs` (Scalar UI)

---

## Discovery Gaps

- [ ] **CI/CD**: No `.github/workflows/` found. Vercel deploy trigger (push to main? PR?) not confirmed.
- [ ] **Docker Compose**: Login page mentions self-hosting; no `docker-compose.yml` in repo. May be in a separate infra/deploy repo.
- [ ] **Team contacts**: No `CODEOWNERS`, `CONTRIBUTORS.md`, or team roster found in source.
- [ ] **Supabase local dev config**: `supabase/config.toml` may or may not exist — not confirmed.
