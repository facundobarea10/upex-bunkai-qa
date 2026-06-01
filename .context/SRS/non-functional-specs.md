# SRS — Non-Functional Specifications

> **Discovery date**: 2026-05-31  
> **Method**: Reverse-engineered from DESIGN.md, lib/env.ts, supabase/migrations/, middleware.ts  
> **Note**: Quantitative targets without explicit source are marked as inferred from design constraints. Items with no signal are marked as Discovery Gaps.

---

## Performance

### NFR-P1 — UI Responsiveness
| Requirement | Target | Source |
|---|---|---|
| Route transitions | < 80ms (no fade/animation) | `DESIGN.md` §No page transitions |
| Command palette open | < 100ms | `DESIGN.md` §Command palette |
| ATC table render (50+ rows) | No explicit target — dense layout implies instant render | `DESIGN.md` §Principle 1 |
| Magic-link email delivery | No target — depends on Supabase + email provider | Inferred |

### NFR-P2 — API Latency
No explicit targets found in source. Inferred from architecture:
- Supabase RPC calls (`bunkai_save_atc`, `bunkai_bootstrap_workspace`) are single-round-trip atomic operations — optimized for low-latency writes.
- Idempotency key TTL: 24 hours — implies repeated requests within TTL window return cached responses instantly.

### NFR-P3 — Database Performance
| Optimization | Detail | Source |
|---|---|---|
| GIN index on `atcs.tsv` | Full-text search on title + tags | `0004_atcs.sql` |
| Composite index `(workspace_id, created_at DESC)` | `activity_log` queries | `0009_cross_cutting.sql` |
| Index `(user_id, revoked_at)` | Fast active PAT lookup | `0008_access_tokens.sql` |
| Index `lower(email)` on invites | Case-insensitive email lookup | `0010_workspace_invites.sql` |
| Index `expires_at` on idempotency_keys | TTL cleanup queries | `0009_cross_cutting.sql` |
| SECURITY DEFINER helpers | Cached RLS functions (no recursion overhead) | `0005_rls_helpers.sql` |

---

## Security

### NFR-S1 — Authentication
| Requirement | Implementation | Source |
|---|---|---|
| Passwordless browser auth | Magic-link OTP via Supabase Auth | `app/api/v1/auth/magic-link/route.ts` |
| Agent/CLI auth | Bearer PAT with scoped permissions | `lib/api/auth.ts` |
| Session management | Supabase cookie-based sessions; refreshed by middleware on every request | `middleware.ts` |
| Token format | `bk_pat_<prefix>.<secret>` — prefix for O(1) DB lookup; secret never stored raw | `lib/api/pat.ts` |
| Secret isolation | PAT/invite/magic-link secrets in 1:1 sibling tables, excluded from QA roles | `0011_split_token_secrets.sql` |

### NFR-S2 — Authorization
| Requirement | Implementation | Source |
|---|---|---|
| Data isolation | RLS on all 18 tables; tenancy enforced at DB layer | All migrations |
| RLS helper functions | `SECURITY DEFINER` to prevent infinite recursion | `0005_rls_helpers.sql` |
| Role hierarchy | `owner > admin > member > viewer`; enforced via RLS policies | `0001_tenancy.sql` |
| Write permissions | Requires `role ∈ {member, admin, owner}` (`bunkai_can_write_workspace`) | `0005_rls_helpers.sql` |
| Admin operations | Requires `role ∈ {admin, owner}` (`bunkai_is_workspace_admin`) | `0005_rls_helpers.sql` |

### NFR-S3 — Input Validation
| Requirement | Implementation | Source |
|---|---|---|
| API request validation | Zod schemas on all API route inputs | `lib/openapi/registry.ts` |
| Slug validation | Regex `^[a-z0-9][a-z0-9-]{1,38}[a-z0-9]$` (workspace + project slugs) | `app/(app)/onboarding/onboarding-form.tsx`, `0006_bootstrap_workspace.sql` |
| Reserved slug rejection | Hardcoded blocklist: api, auth, admin, login, etc. | `app/api/v1/workspaces/route.ts` |
| Email validation | Regex `^[^\s@]+@[^\s@][^\s.@]*\.[^\s@]+$` (client-side) | `app/(auth)/login/magic-link-form.tsx` |
| Open redirect prevention | `next` param: must start with `/`, must not start with `//` | `app/auth/callback/route.ts` |

### NFR-S4 — Secrets Handling
| Requirement | Implementation |
|---|---|
| Credentials never in code | All secrets via `lib/env.ts` Zod schema + `.env` |
| Token hashing | SHA-256 for PATs, invites, and magic-link tokens |
| PAT raw value returned once | Only at creation; not stored; not retrievable |
| Invite token returned once | Only at creation; rotate to get new one |

### NFR-S5 — Replay Protection
- Idempotency keys: `(user_id, endpoint, key)` UNIQUE — 24h TTL, response snapshot cached.
- Magic-link tokens: `consumed_at` timestamp; single-use.

---

## Reliability

