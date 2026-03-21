---
status: partial
phase: 18-quote-email-sending
source: [18-VERIFICATION.md]
started: 2026-03-21T12:00:00Z
updated: 2026-03-21T12:00:00Z
---

## Current Test

[awaiting human testing]

## Tests

### 1. Send dialog pre-fill with template variables
expected: To is pre-filled with customer email; Subject shows resolved template (e.g., Quote Q-001 for Kitchen Renovation); Message body shows resolved text in the rich text editor.
result: [pending]

### 2. Rich text formatting in sent email
expected: The HTML sent to the backend contains `<strong>` and `<em>` tags. The email renders formatted text.
result: [pending]

### 3. No-email customer warning and save-email checkbox
expected: Warning alert appears for customer with no email. After send with save-email checked, customer.email is updated in the database.
result: [pending]

### 4. Business page settings flow through to send dialog
expected: Custom subject with {{quoteNumber}} saved on Business > Quote Email tab resolves in the send dialog Subject field.
result: [pending]

### 5. Re-send delivers new link with different token
expected: The second email contains a different token in the view URL (new token generated per send).
result: [pending]

### 6. SendGrid 503 for placeholder API key
expected: Response returns HTTP 503 with message: Email sending is not configured. Set a valid SENDGRID_API_KEY environment variable.
result: [pending]

## Summary

total: 6
passed: 0
issues: 0
pending: 6
skipped: 0
blocked: 0

## Gaps
