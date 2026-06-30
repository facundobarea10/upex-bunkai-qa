# BK-100 — Context

## Ticket
BK-100: MarkdownEditor: Description: 90% capacity warning threshold not implemented
Priority: Medium | Status: Ready For QA | Epic: BK-12

## Session Notes
- Session started: 2026-06-10
- Parent story BK-16 is BLOCKED — all linked bugs must close to unblock it
- Original bug: at 45,500 bytes counter shows neutral color (text-fg-4), no warning

## Open Questions
- AC6 says "warning at 90%" — does it specify amber/yellow? Or any visual change?
- Is the warning supposed to be just color or also badge/tooltip?

## Test Data
- 90% threshold: 45,000 chars (45,000 bytes all ASCII) = exactly 90% of 50,000
- 90%+ boundary: 45,500 chars (original repro)
- Below threshold: 44,000 chars (~88%)
- At blocked: 51,000 chars (separate state)
- Inject script: `const el = document.querySelector('textarea[placeholder="Describe the story in Markdown."]'); const setter = Object.getOwnPropertyDescriptor(window.HTMLTextAreaElement.prototype, 'value').set; setter.call(el, 'A'.repeat(45500)); el.dispatchEvent(new Event('input', { bubbles: true }))`
