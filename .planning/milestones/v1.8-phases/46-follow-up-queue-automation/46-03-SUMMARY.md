---
phase: 46-follow-up-queue-automation
plan: 03
subsystem: email
tags: [maizzle, email-template, estimate-followup, nestjs]

requires:
  - phase: 44-email-send-flow
    provides: EmailModule with Maizzle render pattern and estimate-sent.html template
provides:
  - EstimateFollowupEmailRenderer service for rendering follow-up emails
  - estimate-followup.html Maizzle template with 3 CTAs and followupStep copy variations
affects: [46-04, 46-05]

tech-stack:
  added: []
  patterns: [followup-step-conditional-template-copy]

key-files:
  created:
    - trade-flow-api/src/email/templates/estimate-followup.html
    - trade-flow-api/src/email/services/estimate-followup-email-renderer.service.ts
    - trade-flow-api/src/estimate-followups/test/services/estimate-followup-email-renderer.service.spec.ts
  modified:
    - trade-flow-api/src/email/email.module.ts

key-decisions:
  - "Used followupStep-based conditional blocks (is3d/is10d/is21d) in template with regex replacement in renderer, matching existing estimate-response.html pattern"
  - "Registered EstimateFollowupEmailRenderer in EmailModule providers and exports for Plan 04 processor injection"

metrics:
  duration: 4min
  completed: 2026-04-14
  tasks: 1
  files: 4
---

# Phase 46 Plan 03: Estimate Follow-up Email Template Summary

Maizzle follow-up email template with 3d/10d/21d copy variations, View Estimate button, three response CTAs (Go Ahead, Message [FirstName], Not right for me), and non-binding disclaimer rendered by EstimateFollowupEmailRenderer service.

## What Was Built

### Maizzle Template (estimate-followup.html)
- Business name header with follow-up body copy that varies by `followupStep` (3d/10d/21d)
- Estimate summary card showing estimate number and price display
- Primary CTA: "View Estimate" button linking to the token URL
- Three response CTAs: "Go Ahead" (green button, #respond), "Message [FirstName]" (outlined button, #message), "Not right for me" (text link, #decline)
- Non-binding disclaimer in amber callout box
- "Sent via Trade Flow" footer
- MSO/VML fallbacks for Outlook compatibility

### Renderer Service (EstimateFollowupEmailRenderer)
- `EstimateFollowupEmailData` interface with businessName, followupStep, estimateNumber, viewUrl, traderFirstName, priceDisplay
- `FollowupStep` type export ("3d" | "10d" | "21d")
- HTML escaping for all user-supplied text variables (XSS prevention via T-46-04 mitigation)
- URL variables embedded as href attributes only (no raw output)
- Maizzle framework dynamic import with graceful fallback to pre-styled HTML

### Unit Tests (15 passing)
- Template variable injection: estimateNumber, viewUrl, businessName, priceDisplay, traderFirstName
- CTA presence: Go Ahead, Message [FirstName], Not right for me
- Disclaimer text presence
- Follow-up step copy variations: 3d ("checking in"), 10d ("Following up"), 21d ("one more time")
- XSS escaping for businessName
- CTA href anchors (#respond, #decline)
- Footer presence

## Task Commits

Each task was committed atomically:

1. **Task 1 RED:** Failing tests for EstimateFollowupEmailRenderer - `133b174` (test)
2. **Task 1 GREEN:** Template, renderer service, and EmailModule registration - `1b1b3f8` (feat)

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 2 - Missing critical functionality] Registered renderer in EmailModule**
- **Found during:** Task 1
- **Issue:** Plan did not explicitly mention registering the renderer in EmailModule, but Plan 04 will need to inject it
- **Fix:** Added EstimateFollowupEmailRenderer to EmailModule providers and exports
- **Files modified:** trade-flow-api/src/email/email.module.ts
- **Commit:** 1b1b3f8

## Verification Results

All verification checks passed:
- `grep "Go Ahead" estimate-followup.html` -- found
- `grep "This is an estimate" estimate-followup.html` -- found
- `npx jest --testPathPatterns=estimate-followup-email-renderer` -- 15/15 passed, exit 0

## Self-Check: PASSED

- [x] trade-flow-api/src/email/templates/estimate-followup.html exists
- [x] trade-flow-api/src/email/services/estimate-followup-email-renderer.service.ts exists
- [x] trade-flow-api/src/estimate-followups/test/services/estimate-followup-email-renderer.service.spec.ts exists
- [x] Commit 133b174 exists
- [x] Commit 1b1b3f8 exists
