---
type: quick
quick_id: 260407-n0u
description: Map Resend email delivery errors to proper HTTP response codes
metrics:
  duration: 3min
  completed: 2026-04-07T15:40:00Z
---

# Quick Task 260407-n0u: Map Resend Email Delivery Errors Summary

Resend email delivery errors now return structured HTTP 502 responses instead of generic 500s. The EmailSenderService throws HttpException with IResponse body containing EMAIL_DELIVERY_FAILED error code, the Resend error message, and actionable detail. The createHttpError utility passes HttpException instances through unchanged as the first check.

## Task 1: Add EMAIL_DELIVERY_FAILED error code and structured error responses (612bc03)

Added EMAIL_DELIVERY_FAILED = "EMAIL_0" to ErrorCodes enum. Updated EmailSenderService.send() to throw HttpException with HTTP 502 status and structured IResponse body containing error code, message from Resend, and detail explaining the upstream failure.

## Task 2: Add HttpException pass-through to createHttpError + tests (ec38e52, 9747b79)

Added HttpException instanceof check as the first condition in createHttpError so structured errors from EmailSenderService pass through with their original status code and body. Added 6 test cases covering all error type mappings.

## Commits

| Task | Commit | Description |
|------|--------|-------------|
| 1 | 612bc03 (trade-flow-api) | Add EMAIL_DELIVERY_FAILED error code and structured error responses |
| 2 | ec38e52 (trade-flow-api) | Add failing test for HttpException pass-through |
| 2 | 9747b79 (trade-flow-api) | Add HttpException pass-through to createHttpError utility |

## Self-Check: PASSED
