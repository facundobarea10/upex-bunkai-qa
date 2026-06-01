# [BK-N] {Story Title}

> **Type**: Story | **Project**: BK — Bunkai TMS  
> **Epic**: BK-{epic-id} — {Epic Name}  
> **Status**: {Backlog | Shift-Left QA | Estimation | Ready For Dev | In Progress | BLOCKED | Ready For QA | In Test | QA Approved | Ready For Release | Deployed to Production | ABORTED}  
> **Priority**: {Highest | High | Medium | Low | Lowest}  
> **Assignee**: {name or unassigned}  
> **Labels**: {labels}

---

## User Story

```
As a [role: QA Lead | SDET | Manual Tester | AI Agent]
I want [feature/capability]
So that [business value / outcome]
```

---

## Context

{Brief description of the feature in plain terms — what it does, who uses it, and why it matters. 2–4 sentences.}

**Module / Epic**: {which Epic and module this belongs to}  
**Related entities**: {Workspace | Project | Module | UserStory | ATC | etc.}  
**API endpoints**: {relevant /api/v1/ routes, if any}  
**DB tables**: {relevant tables from the schema, if any}

---

## Acceptance Criteria

- [ ] **AC1**: {Criterion description — specific, testable}
- [ ] **AC2**: {Criterion description}
- [ ] **AC3**: {Criterion description}

---

## Test Scenarios

| ID | Layer | Scenario | Type | Priority |
|---|---|---|---|---|
| TS-001 | UI | Happy path — {scenario name} | Functional | High |
| TS-002 | API | {scenario name} | Functional | High |
| TS-003 | UI | Error state — {scenario name} | Negative | Medium |
| TS-004 | UI | Edge case — {scenario name} | Boundary | Low |

---

## Test Data Requirements

| Entity | Field | Value | Notes |
|---|---|---|---|
| Workspace | slug | `test-workspace` | Must exist before test |
| User | role | `member` | Invited + accepted |
| ATC | status | `unrun` | Default state |

---

## Open Questions

- [ ] {Question requiring clarification — raised during Shift-Left QA review}

---

## Notes

{Additional context: known limitations, related stories, dependencies, edge cases found during exploration.}

**Jira link**: {ATLASSIAN_URL}/browse/BK-N
