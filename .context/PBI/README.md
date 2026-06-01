# PBI — Product Backlog Items

> **Project**: Bunkai TMS  
> **Jira project key**: `BK`  
> **Jira site**: `upexgalaxy67.atlassian.net`  
> **TMS modality**: **Xray on Jira** (Modality A — Test, Test Plan, Test Set, Test Execution, Precondition issue types active)  
> **Discovery date**: 2026-05-31

---

## Structure

```
PBI/
├── README.md                           ← This file — backlog access recipe + queries
├── templates/                          ← Format-reference guides (NOT copies of Jira)
│   ├── user-story.md                   ← BK Story canonical shape
│   ├── bug-report.md                   ← BK Bug/Defect canonical shape
│   ├── test-plan.md                    ← BK Test Plan (Xray) canonical shape
│   ├── module-context-template.md      ← Module technical context
│   ├── ROADMAP-template.md             ← Module roadmap with phases
│   └── PROGRESS-template.md            ← Cross-session progress tracker
│
├── ── PER-STORY (simple) ─────────────
├── {TICKET-ID}-feature-name.md         ← User Story with ticket ID (simple stories)
│
└── ── PER-MODULE (complex) ───────────
    └── {module-name}/
        ├── {module}-test-plan.md       ← Master module test plan
        ├── BK-{id}-{feature}/          ← Per-ticket context
        │   ├── context.md              ← AC summary, code locations, test data
        │   └── evidence/               ← Screenshots, logs (gitignored)
        └── test-specs/
            ├── ROADMAP.md
            ├── PROGRESS.md
            └── {PREFIX}-T01-{name}/
                ├── spec.md             ← Business-level TCs in Gherkin
                ├── implementation-plan.md ← KATA components, fixtures
                └── atc/               ← Individual ATC specs
                    └── BK-{id}-{name}.md
```

**Important**: Jira is the source of truth. Files here are **read-only cache** — never edit them to change Jira. Per-ticket PBI is synced from Jira by `/sprint-testing` via `bun run jira:sync-issues get BK-<N>`.

---

## Jira Project Catalog

### Epics (module-level containers)

| Key | Epic | Status |
|---|---|---|
| BK-1 | Tenancy & Identity | Planning |
| BK-7 | Project & Module Hierarchy | Planning |
| BK-12 | User Stories & Acceptance Criteria | Planning |
| BK-13 | ATC Library (Atomic Test Components) | Planning |
| BK-24 | Tests (chains of ATCs) | Planning |
| BK-29 | Bunkai TMS — Testing Credentials (DB / API / UI) | Backlog |
| BK-30 | Manual Execution & Runs | Planning |
| BK-31 | Bugs & Defect Heatmap | Planning |

### Issue Types

| Type | Category | Xray? | Key statuses |
|---|---|---|---|
| **Epic** | Container | No | Backlog → Planning → In Progress → Done / ABORTED |
| **Story** | Feature work | No | Backlog → Shift-Left QA → Estimation → Ready For Dev → In Progress → BLOCKED → Ready For QA → In Test → QA Approved → Ready For Release → Deployed to Production / ABORTED |
| **Task** | Technical work | No | ACTIVE → Close |
| **Bug** | Defect | No | Open → In Progress → Ready For QA → In Review → Closed / REJECTED / Duplicated / Deferred / ABORTED |
| **Defect** | QA-filed defect | No | Open → In Progress → Ready For QA → In Review → Closed / REJECTED / Duplicated / Deferred |
| **Tech Debt** | Engineering debt | No | — |
| **Tech Story** | Technical story | No | — |
| **Improvement** | Enhancement | No | — |
| **Test** | Xray test case | Yes | Draft → Candidate → In Design → In Review → In Automation → Pull Request → MANUAL / AUTOMATED / READY → DEPRECATED |
| **Test Plan** | Xray test plan | Yes | Planning → Plan Automated → READY → Completed |
| **Test Set** | Xray test set | Yes | Designing → Close |
| **Test Execution** | Xray test run | Yes | ACTIVE → Close |
| **Re-Test Execution** | Xray re-test | Yes | ACTIVE → Close |
| **Precondition** | Xray precondition | Yes | ACTIVE |

---

## Common Queries

### Current sprint (Stories)
```
project = BK AND issuetype = Story AND sprint in openSprints() ORDER BY priority DESC
```

### Backlog by epic
```
project = BK AND issuetype = Story AND "Epic Link" = BK-13 AND status = Backlog
```

### Ready for QA
```
project = BK AND issuetype = Story AND status = "Ready For QA" ORDER BY priority DESC
```

### Open bugs
```
project = BK AND issuetype in (Bug, Defect) AND status = Open ORDER BY created DESC
```

### All stories for a module (by epic)
```
project = BK AND issuetype = Story AND "Epic Link" = BK-{N} ORDER BY created ASC
```

### Xray tests by status
```
project = BK AND issuetype = Test AND status = Candidate ORDER BY created DESC
```

### Shift-Left QA queue
```
project = BK AND issuetype = Story AND status = "Shift-Left QA" ORDER BY priority DESC
```

---

## Auth Recipe (REST API)

```bash
# Load credentials from .env
source .env

# Get a story
curl -sS -X POST -u "$ATLASSIAN_EMAIL:$ATLASSIAN_API_TOKEN" \
  "${ATLASSIAN_URL}/rest/api/3/search/jql" \
  -H "Content-Type: application/json" \
  -d '{"jql":"key=BK-N","maxResults":1,"fields":["summary","description","status","assignee","labels","priority","customfield_10014"]}'

# Search with JQL (new endpoint — old /rest/api/3/search is DEPRECATED)
curl -sS -X POST -u "$ATLASSIAN_EMAIL:$ATLASSIAN_API_TOKEN" \
  "${ATLASSIAN_URL}/rest/api/3/search/jql" \
  -H "Content-Type: application/json" \
  -d '{"jql":"project=BK AND status=Ready For QA","maxResults":50,"fields":["summary","status","priority"]}'
```

**Note**: `/rest/api/3/search` (GET) is deprecated — always use `POST /rest/api/3/search/jql`.

---

## Conventions

- **Prefix**: `BK-` (project key from `.agents/project.yaml` `{{PROJECT_KEY}}`)
- **File names**: kebab-case, e.g. `BK-42-defect-heatmap/`
- **Module folder name**: maps to Epic name, kebab-case, e.g. `atc-library/`
- **Evidence**: Add `evidence/` to `.gitignore` (screenshots, logs are ephemeral)
- **ACs**: Mark as `[x]` when covered by automated tests
- **TMS modality**: Xray — `[TMS_TOOL]` resolves to `/xray-cli` for test entities; `[ISSUE_TRACKER_TOOL]` resolves to `/acli` for stories/bugs
