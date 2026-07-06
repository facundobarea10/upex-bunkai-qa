# [BK-N] {Bug Title}

> **Type**: Bug | Defect | **Project**: BK — Bunkai TMS  
> **Status**: {Open | In Progress | Ready For QA | In Review | Closed | REJECTED | Duplicated | Deferred | Cannot Reproduce | Enhancement | ABORTED}  
> **Severity**: {Critical | High | Medium | Low}  
> **Priority**: {Highest | High | Medium | Low | Lowest}  
> **Assignee**: {name or unassigned}  
> **Reported by**: {reporter name}  
> **Environment**: {local | staging | production}  
> **Labels**: {labels}

---

## Summary

{One sentence describing the bug — what went wrong and where.}

---

## Steps to Reproduce

1. {Precondition: ensure X exists / user is logged in as Y}
2. {Action: navigate to / click / submit}
3. {Action: enter input / select option}
4. {Action: click submit / trigger event}

---

## Expected Behavior

{What should happen — reference the AC or business rule this violates.}

---

## Actual Behavior

{What actually happens — be specific: error message text, wrong data, UI state.}

---

## Context

**Feature area**: {ATC editor | Auth | Workspace management | Invite flow | etc.}  
**Related story**: BK-{N} — {story title}  
**Related AC**: AC{N} of BK-{story-N}  
**Route/URL**: {the URL where the bug occurs}  
**API endpoint**: {if API-level bug: method + path}  
**DB table**: {if data integrity bug: table name}

---

## Evidence

| Type | Description | Path |
|---|---|---|
| Screenshot | {what it shows} | `evidence/screenshot-N.png` |
| Network trace | {request/response captured} | `evidence/trace.har` |
| Console error | {error message text} | — |
| API response | {status code + body} | — |

---

## Business Impact

**Severity**: {Critical — blocks core flow | High — major feature broken | Medium — workaround exists | Low — cosmetic}  
**Users affected**: {All users | Workspace admins | Viewers | CLI agents}  
**Frequency**: {Always | Intermittent — describe trigger | Rare}

---

## Root Cause (if known)

{Technical explanation of why this happens — file/line reference if identified.}

---

## Fix Notes (for developer)

{Optional: suggested fix approach, related code areas to look at.}

---

**Jira link**: {ATLASSIAN_URL}/browse/BK-N
