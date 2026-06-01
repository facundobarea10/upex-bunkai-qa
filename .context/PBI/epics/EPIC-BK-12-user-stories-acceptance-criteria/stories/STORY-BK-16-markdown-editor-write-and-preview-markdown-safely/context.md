# BK-16 — Session Context

> Hand-authored. NOT a Jira mirror. Session notes, open questions, decisions made during testing.

## Ticket
- **Key**: BK-16
- **Title**: Markdown Editor | Write and preview Markdown safely
- **Epic**: BK-12 — User Stories & Acceptance Criteria
- **Status at session start**: Shift-Left QA
- **TMS Modality**: jira-native (ATP/ATR stored as custom fields on Story)
- **Environment**: staging — https://staging-upexbunkai.vercel.app

## Open Questions

1. **Is this feature deployed to staging?** Story is in Shift-Left QA (pre-dev). Smoke test will confirm.
2. **Where exactly does the editor appear?** Per architect annotation — US description form and AC description form. Need to navigate to User Story edit page in staging to find the editor surface.
3. **Is `sanitize-html` already wired server-side?** Check in `../upex-bunkai-tms/` source if the sanitizer is active.
4. **50KB size cap**: client warns at 90% (45KB), hard stops at 100% (50KB). Both behaviors need testing.

## Session Notes

_Empty at session start. Updated during Stage 2 execution._

## Decisions Made

- `/sprint-testing` selected over `/shift-left-testing` — user chose this flow to understand the full end-to-end testing pipeline.
- Story status is Shift-Left QA (pre-dev) — feature may not be deployed yet. Smoke test is the gate.
