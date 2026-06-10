# BK-16 — Test Session Memory

> Shared state across the 4 sub-agent dispatches (Session Start → Stage 1 → Stage 2 → Stage 3).
> Updated by each stage. Never commit to Jira.

## Identity
- **Ticket**: BK-16
- **Epic**: EPIC-BK-12-user-stories-acceptance-criteria
- **Story folder**: STORY-BK-16-markdown-editor-write-and-preview-markdown-safely
- **TMS Modality**: jira-native
- **Active environment**: staging
- **WEB_URL**: https://staging-upexbunkai.vercel.app
- **API_URL**: https://staging-upexbunkai.vercel.app/api

## Stage Progress
- [x] Session Start — completed
- [x] Stage 1 — Planning (ATP) — completed 2026-05-31
- [x] Stage 2 — Execution — completed 2026-06-09
- [x] Stage 3 — Reporting (in progress)

## Smoke Test Result
**GO** — 2026-06-09
- Login succeeded via cookie-injected Supabase session (bunkai-qa-stage2@vezaakarii.resend.app)
- Workspace "BK-16 QA Testing" created, project "Markdown Editor Test" created
- User Story form opened with description textarea (`data-testid="user-story-description-textarea"`, placeholder "Describe the story in Markdown.")
- Textarea accepts Markdown input, Preview toggle works, KB size counter present
- Feature IS deployed to staging

## Bugs Found
- **BUG-01** (High) — TC-06: 50 KB description size limit not enforced end-to-end. A 51,000-byte description was accepted and saved to DB (51,000 bytes confirmed). Client-side counter DOES change to `text-signal-blocked` (red) at >50KB, but the submit button remains ENABLED and the server accepts/persists the payload with no error. AC5 violated — submission not blocked. (Note: partial client-side indicator added between Stage 2 and evidence re-capture; core bug persists.)
- **BUG-02** (Medium) — TC-07: 90% size warning threshold not implemented. At 45,500 bytes (>90% of 50 KB limit), the `markdown-size` counter shows neutral color (`text-fg-4`) with no warning state. No progress bar, tooltip, or color change appears. Size counter displays value in KiB (44.4 KB for 45,500 bytes), not true KB.

## Key Findings
- Server-side sanitizer (`sanitize-html`) is active and correctly strips `<script>` tags, `javascript:` hrefs, and event handlers (onclick, onmouseover) before DB write.
- DB stores raw Markdown source (not HTML). Markdown syntax (##, -, |) and newlines preserved as-is.
- Client-side renderer (`react-markdown` + `rehype-sanitize`) also strips all dangerous content in preview.
- Preview toggle button: `data-testid="md-tool-preview"`. Description textarea: `data-testid="user-story-description-textarea"`. Submit: `data-testid="user-story-submit"`. Cancel: `data-testid="user-story-cancel"`. Size counter: `data-testid="markdown-size"`.
- Auth: App uses Supabase magic link. Token refresh via cookie injection works for testing. Refresh token in `.playwright/user-data` localStorage persists across sessions.
- DB connection: PostgreSQL via `pg` npm package works directly with `.env` DBHUB_* vars (no DBHub MCP needed).
- The user story in-tree expand uses `story-row-<uuid>` testid; edit button uses `story-edit-<uuid>`; new story button is hover-only on module row.

Evidence capture run completed 2026-06-09. Files: tc-06-pre-submit.png, tc-06-no-error.png, tc-06-after-reload.png, tc-07-no-warning.png, tc-07-counter-detail.png, tc-04-result.png (refreshed), tc-05-result.png (refreshed), tc-09-result.png (refreshed), tc-06-db.txt (existing from Stage 2).

## ATP Reference
Written to Jira `customfield_10120` (Acceptance Test Plan) on 2026-05-31.
Local materialized file: `acceptance-test-plan.md` (via jira:sync-issues).
Risk score: 13/HIGH. Test outlines: 9 (5 ACs + 4 edge cases).

## ATR Reference
_Not yet created. Will be written to Jira `acceptance_test_results` field after Stage 3._
