# Test Plan — {Module / Feature Name}

> **Type**: Test Plan (Xray) | **Project**: BK — Bunkai TMS  
> **Xray key**: BK-{N}  
> **Status**: {Planning | Plan Automated | READY | Completed}  
> **Scope**: {Epic BK-N — {Epic Name} | Sprint {N} | Release {version}}  
> **Owner**: {QA Lead name}  
> **Date**: {YYYY-MM-DD}

---

## Scope & Objective

{What this test plan covers, why it exists, and what "done" looks like.}

**In scope**:
- {Feature / module / flow 1}
- {Feature / module / flow 2}

**Out of scope**:
- {What is explicitly NOT tested here and why}

---

## Entry Criteria

- [ ] Story `BK-{N}` status = `Ready For QA`
- [ ] Development deployed to staging environment
- [ ] Test data provisioned (see §Test Data)
- [ ] Prior sprint regression passed (no P1/P2 open bugs)

---

## Exit Criteria

- [ ] All High priority test cases executed
- [ ] Zero P1 (Critical) open bugs
- [ ] P2 (High) bugs: approved exception list or fixed
- [ ] ATC coverage ≥ 80% of ACs in scope
- [ ] Xray Test Execution closed with verdict

---

## Test Strategy

### Layers covered

| Layer | Tool | Scope |
|---|---|---|
| UI | Playwright (KATA) | User journeys, form validation, status chips |
| API | Playwright API / curl | REST `/api/v1/` endpoints, error responses |
| Unit | — | N/A for this plan |

### Execution modes

| Mode | Runner | When |
|---|---|---|
| Manual | QA Analyst | Exploratory + edge cases |
| Agentic | Claude / OpenCode (Bearer PAT) | Happy path regression |
| CI | GitHub Actions (planned) | Post-deploy smoke |

---

## Test Cases

### Happy Path

| TC ID | ATC slug | Scenario | Layer | Priority | Status |
|---|---|---|---|---|---|
| TC-001 | {atc-slug} | {Happy path scenario} | UI | High | unrun |
| TC-002 | {atc-slug} | {API happy path} | API | High | unrun |

### Negative / Error Cases

| TC ID | ATC slug | Scenario | Layer | Priority | Status |
|---|---|---|---|---|---|
| TC-003 | {atc-slug} | {Invalid input → error message} | UI | Medium | unrun |
| TC-004 | {atc-slug} | {Unauthorized access → 401} | API | High | unrun |

### Edge Cases

| TC ID | ATC slug | Scenario | Layer | Priority | Status |
|---|---|---|---|---|---|
| TC-005 | {atc-slug} | {Boundary condition} | UI | Low | unrun |

---

## Test Data

| Entity | Value | Environment | Notes |
|---|---|---|---|
| Workspace | `test-workspace-{n}` | staging | Created via API before suite |
| User (owner) | `STAGING_USER_EMAIL` from `.env` | staging | Magic-link login |
| User (member) | `staging-member@test.com` | staging | Invited via invite flow |
| ATC | `test-atc-slug` | staging | Pre-seeded via `bunkai_save_atc` RPC |

**Credentials**: Read from `.env` — never hardcode. Keys: `STAGING_USER_EMAIL`, `STAGING_USER_PASSWORD`.

---

## Risk Register

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Shared staging/prod DB | High | High | Use isolated test workspace; tear down after suite |
| No magic-link email in test env | Medium | High | Use `POST /api/v1/auth/signin` (password-based) for agents |
| Flaky invite flow (clipboard) | Low | Medium | Test invite acceptance via API directly |

---

## Defect Triage

| Severity | SLA | Action |
|---|---|---|
| P1 Critical | Block release | Immediately file Bug in BK, notify lead |
| P2 High | Fix in sprint | File Bug in BK, link to story |
| P3 Medium | Next sprint | File Bug in BK, backlog |
| P4 Low | Backlog | File Improvement in BK |

**Bug workflow**: Open → In Progress → Ready For QA → In Review → Closed  
**Defect workflow**: Open → In Progress → Ready For QA → In Review → Closed  

---

## Xray Artifacts

| Artifact | Xray type | Key |
|---|---|---|
| This test plan | Test Plan | BK-{N} |
| Test cases | Test | BK-{N}, BK-{N}, ... |
| Execution record | Test Execution | Created when run starts |
| Preconditions | Precondition | BK-{N} (if applicable) |

---

## Sign-off

| Role | Name | Date | Status |
|---|---|---|---|
| QA Lead | {name} | {date} | Pending |
| Dev Lead | {name} | {date} | Pending |
| Product Owner | {name} | {date} | Pending |

---

**Jira link**: {ATLASSIAN_URL}/browse/BK-N
