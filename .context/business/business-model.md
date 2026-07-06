# Business Model — Bunkai TMS

> **Discovery date**: 2026-05-31  
> **Method**: Source code analysis (DESIGN.md, login page copy, qa-config.ts, lib/types.ts)  
> **Status**: Phase 1 Constitution — discovered, not aspirational

---

## Product Identity

**Name**: Bunkai (分解) — Japanese martial-arts term: "decomposing a kata into its real combat applications"  
**Tagline**: "A test management system that decomposes user stories into executable Acceptance Test Cases."  
**Pronunciation**: BOON-kai (u as in "book"). Note: real martial-arts term — not the anime word. Surface this in onboarding, footer, and brand introductions.  
**License**: Apache-2.0  
**Domain**: bunkai.io (defensive: bankai.io redirects)

Source: `DESIGN.md` §Brand identity, `app/(auth)/login/page.tsx` left panel

---

## Problem Statement

QA teams write unstructured test steps that lose reusability and traceability. Three core problems:

1. **Disconnected test cases** — test steps don't trace to user stories or acceptance criteria
2. **Execution fragmentation** — manual, agentic (AI), and CI execution use incompatible formats
3. **No reuse model** — one-off scripts can't be reassembled; knowledge dies with the sprints

Source: `app/(auth)/login/page.tsx` feature ticks, `app/qa/qa-config.ts` product description

---

## Solution

Bunkai decomposes user stories into **ATCs (Acceptance Test Cases)** — atomic, layer-aware, reusable test cases that:

- Trace from UserStory → AcceptanceCriterion → ATC (via the IQL methodology)
- Execute identically in manual (human), agentic (Claude/OpenCode), and CI modes
- Assemble into automated tests via the KATA architecture

Source: `app/(auth)/login/page.tsx` feature ticks (IQL, ATC, KATA, ×3 execution)

---

## Target Users

| Persona | Signals |
|---|---|
| **QA Engineers (primary)** | "think in reusable test cases, not freeform steps"; manage hundreds of ATCs daily; need dense information UI |
| **Team Leads / SDETs** | workspace admin role; manages members, assigns roles (viewer/member/admin/owner) |
| **Developers / Stakeholders** | viewer role; read-only access to test coverage and results |
| **AI Agents / CI systems** | headless access via Bearer PAT; scoped permissions (atc:read, run:execute) |

Source: `DESIGN.md` §Design principles (principle 1: QAs manage hundreds of ATCs daily), `lib/types.ts` role enum

---

## Value Proposition

| Value | Description |
|---|---|
| **Reusability** | ATCs are KATA components — assemble into test suites without copy-paste |
| **Traceability** | Full IQL chain: Story → AC → ATC → Run → Bug |
| **Execution convergence** | Manual, agentic (Claude/OpenCode), CI pipelines use the same schema and produce the same reports |
| **Data sovereignty** | Self-hostable (Docker Compose), Apache-2.0 — test specs stay on the user's servers |
| **Information density** | Packed UI with status chips, layer chips, monospace IDs — built for high-volume QA ops |
| **Layer clarity** | UI / API / Unit testing visually distinguished in one system |

Source: `app/(auth)/login/page.tsx` FEATURE_TICKS array, `DESIGN.md` §Design principles

---

## Revenue / Monetization Model

| Plan | Tier | Deployment |
|---|---|---|
| **community** | OSS free | Self-hosted (Docker Compose) |
| **cloud** | SaaS (paid) | Managed cloud (Vercel + Supabase) |
| **enterprise** | Custom pricing | Custom deployment |

Source: `supabase/migrations/0001_tenancy.sql` — `workspaces.plan` CHECK constraint; `app/(auth)/login/page.tsx` "Self-hosted" CTA

---

## Competitive Positioning

| Compared to | Bunkai's differentiation |
|---|---|
| Jira / Linear | Test-case-centric (not issue-centric); structured story→AC→ATC traceability; dense QA-first UI |
| TestRail / Zephyr | Passwordless onboarding; OSS self-hosting; agentic execution built-in; no per-seat pricing for OSS |
| Playwright / Cypress | Higher abstraction layer (story/case); KATA architecture for reuse; agent-friendly schema |
| Notion / Confluence | Executable test cases, not docs; status tracking; run history |

Source: `DESIGN.md` §Brand identity (inspired-by list), `app/(auth)/login/page.tsx` brand copy

---

## Current Phase & Roadmap Signals

| Phase | Status | Key features |
|---|---|---|
| **Phase 1 (current)** | Implemented | Dark mode, core UI/API/Unit layers, passwordless magic-link auth, ATC editor, workspace management, PAT auth, invite flow |
| **Phase 2 (planned)** | Mentioned | Light theme, GitHub OAuth, Google OAuth, SSO |
| **Run execution** | Schema ready | `atcs.status` enum exists; full run result tables deferred |
| **CI integration** | Planned | Same schema — execution mode abstracted in IQL |

Source: `DESIGN.md` §Phase roadmap, `app/(auth)/login/page.tsx` OAuth "coming next sprint", `lib/types.ts` AtcStatus enum

---

## Discovery Gaps

- [ ] **Pricing tiers**: Exact pricing for `cloud` and `enterprise` plans not found in source code. No pricing page detected.
- [ ] **Run history tables**: ATC status exists as a field; full run-result/execution-history tables not present in migrations (Phase 2 deferred).
- [ ] **Email provider**: `RESEND_API_KEY` in `.env.example` but MVP uses Supabase native email for magic links — final email provider unconfirmed.
- [ ] **Self-hosted Docker Compose**: Login page mentions it; no `docker-compose.yml` found in repo root or docs/. May be in a separate deploy repo.
- [ ] **Team / company behind Bunkai**: No `COMPANY.md`, `CONTRIBUTORS.md`, or `package.json` author field. Associated with UPEX brand but formal org not documented in source.
