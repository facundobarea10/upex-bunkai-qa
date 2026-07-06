# Infrastructure — Environments & Deploy

> **Discovery date**: 2026-05-31  
> **Sources**: `.agents/project.yaml`, `.env.example`, `package.json`, `scripts/agents-setup.ts`, `scripts/gen-supabase-types.ts`

---

## Environments

| Environment | Web URL | API URL | DB Project Ref |
|---|---|---|---|
| **local** | `http://localhost:3000` | `http://localhost:3000/api` | `fmbpikzpkafptqximhxn` |
| **staging** | `https://staging-upexbunkai.vercel.app` | `https://staging-upexbunkai.vercel.app/api` | `fmbpikzpkafptqximhxn` (shared) |
| **production** | `https://upexbunkai.vercel.app` | `https://upexbunkai.vercel.app/api` | `fmbpikzpkafptqximhxn` (shared) |

**Note**: Staging and Production currently share the same Supabase project (`fmbpikzpkafptqximhxn`). Separate projects are a Phase 2 concern.

---

## Deployment Targets

### Frontend + Backend (monorepo)
| Property | Value |
|---|---|
| Host | Vercel |
| Framework preset | Next.js (auto-detected) |
| Build command | `next build` |
| Output | `.next/` (standard mode, NOT standalone) |
| Runtime | Vercel Edge Functions / Node.js serverless |
| Deploy trigger | Inferred: git push to main branch (no CI config found) |
| Staging URL | `https://staging-upexbunkai.vercel.app` |
| Production URL | `https://upexbunkai.vercel.app` |

### Database + Auth
| Property | Value |
|---|---|
| Host | Supabase Cloud |
| Project ref | `fmbpikzpkafptqximhxn` |
| Region | Not documented in source |
| Plan | Unknown (Pro inferred for shared staging/prod) |
| Backups | Supabase managed (daily on Pro plan) |
| Migrations | Applied via Supabase CLI (`supabase db push`) or dashboard |

---

## Local Development Setup

```bash
# 1. Clone repo
git clone <repo-url>
cd upex-bunkai-tms

# 2. Install dependencies
bun install

# 3. Configure environment
cp .env.example .env
# Fill in: NEXT_PUBLIC_SUPABASE_URL, SUPABASE_PUBLISHABLE_KEY,
#          SUPABASE_SECRET_KEY, SUPABASE_JWT_SECRET, NEXT_PUBLIC_APP_URL

# 4. Configure agent/skill variables
bun run agents:setup           # interactive CLI → writes .agents/project.yaml

# 5. Generate Supabase TypeScript types
bun run types:gen              # requires NEXT_PUBLIC_SUPABASE_URL in .env

# 6. Sync OpenAPI spec + types (optional for dev, required for full type safety)
bun run api:sync

# 7. Start development server
bun run dev                    # http://localhost:3000
```

**Load env in Claude Code session**: `bun run claude` (wraps with dotenv-cli). Auth magic-link redirects point to `NEXT_PUBLIC_APP_URL` — must be `http://localhost:3000` for local dev.

---

## CI/CD

| Property | Status |
|---|---|
| CI platform | **Not found** — no `.github/workflows/` directory detected |
| Deploy method | Inferred: Vercel git integration (auto-deploy on push) |
| Pre-commit hooks | Husky + lint-staged (ESLint + Prettier) |
| Repo gate | `bun run repo:check` (format + lint + types + vars + skills) |

**Pre-commit hooks** (`package.json` lint-staged):
```json
{
  "*.{ts,tsx,js,jsx}": ["eslint --fix"],
  "*.{json,yml,yaml,css,scss,html}": ["prettier --write --ignore-path .prettierignore"]
}
```

**Inferred Vercel pipeline**:
```
git push → Vercel webhook → next build → deploy to Vercel CDN
```
No build matrix, no test step in CI (no test runner found).

---

## Key Infrastructure Scripts

| Script | Command | Purpose |
|---|---|---|
| Agent config | `bun run agents:setup` | Interactive `.agents/project.yaml` setup |
| Supabase types | `bun run types:gen` | Regenerate `lib/types/supabase.ts` from live schema |
| OpenAPI sync | `bun run api:sync` | Download spec + generate `api/openapi-types.ts` |
| OpenAPI diff | `bun run openapi:diff` | Diff current spec vs. last sync |
| Jira field sync | `bun run jira:sync-fields` | Refresh `jira-fields.json` from Jira workspace |
| Jira workflow sync | `bun run jira:sync-workflows` | Refresh `jira-workflows.json` |
| Jira issue sync | `bun run jira:sync-issues` | Pull PBI data into `.context/PBI/` |
| Repo gate | `bun run repo:check` | Full lint/type/format check |
| Auto-fix | `bun run repo:fix` | Auto-fix all fixable issues |
| Onboarding | `bun run onboarding` | Launch interactive onboarding guide |

---

## Secrets Required per Environment

| Secret | Local | Staging | Production |
|---|---|---|---|
| `NEXT_PUBLIC_SUPABASE_URL` | `.env` | Vercel env | Vercel env |
| `SUPABASE_PUBLISHABLE_KEY` | `.env` | Vercel env | Vercel env |
| `SUPABASE_SECRET_KEY` | `.env` | Vercel env | Vercel env |
| `SUPABASE_JWT_SECRET` | `.env` | Vercel env | Vercel env |
| `NEXT_PUBLIC_APP_URL` | `.env` (localhost) | Vercel env | Vercel env |
| `RESEND_API_KEY` | `.env` (optional) | Vercel env | Vercel env |
| `ATLASSIAN_API_TOKEN` | `.env` (optional) | Not needed | Not needed |
| `TAVILY_API_KEY` | `.env` (optional) | Not needed | Not needed |

---

## Monitoring & Observability

| Concern | Status |
|---|---|
| Health endpoint | `GET /api/v1/health` → `{ ok, service, environment, timestamp }` |
| Request logging | `lib/api/logging.ts` (all API routes) |
| Audit trail | `activity_log` table (entity mutations) |
| APM / Error tracking | **Not found** — no Sentry, Datadog, or similar |
| Uptime monitoring | **Not found** — no external uptime service configured |

---

## Self-Hosting (Community Edition)

Login page references Docker Compose self-hosting. However:
- No `docker-compose.yml` or `Dockerfile` found in the repo
- Likely in a separate deploy/infra repo (not yet discovered)
- Community plan (`workspaces.plan = 'community'`) targets self-hosted users

---

## Discovery Gaps

- [ ] **CI/CD pipeline**: No `.github/workflows/` directory. Deploy process (branch strategy, PR checks, production promotion) not documented.
- [ ] **Vercel configuration**: No `vercel.json` found. Region selection, environment variable mapping, and build overrides not confirmed.
- [ ] **Supabase local dev**: `supabase/config.toml` not found — local Supabase stack (`supabase start`) not initialized.
- [ ] **Docker Compose for self-hosting**: Referenced in login page but not present in this repo.
- [ ] **Database migrations deployment**: How migrations are applied to staging/production (Supabase CLI, dashboard, or automated) not confirmed.
- [ ] **RTO / RPO**: No recovery targets documented. Dependent on Supabase managed backup schedule.
- [ ] **Staging vs. production isolation**: Shared Supabase project ref risks data bleed between staging and production tests.
