---
id: 260410-tkt
name: Review onboarding defect fixes and refactor for better holistic solution
mode: quick
created: 2026-04-10
---

# Plan: Review onboarding defect fixes and holistic refactor

## Context

The debug session at `.planning/debug/resolved/onboarding-skips-step-1.md` resolved four
defects in the new-user onboarding flow (auth name default, RTK cache race, subscription
E11000, StrictMode double-fire). All four fixes are user-verified and the code is currently
**staged** (not yet committed) in both repos:

- `trade-flow-api`: `src/auth/auth.guard.ts`, `src/subscription/services/subscription-trial-creator.service.ts`,
  `src/subscription/test/services/subscription-trial-creator.service.spec.ts`
- `trade-flow-ui`: `src/features/onboarding/components/SetupLoadingScreen.tsx`,
  `src/features/onboarding/components/__tests__/SetupLoadingScreen.test.tsx`

## Review against project standards

After reviewing the staged diffs against each project's CLAUDE.md, three opportunities
for a cleaner, more holistic solution exist:

### 1. API — Stripe customer leak in E11000 recovery path

The debug doc explicitly calls this out as a known follow-up:

> SubscriptionTrialCreator E11000 recovery path leaks one Stripe customer when a genuine
> network-retry race occurs in production.

With the `hasRunRef` guard, the StrictMode trigger is gone, so in practice the leak is
only exposed by genuine network retries (double-click, router remount, explicit retry).
But it is still a real side effect on a real third-party system. Cleanup is cheap: delete
the orphaned Stripe customer in the recovery branch before returning the winning row.
Failure of the delete itself must not fail the request (log warning, still return).

### 2. API — `isMongoDuplicateKeyError` helper should be a shared utility

Per `trade-flow-api/CLAUDE.md`:

> Utility Files — Each utility file exports exactly one function; function name must match
> file name converted to camelCase.

The helper currently lives at module scope inside `subscription-trial-creator.service.ts`.
The pattern is reusable — the existing debug knowledge base
(`checkout-duplicate-subscription-key.md`) documents an earlier defect with exactly the
same bug class in `SubscriptionCreator.createCheckoutSession`. Promoting this to
`src/core/errors/is-mongo-duplicate-key-error.utility.ts` follows the naming convention
and makes the guard importable from any service facing unique-index races.

### 3. UI — Dead dep-sync effect and module-scoped `executeSetup`

The staged UI fix introduced `hasRunRef` to prevent double-invocation under StrictMode.
This correctly solves the race, but it also makes two earlier pieces of machinery
architecturally dead:

- The **dep-sync `useEffect`** that re-writes `depsRef.current` on every render. Because
  `runSetup` now runs exactly once per mount, later renders of this effect can never
  affect execution. The debug doc explicitly notes the pre-existing exhaustive-deps warning
  this effect produces.
- The **module-scoped `executeSetup` function** that takes 8 dependencies as arguments.
  This indirection has no testing benefit — the tests mock at the `@/services` and
  `@/features/subscription/api/subscriptionApi` module boundaries, not by stubbing
  `executeSetup` itself.

Inlining the setup logic as a `useCallback` inside the component:
- Eliminates `depsRef` and the dep-sync effect (and the exhaustive-deps warning).
- Removes an 8-arg "deps bag" parameter that obscures the flow.
- Keeps tests passing unchanged (module-boundary mocks still work).
- Keeps the `hasRunRef` guard as the one-line idempotency barrier against StrictMode.

## Scope boundary

- **IN SCOPE**: Improvements 1, 2, 3 above — direct quality wins with low risk.
- **OUT OF SCOPE**: Changing from `upsertByStripeCustomerId` to `upsertByUserId`.
  Initially tempting, but flipping the upsert key would let racing calls silently
  overwrite each other's `stripeCustomerId`, orphaning the first caller's Stripe
  subscription entirely. The current pattern + Stripe cleanup on race is the
  correct trade-off.
- **OUT OF SCOPE**: Removing the defensive 422 fallback in `SetupLoadingScreen`.
  The backend-frontend boundary is a genuine system boundary, so belt-and-braces
  defence against backend contract changes is reasonable.

## Tasks

### Task 1 — Extract `isMongoDuplicateKeyError` to a shared utility

**Repo**: trade-flow-api
**Files**:
- CREATE `src/core/errors/is-mongo-duplicate-key-error.utility.ts`
- EDIT `src/subscription/services/subscription-trial-creator.service.ts`

**Actions**:
1. Create the utility file with one exported function matching the file name in camelCase.
   Use the same type-guard shape already in the service. Include a JSDoc explaining when
   MongoDB raises code 11000 and why a shared helper isolates the magic number.
2. Remove the module-scope helper + `MONGO_DUPLICATE_KEY_ERROR_CODE` constant from
   `subscription-trial-creator.service.ts`.
