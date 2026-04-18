# Stack Research — v1.9 Support & Admin Tools

**Domain:** SaaS admin tooling (user management, impersonation, role administration)
**Researched:** 2026-04-18
**Confidence:** HIGH

## Recommended Stack

### Core Technologies — No New Frameworks

The existing NestJS 11 + React 19 stack handles all v1.9 requirements without new frameworks. The only new package is `firebase-admin` for impersonation token generation and server-side user queries.

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| firebase-admin | 13.x | Custom token generation for impersonation, server-side user lookup | Only way to generate Firebase custom tokens server-side; required for impersonation; also provides `listUsers`/`getUser` for user management without querying Firebase from the client |
| NestJS Reflector + custom decorators | (built-in) | RBAC guard and `@Roles()` decorator | NestJS ships `Reflector` for metadata-driven guards; no extra packages needed; follows official NestJS authorization docs exactly |

### Supporting Libraries — Nothing New

| Library | Status | Purpose | Why No Addition Needed |
|---------|--------|---------|----------------------|
| @reduxjs/toolkit | Already installed | Impersonation state slice | Redux slice stores `impersonatingUserId`, `impersonatingUserName`, banner visibility |
| React Router 7 | Already installed | Support-only routes | Route guards already exist for auth/onboarding/paywall; add support role check |
| sonner | Already installed | Toast notifications for admin actions | Role grant/revoke confirmation toasts |
| lucide-react | Already installed | Icons for admin UI | Shield, UserCog, Eye icons for support features |
| Radix UI (Dialog, DropdownMenu, Tabs) | Already installed | User detail dialogs, action menus | Same component primitives used across entire app |

### Development Tools — No Additions

| Tool | Status | Notes |
|------|--------|-------|
| Jest + supertest | Already installed | Unit/integration tests for new guards, services, controllers |
| Vitest + Testing Library | Already installed | Component tests for admin dashboard, impersonation banner |

## Installation

```bash
# API only — single new package
cd trade-flow-api
npm install firebase-admin@13

# UI — nothing to install
# All required packages already present
```

## firebase-admin Integration Details

### Why firebase-admin Is Required

The current backend verifies Firebase JWTs using raw public key fetching (`firebase-key-provider.service.ts`). This is read-only verification. Impersonation requires **generating** custom tokens, which only the Firebase Admin SDK can do (`auth.createCustomToken(uid, claims)`).

### Service Account Credential Requirement

`firebase-admin` requires a service account JSON credential file. This is a new infrastructure requirement:

| Item | Detail |
|------|--------|
| Credential source | Firebase Console > Settings > Service Accounts > Generate New Private Key |
| Storage | `GOOGLE_APPLICATION_CREDENTIALS` env var pointing to JSON file, or inline `FIREBASE_SERVICE_ACCOUNT_JSON` env var |
| Permissions needed | "Service Account Token Creator" IAM role (default for Firebase service accounts) |
| Environment variable | Add `FIREBASE_SERVICE_ACCOUNT_JSON` to `.env` (base64-encoded JSON for Railway deployment) |

### Key Methods Used

```typescript
import { getAuth } from "firebase-admin/auth";

// Impersonation: generate a short-lived token for target user
const customToken = await getAuth().createCustomToken(targetUserId, {
  impersonatedBy: supportUserId,
  impersonationSessionId: sessionId,
});

// User management: list all users (paginated)
const listResult = await getAuth().listUsers(1000, nextPageToken);

// User management: get single user details
const userRecord = await getAuth().getUser(uid);

// Role administration: set custom claims
await getAuth().setCustomUserClaims(uid, { role: "support" });
```

## Impersonation Architecture Decision: API-Level vs Token-Level

### Option A: Token-Level Impersonation (Recommended)

Support user requests a custom Firebase token for the target user. Frontend calls `signInWithCustomToken()` to switch identity. All subsequent API requests naturally carry the target user's token.

**Pros:** Target user's exact experience reproduced; no API changes needed for impersonated requests; works with all existing guards and policies.

**Cons:** Requires `firebase-admin` + service account; frontend must manage session swap and restoration; custom token includes `impersonatedBy` claim for audit trail.

### Option B: API-Level Impersonation (Header-Based)

Support user sends own JWT but adds an `X-Impersonate-User: {targetUserId}` header. API middleware swaps `request.user` to the target user.

**Pros:** No `firebase-admin` needed; simpler infrastructure.

