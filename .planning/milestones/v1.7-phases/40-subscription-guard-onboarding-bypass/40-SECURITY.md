---
phase: 40
slug: subscription-guard-onboarding-bypass
status: verified
threats_open: 0
asvs_level: 1
created: 2026-04-07
---

# Phase 40 — Security

> Per-phase security contract: threat register, accepted risks, and audit trail.

---

## Trust Boundaries

| Boundary | Description | Data Crossing |
|----------|-------------|---------------|
| client -> API | HTTP requests cross from untrusted browser to API; JwtAuthGuard validates identity | Firebase JWT token (bearer) |
| subscription decorator -> guard | Metadata on controller methods signals guard to skip subscription check | Boolean metadata flag |

---

## Threat Verification

| Threat ID | Category | Disposition | Status | Evidence |
|-----------|----------|-------------|--------|----------|
| T-40-01 | Elevation of Privilege | accept | CLOSED | Accepted risk logged below. JwtAuthGuard present at method level (user.controller.ts:34). PATCH /v1/user/me is self-only by design. |
| T-40-02 | Elevation of Privilege | accept | CLOSED | Accepted risk logged below. JwtAuthGuard present at method level (business.controller.ts:63). One-business-per-user constraint enforced by BusinessPolicy in service layer. |
| T-40-03 | Tampering | mitigate | CLOSED | subscription.guard.spec.ts:135-175 — `describe("onboarding bypass routes")` block contains three test cases: PATCH bypass (line 136), POST bypass (line 151), non-regression ForbiddenError for POST without subscription (line 166). All 13 guard tests pass. |

---

## Accepted Risks Log

### T-40-01 — Elevation of Privilege: UserController.patch() with @SkipSubscriptionCheck

**Component:** `trade-flow-api/src/user/controllers/user.controller.ts` — `patch()` method (PATCH /v1/user/me)

**Risk:** A user with an expired or lapsed subscription can update their own user profile (display name, etc.) without an active subscription.

**Rationale:** This is intentional. The bypass is strictly scoped to the authenticated user's own record — no cross-user access is possible. `JwtAuthGuard` is applied to this route (line 34), so the caller must hold a valid Firebase JWT. The action (updating one's own name) is functionally low-risk and identical in privilege level to onboarding. Blocking this endpoint would make onboarding impossible for new users who have not yet created a subscription.

**Compensating Controls:**
- Authentication is required (JwtAuthGuard enforced).
- The endpoint operates only on `request.user` — the authenticated user's own record. No user ID is taken from the request body or URL params.
- Scope is limited to non-sensitive profile fields.

**Owner:** Product — accepted as low risk for v1.7 onboarding milestone.

---

### T-40-02 — Elevation of Privilege: BusinessController.create() with @SkipSubscriptionCheck

**Component:** `trade-flow-api/src/business/controllers/business.controller.ts` — `create()` method (POST /v1/business)

**Risk:** A user with an expired or lapsed subscription can create a business record without an active subscription.

**Rationale:** This is intentional. The bypass is required to allow new users to complete mandatory onboarding (creating a business is step one before a subscription can be established). `JwtAuthGuard` is applied to this route (line 63), so the caller must hold a valid Firebase JWT. The one-business-per-user constraint enforced by `BusinessPolicy` in the service layer prevents abuse — a user cannot exploit this endpoint to create multiple free businesses.

**Compensating Controls:**
- Authentication is required (JwtAuthGuard enforced).
- BusinessPolicy enforces one-business-per-user at the service layer, preventing repeated creation.
- Decorator is applied at method level only — all other `BusinessController` write methods (`update`) retain full subscription enforcement.

**Owner:** Product — accepted as low risk for v1.7 onboarding milestone.

---

## Unregistered Flags

None — SUMMARY.md contains no `## Threat Flags` section. No new attack surface was flagged during implementation.

---

## Scope

Files audited:

- `trade-flow-api/src/user/controllers/user.controller.ts`
- `trade-flow-api/src/business/controllers/business.controller.ts`
- `trade-flow-api/src/subscription/guards/subscription.guard.ts`
- `trade-flow-api/src/subscription/decorators/skip-subscription-check.decorator.ts`
- `trade-flow-api/src/subscription/test/guards/subscription.guard.spec.ts`

---

## Security Audit Trail

| Audit Date | Threats Total | Closed | Open | Run By |
|------------|---------------|--------|------|--------|
| 2026-04-07 | 3 | 3 | 0 | gsd-security-auditor |

---

## Sign-Off

- [x] All threats have a disposition (mitigate / accept / transfer)
- [x] Accepted risks documented in Accepted Risks Log
- [x] `threats_open: 0` confirmed
- [x] `status: verified` set in frontmatter

**Approval:** verified 2026-04-07
