---
status: partial
phase: 36-public-landing-page-and-route-restructure
source: [36-VERIFICATION.md]
started: 2026-04-07T00:00:00Z
updated: 2026-04-07T00:00:00Z
---

## Current Test

[awaiting human testing]

## Tests

### 1. Landing Page Visual Rendering
expected: Full landing page renders with sticky header ("Trade Flow" logo, "Log in" outline button, "Start Free Trial" primary button); hero section with h1 "Run your trade business in one place" and "Start Free Trial" CTA; three feature cards; pricing card "Trade Flow Pro" showing GBP 6/month; footer with copyright year
result: [pending]

### 2. Start Free Trial CTA — Sign-Up Tab Pre-Selected
expected: Click "Start Free Trial" navigates to /login?mode=signup with "Sign Up" tab active (not "Sign In")
result: [pending]

### 3. Authenticated Redirect at Root URL
expected: Logged-in user navigating to root URL is immediately redirected to /dashboard without landing page flash
result: [pending]

### 4. Bundle Isolation — Separate LandingPage Chunk
expected: `npm run build` produces a separate JS chunk for LandingPage in dist/assets/, confirming React.lazy code-splitting
result: [pending]

## Summary

total: 4
passed: 0
issues: 0
pending: 4
skipped: 0
blocked: 0

## Gaps
