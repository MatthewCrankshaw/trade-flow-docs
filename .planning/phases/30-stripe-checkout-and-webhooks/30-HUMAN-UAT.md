---
status: partial
phase: 30-stripe-checkout-and-webhooks
source: [30-VERIFICATION.md]
started: 2026-03-29T00:00:00Z
updated: 2026-03-29T00:00:00Z
---

## Current Test

[awaiting human testing]

## Tests

### 1. Webhook signature validation
expected: POST to `/v1/webhooks/stripe` with a real Stripe-signed payload returns HTTP 200 and event appears in BullMQ queue; invalid signature returns 400
result: [pending]

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
issues: 0
pending: 4
skipped: 0
blocked: 0

## Gaps