3. Import `isMongoDuplicateKeyError` from `@core/errors/is-mongo-duplicate-key-error.utility`.

**Verify**: `npm run ci` in trade-flow-api passes.

### Task 2 — Close the Stripe customer leak in the E11000 recovery path

**Repo**: trade-flow-api
**Files**:
- EDIT `src/subscription/services/subscription-trial-creator.service.ts`
- EDIT `src/subscription/test/services/subscription-trial-creator.service.spec.ts`

**Actions**:
1. In the `catch (error)` block, after confirming `isMongoDuplicateKeyError(error)` and
   re-fetching the winning row, attempt to delete the losing Stripe customer.
2. Wrap `stripe.customers.del(customer.id)` in its own try/catch so a delete failure logs
   a warning and does NOT fail the request (the caller's subscription DTO is already
   hydrated from the winning row). Log at `.warn` for the attempt, `.error` for the
   delete failure (matching the existing AppLogger usage pattern in the service).
3. Update the existing "should recover from E11000 duplicate key race" test to assert
   `mockStripe.customers.del` was called with the orphan customer id.
4. Add a new test: "should not fail the request if the orphan Stripe customer deletion
   itself fails" — arrange `stripe.customers.del` to reject, assert result still equals
   `racedExisting`.

**Verify**: `npm run ci` in trade-flow-api passes; new test asserts cleanup path.

### Task 3 — Inline `executeSetup` into SetupLoadingScreen and remove dead dep-sync effect

**Repo**: trade-flow-ui
**Files**:
- EDIT `src/features/onboarding/components/SetupLoadingScreen.tsx`

**Actions**:
1. Delete the module-scope `executeSetup` function and its deps interface.
2. Define `runSetup` as a `useCallback` inside the component. Include all closed-over
   hook values in the dependency array (`businessData`, `createBusiness`, `startTrial`,
   `refetchBusinesses`, `refetchUser`, `navigate`). RTK Query mutation handles and
   `useNavigate`'s result are stable across renders, so in practice `runSetup` is stable
   until `businessData` changes — which it does not after mount in this component.
3. Delete `depsRef` and the dep-sync `useEffect` entirely.
4. Keep the `hasRunRef` useRef and the mount `useEffect` — the effect now depends on
   `runSetup` and remains guarded by `hasRunRef`, so StrictMode double-invocation is
   still blocked and there is no exhaustive-deps warning.
5. `handleRetry` calls `runSetup()` directly (no `hasRunRef` fiddling needed since the
   mount effect already flipped the ref to true).
6. Remove the `User` type import — it was only used to type the `refetchUser` parameter
   of the module-scope function. Inline narrowing via optional chaining works with
   RTK Query's inferred types.
7. Use `unknown[]` narrowing (not `as` casts) — preserve the existing `as BusinessTrade`
   cast since that matches the component's pre-existing style and the CLAUDE.md rule
   permits type assertions at "third-party SDK type limitations"; the BusinessTrade cast
   is at a form-data → API shape boundary and is pre-existing. Do not expand the use of
   `as` beyond what was already there.

**Verify**: `npm run ci` in trade-flow-ui passes; all 10 existing tests continue to pass
without modification.

## must_haves

### Truths
- Backend's E11000 recovery path in `SubscriptionTrialCreator.create` deletes the
  orphan Stripe customer (with defensive try/catch around the delete itself).
- `isMongoDuplicateKeyError` lives in `src/core/errors/is-mongo-duplicate-key-error.utility.ts`
  and is consumed by `subscription-trial-creator.service.ts` via an `@core/errors/*`
  import (per project path-alias convention).
- `SetupLoadingScreen` has no module-scope `executeSetup`, no `depsRef`, and no dep-sync
  `useEffect`. The `hasRunRef` guard and mount `useEffect` remain.
- All existing tests in both repos continue to pass; new API tests cover the Stripe
  customer cleanup path (both success and failure branches).

### Artifacts
- `trade-flow-api/src/core/errors/is-mongo-duplicate-key-error.utility.ts` (new)
- `trade-flow-api/src/subscription/services/subscription-trial-creator.service.ts` (edited)
- `trade-flow-api/src/subscription/test/services/subscription-trial-creator.service.spec.ts` (edited)
- `trade-flow-ui/src/features/onboarding/components/SetupLoadingScreen.tsx` (edited)
- `.planning/quick/260410-tkt-review-onboarding-defect-fixes-and-refac/260410-tkt-PLAN.md` (this file)
- `.planning/quick/260410-tkt-review-onboarding-defect-fixes-and-refac/260410-tkt-SUMMARY.md` (created after execution)

### Key links
- Debug record: `.planning/debug/resolved/onboarding-skips-step-1.md`
- API CLAUDE.md: `trade-flow-api/CLAUDE.md` (utility file + TypeScript standards)
- UI CLAUDE.md: `trade-flow-ui/CLAUDE.md` (tech stack + hooks rules)