### NFR-R1 — Data Integrity
| Requirement | Mechanism | Source |
|---|---|---|
| Atomic workspace creation | `bunkai_bootstrap_workspace` RPC transaction | `0006_bootstrap_workspace.sql` |
| Atomic ATC save | `bunkai_save_atc` RPC: delete-then-insert steps/assertions/bindings | `0007_save_atc.sql` |
| Cascade deletes | Module → stories → ACs → ATCs cascade on delete | `0003_authoring.sql`, `0004_atcs.sql` |
| RESTRICT delete | ATC → user_story FK is RESTRICT (can't delete story with ATCs) | `0004_atcs.sql` |
| ATC version counter | `atcs.version` incremented on save (optimistic locking, Phase E) | `0004_atcs.sql` |
| Auto-updated timestamps | `atcs_set_updated_at` trigger on `atcs` table | `0004_atcs.sql` |
| Max module depth | CHECK constraint: path depth ≤ 6 | `0002_projects_modules.sql` |

### NFR-R2 — Availability
| Environment | SLA | Notes |
|---|---|---|
| Staging | No explicit SLA | Shared Supabase project with Production |
| Production | Not documented | Dependent on Vercel + Supabase Cloud SLAs |
| Supabase Cloud | 99.9% (Supabase Pro) | External dependency |
| Vercel | 99.99% (Vercel Pro) | External dependency |

**Discovery Gap**: No custom RTO/RPO targets found in source.

### NFR-R3 — Backup & Recovery
No backup policy found in source. Dependent on Supabase managed backups (daily on Pro plan).

---

## Scalability

### NFR-SC1 — Multi-Tenancy
- Full tenant isolation via RLS (workspace_id scoping on all tables)
- Workspace plan tiers (community / cloud / enterprise) indicate planned tier-based resource limits
- No rate limiting found in Phase 1 source

### NFR-SC2 — Data Volume Signals
| Entity | Volume signal |
|---|---|
| ATCs | UI designed for 50+ visible without scroll (`DESIGN.md` §Principle 1: "QAs manage hundreds of ATCs daily") |
| Modules | Max depth 6; no breadth limit found |
| Workspace members | No explicit limit |
| PAT scopes | Hardcoded to 4 scope values |

---

## Accessibility

### NFR-A1 — WCAG Compliance
| Requirement | Detail | Source |
|---|---|---|
| Color contrast | WCAG AA on all design token combinations | `DESIGN.md` §Accessibility note |
| Status indicators | Status chips always pair color + text label + icon (no color-only signal) | `DESIGN.md` §Principle 4 |
| Keyboard navigation | Command palette (Cmd/Ctrl+K), keyboard shortcut "N" for New ATC | `app/(app)/projects/[projectSlug]/page.tsx` |
| Focus rings | Accent vermillion used for focus state | `DESIGN.md` §Accent |

---

## Observability

### NFR-O1 — Logging
- `lib/api/logging.ts`: request/response logging on all API routes
- `lib/api/request-id.ts`: unique request ID per call
- `activity_log` table: entity-level audit trail (create/update/delete + JSONB payload)

### NFR-O2 — Monitoring
- `GET /api/v1/health`: returns `{ ok, service, environment, timestamp }` — suitable for uptime monitors
- No APM integration (Datadog, Sentry, etc.) found in Phase 1

---

## Maintainability

### NFR-M1 — Code Quality Gates (from `package.json`)
| Gate | Command | Tool |
|---|---|---|
| Type checking | `bun run typecheck` | TypeScript 5.8 `tsc --noEmit` |
| Linting | `bun run lint:check` | ESLint 9 + `@antfu/eslint-config` |
| Formatting | `bun run format:check` | Prettier |
| Pre-commit | Husky + lint-staged | ESLint on .ts/.tsx; Prettier on .json/.yml/.css |
| Full gate | `bun run repo:check` | All of the above combined |

### NFR-M2 — API Documentation
- OpenAPI spec auto-generated from Zod schemas at `/api/openapi`
- Scalar interactive docs at `/api/docs`
- `bun run api:sync` refreshes `api/openapi-types.ts` from live spec

---

## Compliance

### NFR-C1 — License
- Apache-2.0 — full open source, commercial use allowed, patent protection
- "Your test specifications stay on your servers" — data sovereignty by design
- Self-hosting option (Docker Compose) for air-gapped or regulated environments

### NFR-C2 — Data Residency
- Cloud deployment: Supabase Cloud (region not specified in source)
- Self-hosted: user controls data location (community plan)

---

## Discovery Gaps

- [ ] **RTO / RPO**: No recovery time or recovery point objectives documented.
- [ ] **Rate limiting**: No rate limit middleware or headers found in Phase 1.
- [ ] **GDPR / CCPA compliance**: No data deletion ("right to be forgotten") endpoints found. `auth.users` is Supabase managed.
- [ ] **CI test coverage targets**: No test coverage configuration found (no test runner in devDependencies).
- [ ] **Performance budgets (Core Web Vitals)**: No LCP/CLS/FID targets found in source.
- [ ] **Audit log retention**: `activity_log` has no TTL or archival policy defined.
