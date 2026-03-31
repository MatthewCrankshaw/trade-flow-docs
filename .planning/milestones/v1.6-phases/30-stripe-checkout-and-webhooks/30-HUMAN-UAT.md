---
status: testing
phase: 30-stripe-checkout-and-webhooks
source: [30-VERIFICATION.md]
started: 2026-03-29T00:00:00Z
updated: 2026-03-29T00:00:00Z
---

## Current Test

number: 2
name: Duplicate checkout guard
expected: |
  POST `/v1/subscription/checkout` for a user with `status: "trialing"` in MongoDB returns HTTP 422 with `SUBSCRIPTION_ALREADY_ACTIVE` error code
awaiting: user response

## Tests

### 1. Webhook signature validation
expected: POST to `/v1/webhooks/stripe` with a real Stripe-signed payload returns HTTP 200 and event appears in BullMQ queue; invalid signature returns 400
result: issue
reported: "HTTP 405 with empty response body"
severity: major

### 2. Duplicate checkout guard
expected: POST `/v1/subscription/checkout` for a user with `status: "trialing"` in MongoDB returns HTTP 422 with `SUBSCRIPTION_ALREADY_ACTIVE` error code
result: [pending]

### 3. verify-session fast path
expected: GET `/v1/subscription/verify-session?sessionId=cs_test_123` with a seeded local MongoDB record returns `{ data: [{ status: "trialing", verified: true }] }` without calling Stripe
result: [pending]

### 4. verify-session Stripe fallback
expected: GET `/v1/subscription/verify-session?sessionId=<unknown>` with no local record triggers Stripe API call and returns `payment_status` in response
result: [pending]

## Summary

total: 4
passed: 0
issues: 1
pending: 3
skipped: 0
blocked: 0

## Gaps

- truth: "POST to `/v1/webhooks/stripe` with a real Stripe-signed payload returns HTTP 200 and event appears in BullMQ queue; invalid signature returns 400"
  status: failed
  reason: "User reported: HTTP 405 with empty response body"
  severity: major
  test: 1
  root_cause: ""
  artifacts: []
  missing: []
  debug_session: ""
