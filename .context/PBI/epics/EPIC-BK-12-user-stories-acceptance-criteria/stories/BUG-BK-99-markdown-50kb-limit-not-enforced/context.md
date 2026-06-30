# BK-99 — Context

## Ticket
BK-99: MarkdownEditor: Description: 50 KB size limit not enforced on submission
Priority: High | Status: Ready For QA | Epic: BK-12

## Session Notes
- Session started: 2026-06-10
- Parent story BK-16 is BLOCKED waiting for this fix to be verified
- Previous QA session (BK-16 Stage 2) found: counter goes red at >50KB but submit button stays ENABLED, server accepts and persists 51,000-char payloads
- Client partially fixed between Stage 2 and evidence re-capture: counter now shows red (text-signal-blocked) but submit NOT disabled

## Open Questions
- Does the fix address server-side validation (400 reject) or just client-side (button disable)?
- Is the textarea selector still `textarea[placeholder="Describe the story in Markdown."]`?

## Test Data
- Repro script: `const el = document.querySelector('textarea[placeholder="Describe the story in Markdown."]'); const setter = Object.getOwnPropertyDescriptor(window.HTMLTextAreaElement.prototype, 'value').set; setter.call(el, 'A'.repeat(51000)); el.dispatchEvent(new Event('input', { bubbles: true }))`
- Over-limit: 51,000 chars (51,000 bytes all ASCII)
- At-limit: 50,000 chars
- Under-limit: 49,000 chars
