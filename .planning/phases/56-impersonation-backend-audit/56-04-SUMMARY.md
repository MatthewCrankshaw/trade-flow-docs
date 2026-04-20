---
phase: 56-impersonation-backend-audit
plan: 04
subsystem: api
tags: [impersonation, subscription-guard, session-management, gap-closure]
dependency_graph:
  requires: [56-02]
  provides: [impersonation-subscription-bypass, stale-session-auto-termination]
  affects: [subscription-guard, impersonation-creator]
tech_stack:
  added: []
  patterns: [guard-bypass-via-request-augmentation, stale-session-auto-termination]
key_files:
  created: []
  modified:
    - trade-flow-api/src/subscription/guards/subscription.guard.ts
    - trade-flow-api/src/subscription/test/guards/subscription.guard.spec.ts
    - trade-flow-api/src/impersonation/services/impersonation-creator.service.ts
    - trade-flow-api/src/impersonation/test/services/impersonation-creator.service.spec.ts
decisions:
  - Bypass subscription check via request.impersonator presence (not permission check) -- impersonator field is only set after cryptographic JWT verification in JwtAuthGuard
  - Auto-terminate stale sessions instead of rejecting -- preserves audit trail via markEnded while allowing browser-refresh recovery
metrics:
  duration: 5m
  completed: 2026-04-20
---

# Phase 56 Plan 04: Gap Closure -- SubscriptionGuard Bypass and Stale Session Auto-Termination Summary

Fix two UAT blockers: SubscriptionGuard blocking impersonated requests, and ImpersonationCreator rejecting new sessions after browser refresh due to stale ACTIVE sessions.

## What Was Done

### Task 1: Add impersonation bypass to SubscriptionGuard (2c685c8)

Added `request.impersonator` check in `canActivate()` method, placed after the `!user` null check and before the subscription status DB lookup. When the impersonator field is present on the request (set by JwtAuthGuard after cryptographic JWT verification), the guard returns `true` immediately without checking subscription status.

Added two tests confirming impersonation bypass behavior and unchanged normal flow.

### Task 2: Auto-terminate stale ACTIVE sessions in ImpersonationCreator (b38493e)

Replaced the `ForbiddenError` throw for concurrent active sessions with auto-termination. When the creator finds existing ACTIVE sessions for the support user, it now calls `markEnded()` on each stale session with a warning log, then proceeds to create a new session normally.

Replaced the old "throws ForbiddenError for active session" test with two new tests: one confirming auto-termination calls `markEnded` and creates a new session, another confirming `markEnded` is not called when no active sessions exist.

## Deviations from Plan

None -- plan executed exactly as written.

## Out-of-Scope Issues Discovered

Three TypeScript compilation errors exist in the impersonation module, all introduced by a concurrent Phase 57 commit (`e9e6682`) that added `targetUser` to the impersonation response. The `IUserDto.name` field is `string | null` but `IImpersonationResponse.targetUser.name` expects `string`. These errors are not caused by Plan 56-04 changes and must be resolved in Phase 57.

## Threat Flags

None -- changes align with the plan's threat model. The `request.impersonator` bypass is safe because the field is only populated by JwtAuthGuard after cryptographic JWT verification and audit session status validation.

## Self-Check: PASSED