**Cons:** Does not reproduce exact user experience (frontend still shows support user's data until RTK Query refetches); requires modifying every policy/guard to respect impersonation header; audit trail harder to enforce consistently; potential security surface if header validation is incomplete.

### Recommendation: Token-Level (Option A)

Use token-level impersonation because:
1. The whole point is to "see exactly what the customer sees" -- token-level achieves this naturally
2. All existing guards, policies, and business scoping work without modification
3. The `impersonatedBy` custom claim provides a built-in audit trail in every request
4. `firebase-admin` also provides `listUsers`/`getUser` which the user management dashboard needs anyway

## RBAC Implementation — Built-In NestJS Patterns

No additional packages needed. The project already has:
- `IUserDto` with a `role` field
- `SubscriptionGuard` with `@SkipSubscriptionCheck` decorator pattern

Follow the same pattern for roles:

```typescript
// roles.decorator.ts
import { Reflector } from "@nestjs/core";
export const Roles = Reflector.createDecorator<string[]>();

// roles.guard.ts
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}
  canActivate(context: ExecutionContext): boolean {
    const roles = this.reflector.getAllAndOverride(Roles, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (!roles) return true;
    const { user } = context.switchToHttp().getRequest();
    return roles.some((role) => user.role === role);
  }
}
```

Apply globally via `APP_GUARD` (same as `SubscriptionGuard`), with `@Roles(["support"])` on support-only endpoints.

## User Management Dashboard — Existing Patterns Sufficient

The frontend already has:
- `DataView` pattern (table on desktop, cards on mobile) used by Customers, Items, Jobs, Quotes
- RTK Query for data fetching with cache invalidation
- `SearchableItemPicker` pattern for filtering
- Pagination support in API responses

The user management dashboard follows the same pattern. No data table library (TanStack Table, AG Grid) needed.

## Alternatives Considered

| Recommended | Alternative | When to Use Alternative |
|-------------|-------------|-------------------------|
| firebase-admin 13.x | API-level impersonation (X-Impersonate header) | Only if service account credentials are impossible to provision; loses "see what they see" fidelity |
| NestJS built-in Reflector RBAC | @nestjs/casl or casbin | Only if role model grows beyond 3-4 roles with complex resource-level permissions; current 2-role model (user/support) is too simple for CASL overhead |
| Existing DataView pattern | TanStack Table | Only if admin dashboard needs column sorting, resizing, grouping, virtual scrolling for 10k+ rows; current user base is small enough for simple pagination |
| Redux slice for impersonation state | React Context | Redux is already the state management solution; adding Context for one feature would be inconsistent |

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| @nestjs/casl / casbin | Massive overkill for 2-role system (user + support); adds complex policy DSL for a simple "is support?" check | NestJS Reflector + `@Roles()` decorator (3 files, ~40 LOC total) |
| TanStack Table | Adds 15KB+ dependency for a dashboard that lists dozens-to-hundreds of users, not thousands; existing table components already work | Existing `DataView` pattern with native HTML tables |
| react-admin / Refine | Full admin frameworks designed for CRUD-heavy admin panels; Trade Flow's admin is a small overlay on the existing app, not a separate admin app | Build within existing feature module structure (`src/features/support/`) |
| Separate admin SPA | Two frontends doubles maintenance; support features are 4-5 pages max | Support feature module within existing `trade-flow-ui` with route-level guards |
| Firebase client-side Admin SDK | Firebase Admin SDK is server-only; never expose service account credentials to the browser | `firebase-admin` on API only; frontend receives pre-generated custom tokens |

## Version Compatibility

| Package | Compatible With | Notes |
|---------|-----------------|-------|
| firebase-admin@13.x | Node.js 22.x | Requires Node 18+; Node 22 fully supported |
| firebase-admin@13.x | firebase@12.8.0 (client) | Admin SDK generates tokens consumed by client SDK `signInWithCustomToken()` |
| firebase-admin@13.x | @nestjs/core@11.x | Standard npm dependency; no NestJS-specific integration needed; wrap in NestJS service |

## Environment Variable Additions

| Variable | Repository | Purpose |
|----------|-----------|---------|
| `FIREBASE_SERVICE_ACCOUNT_JSON` | trade-flow-api | Base64-encoded Firebase service account JSON for firebase-admin initialization |

No frontend environment variables needed for this milestone.

## Sources

- [Firebase Admin Node.js SDK (Context7)](/firebase/firebase-admin-node) -- createCustomToken, setCustomUserClaims, listUsers API verified
- [NestJS Authorization Docs (Context7)](/nestjs/docs.nestjs.com) -- Reflector-based RBAC guard pattern, @Roles decorator
- [Firebase Custom Tokens Docs](https://firebase.google.com/docs/auth/admin/create-custom-tokens) -- Service account requirements, token creation
- [Firebase Custom Claims Docs](https://firebase.google.com/docs/auth/admin/custom-claims) -- setCustomUserClaims for role management
- [firebase-admin npm](https://www.npmjs.com/package/firebase-admin) -- Latest version 13.8.0 confirmed
- [User Impersonation for SaaS (Yaro Labs)](https://yaro-labs.com/blog/user-impersonation-tool-saas) -- Best practices: time-limited sessions, audit trail, banner UX
- [DEV Community Impersonation Guide](https://dev.to/akash_shukla/-how-i-built-a-secure-and-clean-user-impersonation-feature-reactjs-nodejs-40kn) -- Secure session management patterns

---
*Stack research for: v1.9 Support & Admin Tools*
*Researched: 2026-04-18*
