---
id: 260410-tkt
name: Review onboarding defect fixes and refactor for better holistic solution
mode: quick
created: 2026-04-10
completed: 2026-04-10
---

# Summary: Onboarding defect-fix review and holistic refactor

## Context

The debug session `.planning/debug/resolved/onboarding-skips-step-1.md` resolved four
defects in the new-user onboarding flow. All four fixes were user-verified, but the code
was still **staged** (uncommitted) in both repos. The user asked for a review against
project code standards and a holistic refactor before committing.

## Review outcome

The four defect fixes are technically correct. Three quality improvements were identified
against `trade-flow-api/CLAUDE.md` and `trade-flow-ui/CLAUDE.md`:

1. **API — Stripe customer leak** (debug doc's own known follow-up): the E11000 recovery
   path left an orphaned Stripe customer for every raced trial attempt.
2. **API — utility file convention**: the `isMongoDuplicateKeyError` helper lived at
   module scope inside the service, violating the "each utility file exports one
   function named to match the file" rule. The helper is also reusable — a prior
   debug session (`checkout-duplicate-subscription-key.md`) showed the same bug class
   in `SubscriptionCreator.createCheckoutSession`.
3. **UI — dead dep-sync machinery**: the `hasRunRef` guard made the pre-existing
   `depsRef` + dep-sync `useEffect` architecturally dead, and that dep-sync effect
   produced a pre-existing `react-hooks/exhaustive-deps` warning noted in the debug
   doc.

## Changes

### trade-flow-api

**New file**: `src/core/errors/is-mongo-duplicate-key-error.utility.ts`

- Exports a single `isMongoDuplicateKeyError(error: unknown): boolean` matching the
  project's one-function-per-utility convention.
- Hoists `MONGO_DUPLICATE_KEY_ERROR_CODE = 11000` out of the service.

**Edited**: `src/subscription/services/subscription-trial-creator.service.ts`

- Removed the module-scope `isMongoDuplicateKeyError` helper and `MONGO_DUPLICATE_KEY_ERROR_CODE`
  constant; imported the new utility via `@core/errors/is-mongo-duplicate-key-error.utility`.
- In the E11000 recovery branch, added a call to a new private `deleteOrphanStripeCustomer`
  method that deletes the losing Stripe customer via `stripe.customers.del(customer.id)`.
  The delete is wrapped in its own try/catch so a Stripe-side failure logs `.error` with
  full context (customer id, user id) but does not fail the request. The caller's DTO is
  already hydrated from the winning row, so orphan cleanup is best-effort.
- Updated the JSDoc on `create` to describe the complete race-and-recovery contract
  including the orphan cleanup step.

**Edited**: `src/subscription/test/services/subscription-trial-creator.service.spec.ts`

- Added `del: jest.fn()` to the Stripe mock's `customers` object.
- Expanded the existing "recover from E11000 duplicate key race" test to assert
  `mockStripe.customers.del` was called exactly once with the orphan customer id.
- Added a new regression test: "should still return the winning row when the orphan
  Stripe customer deletion itself fails" — stubs `stripe.customers.del` to reject and
  asserts the caller still receives the winning row without error.
- Updated the existing InternalServerError test to stub `customers.del` as well so the
  recovery path runs to completion.

### trade-flow-ui

**Edited**: `src/features/onboarding/components/SetupLoadingScreen.tsx`

- Deleted the runtime `depsRef` useRef and its dep-sync `useEffect`. Both became
  architecturally dead once `hasRunRef` ensured the setup effect runs exactly once per
  mount, and the dep-sync effect was responsible for a pre-existing
  `react-hooks/exhaustive-deps` warning.
- Kept `executeSetup` at module scope (not a `useCallback`): hoisting it out of the
  component body means the React Compiler lint rule `react-hooks/set-state-in-effect`
  treats the call site as opaque, so the mount effect is no longer flagged for
  "synchronous setState within an effect". A component-local `useCallback` version was
  tried first and rejected by lint.
- The mount `useEffect` now lists its actual dependencies
  (`[businessData, refetchBusinesses, refetchUser, createBusiness, startTrial, navigate]`)
  instead of `[]`, eliminating the exhaustive-deps warning. The `hasRunRef` guard still
  ensures the body runs at most once per mount — dep changes are intentionally ignored
  because RTK Query mutation handles and the router navigate function are stable across
  renders in practice, and `businessData` does not change post-mount in this component.
- Retry handler explicitly resets `hasError` and `phase` before calling `executeSetup`
  directly (bypassing the mount effect's ref guard), so Try-again remains functional.
- Introduced a `SetupDeps` interface to type the `executeSetup` argument (replaces the
  inline deps-bag type that the staged version had). Uses `Dispatch<SetStateAction<...>>`
  and `NavigateFunction` from their official types instead of hand-rolled signatures,
  improving type fidelity.
- Removed the unused `User` type import.
- Removed the unused `useCallback`/`useRef` imports left over from the earlier attempt
  (kept only what the final structure needs).

**No test changes needed**: the existing 10 tests (including the StrictMode regression
test) continue to pass unchanged because all mocks are at the module-boundary level
(`vi.mock("@/services", ...)` etc.), not at the `executeSetup` level.

### Quality gates

**trade-flow-api `npm run ci`**: PASSED
- 390 tests across 57 suites (+1 new test for Stripe delete failure recovery)
- 0 lint errors (24 pre-existing warnings unrelated to this task)
- Prettier clean
- Typecheck clean

**trade-flow-ui `npm run ci`**: PASSED
- 72 tests across 9 suites (no test changes)
- 0 lint errors (1 pre-existing warning: BusinessStep.tsx react-hook-form watch())
- Prettier clean
- Typecheck clean
- Resolved the pre-existing `react-hooks/exhaustive-deps` warning noted in the debug doc

## What was deliberately NOT changed

- **Switching from `upsertByStripeCustomerId` to `upsertByUserId`**: initially tempting
  as a race-safe alternative, but it would silently overwrite the winning caller's
  `stripeCustomerId` with the losing caller's on race, orphaning the first caller's
  Stripe subscription entirely. The current `upsertByStripeCustomerId` + E11000 catch
  pattern plus orphan Stripe customer cleanup is the correct trade-off.
- **Removing the defensive 422 fallback in `executeSetup`**: the backend is now
  guaranteed idempotent (returns 201 on both first-time and repeat calls), but the
  frontend/backend boundary is a genuine system boundary per CLAUDE.md ("only validate
  at system boundaries"). Belt-and-braces defence against future backend contract drift
  is reasonable and cheap to keep.
- **The `as BusinessTrade` cast in `executeSetup`**: pre-existing, at a form-data → API
  shape boundary. Not expanded, but also not reworked to a type guard in this pass to
  keep the diff focused on the defect-fix holism goals.

## Files touched

- `trade-flow-api/src/core/errors/is-mongo-duplicate-key-error.utility.ts` (new)
- `trade-flow-api/src/subscription/services/subscription-trial-creator.service.ts` (edit)
- `trade-flow-api/src/subscription/test/services/subscription-trial-creator.service.spec.ts` (edit)
- `trade-flow-api/src/auth/auth.guard.ts` (edit — defect #1, staged prior to this task)
- `trade-flow-ui/src/features/onboarding/components/SetupLoadingScreen.tsx` (edit)
- `trade-flow-ui/src/features/onboarding/components/__tests__/SetupLoadingScreen.test.tsx` (edit — staged prior to this task)
- `.planning/quick/260410-tkt-review-onboarding-defect-fixes-and-refac/260410-tkt-PLAN.md` (new)
- `.planning/quick/260410-tkt-review-onboarding-defect-fixes-and-refac/260410-tkt-SUMMARY.md` (new — this file)

## Commit strategy

Because this task spans two independent repos, the changes are committed as:

1. One atomic commit in `trade-flow-api` (containing original defect #1 fix, defect #3
   fix, plus the utility extraction and orphan Stripe customer cleanup).
2. One atomic commit in `trade-flow-ui` (containing original defect #2/#4 fix plus the
   SetupLoadingScreen simplification).
3. One docs commit in `trade-flow-docs` (PLAN.md, SUMMARY.md, STATE.md update).
