# PRD — Executive Summary

> **Discovery date**: 2026-05-31  
> **Method**: Reverse-engineered from source code (DESIGN.md, login page, page components, lib/types.ts)  
> **Status**: Discovered — describes what the system IS, not what stakeholders wish it were

---

## Product Overview

**Bunkai** (分解) is a developer-first Test Management System (TMS) that decomposes user stories into executable Acceptance Test Cases (ATCs). It is built for QA engineers who manage large volumes of structured tests and need traceability from requirement to execution.

The product runs as a multi-tenant SaaS (and self-hosted OSS) application. The core unit is the **ATC** — an atomic, layer-aware, reusable test case anchored to one or more Acceptance Criteria.

---

## Problem Statement

QA teams suffer from three compounding problems:

| Problem | Symptom |
|---|---|
| **Disconnected tests** | Test cases live in spreadsheets or test tools, unlinked to the user stories or acceptance criteria they verify |
| **Fragmented execution** | Manual testing, AI-agent execution, and CI pipelines use incompatible formats — same test described three different ways |
| **No reuse model** | One-off test scripts can't be assembled into suites; institutional knowledge dies with each sprint |

Source: `app/(auth)/login/page.tsx` feature ticks, `app/qa/qa-config.ts` product description

---

## Solution

Bunkai introduces the **IQL methodology** (Integrated Quality Lifecycle) as a structural constraint:

```
UserStory → AcceptanceCriterion → ATC → Run → Bug
```

Every entity maps to exactly one IQL stage. ATCs are the unit of reuse — composable via the **KATA architecture** (Komponent Action Test Architecture) into automated test suites. Manual, agentic (Claude/OpenCode), and CI executions run against the same schema and produce the same reports.

---

## Success Metrics

Derived from design specifications and source code constraints:

| Metric | Target | Source |
|---|---|---|
| Navigation speed | Route transitions < 80ms | `DESIGN.md` §No transitions |
| Command palette latency | Open in < 100ms | `DESIGN.md` §Command palette |
| Onboarding time | First workspace created in < 2 minutes | Onboarding form: 2 fields (name + slug) |
| Invite generation | < 5 seconds from email entry to link-copied | Members page flow |
| Accessibility | WCAG AA contrast on all color tokens | `DESIGN.md` §Accessibility note |
| Information density | Module tree + 50+ ATCs visible without horizontal scroll | `DESIGN.md` §Design principle 1 |

---

## Scope — Phase 1 (Implemented)

| In scope | Details |
|---|---|
| Multi-tenant workspaces | Workspace creation, RBAC (viewer/member/admin/owner), invite flow |
| Project hierarchy | Projects → Modules (tree, max depth 6) → UserStories → AcceptanceCriteria |
| ATC management | Create, edit, delete ATCs; bind to ACs; steps + assertions editor |
| Authentication | Magic-link OTP (browser) + Bearer PAT (CLI/agents) |
| Team management | Invite by email, role assignment, revoke, rotate invite links |
| API | RESTful `/api/v1/` with OpenAPI spec + Scalar docs UI |
| Dark mode UI | Canonical theme; shadcn/ui components with custom design tokens |

| Out of scope (Phase 1) | Deferred to |
|---|---|
| Test run execution / reporting | Phase 2 |
| Light mode | Phase 2 |
| GitHub OAuth / Google SSO | Next sprint (per login page) |
| CI/CD pipeline integration | Planned |
| Email automation for invites | Post-MVP (MVP: clipboard copy) |
| Mind-map view | Phase 2 |
| Bulk import/export | Not in roadmap |
| Mobile/responsive design | Not in roadmap |

---

## Competitive Differentiation

| vs. | Bunkai's angle |
|---|---|
| Jira / Linear | Test-case-centric, not issue-centric; ATC ↔ AC traceability built-in |
| TestRail / Zephyr | OSS + self-hostable (Apache-2.0); passwordless onboarding; agentic execution |
| Playwright / Cypress | Higher abstraction (story/case layer); KATA architecture for reuse |
| Notion / Confluence | ATCs are executable, not docs; status tracking + run history |

---

## Discovery Gaps

- [ ] **Quantitative success metrics**: No KPI targets (MAU, test coverage %, uptime SLA) found in source. Metrics above are inferred from design constraints.
- [ ] **Test run model**: `atcs.status` exists but no `runs` or `run_results` tables in current migrations. Run execution is partially scoped.
- [ ] **Revenue targets**: No pricing page or revenue model documentation in repo.
