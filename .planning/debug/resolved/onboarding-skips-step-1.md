---
status: resolved
trigger: "onboarding-skips-step-1: After registration, onboarding lands directly on 'step 2 of 2' instead of 'step 1 of 2'"
created: 2026-04-10T00:00:00Z
updated: 2026-04-10T00:10:00Z
---

## Current Focus

hypothesis: RESOLVED — all four defects confirmed fixed end-to-end by the end user on 2026-04-10.
test: User re-tested onboarding with a brand-new registration and reported "confirmed fixed". Step 1 → step 2 → trial → dashboard flow now works with exactly one POST /v1/business and one POST /v1/subscription/trial per submission, a single businessId attached to the user, and a single subscription row.
expecting: Nothing further. Session archived to .planning/debug/resolved/ and appended to knowledge base.
next_action: None. Known follow-up (out of scope): SubscriptionTrialCreator E11000 recovery path leaks one Stripe customer when a genuine network-retry race occurs in production. Recommended cleanup is either deleting the losing Stripe customer in the recovery branch or creating the Mongo row first and attaching the Stripe customer after.

## Symptoms

expected: After a brand-new user registers and is logged in, the onboarding flow should begin at step 1 of 2, then progress to step 2 of 2 (business name and primary trade).
actual: After registration and automatic login, the user lands directly on the "Tell us about your business" page showing "step 2 of 2" — step 1 of 2 is not rendered at all.
errors: None reported. Investigate browser console/network during render.
reproduction:
1. Go to register page in trade-flow-ui
2. Enter new user credentials and click Register
3. User is created and auto-logged in
4. Observe: onboarding page appears, displaying "step 2 of 2" instead of "step 1 of 2"
started: v1.7 "Onboarding & Landing Page" milestone (shipped 2026-04-07)

## Eliminated

## Evidence

- timestamp: 2026-04-10
  checked: trade-flow-ui src/features/onboarding/components/OnboardingWizard.tsx
  found: Wizard renders progress as `currentStep={step === "profile" ? 1 : 2}`. Initial step comes from useOnboardingStep().initialStep ("profile" | "business").
  implication: If initialStep is "business" the user lands on step 2 immediately. Need to find what makes initialStep "business".

- timestamp: 2026-04-10
  checked: trade-flow-ui src/features/onboarding/hooks/useOnboardingStep.ts
  found: Logic is `if (hasName && !hasBusiness) return "business"; else return "profile";`. hasName uses `Boolean(user?.name?.trim())`.
  implication: A brand-new user's `user.name` is non-empty/non-whitespace. Need to check API source of `user.name`.

- timestamp: 2026-04-10
  checked: trade-flow-ui src/types/api.types.ts User interface and src/features/auth/components/AuthForm.tsx signup
  found: `User.name: string | null`. AuthForm signup only calls Firebase `updateProfile` with displayName when the optional Name field is filled in. Otherwise displayName is unset on the Firebase user.
  implication: With the optional Name field empty, the Firebase JWT will not have a `name` claim. Backend must be the one populating `user.name`.

- timestamp: 2026-04-10
  checked: trade-flow-api src/auth/auth.guard.ts JwtAuthGuard.canActivate (lines 56-67)
  found: Newly-created users get `name: userInfo.name || userInfo.displayName || userInfo.email`. When the JWT has no name claim (email/password registration without setting displayName), `userInfo.name` and `userInfo.displayName` are both undefined and the fallback uses `userInfo.email`.
  implication: The backend persists the user with `name = email`, never null, for any email/password signup that does not pre-set the Firebase displayName.

- timestamp: 2026-04-10
  checked: trade-flow-api src/user/repositories/user.repository.ts findOrCreate + mapToDto
  found: `findOrCreate` writes `name: userData.name ?? null` on insert; `mapToDto` returns it back as-is. So the email-as-name persists into the DTO/response.
  implication: The /v1/user/me response will include `name = "user@example.com"` for any newly registered user, making `hasName` true on the frontend.

- timestamp: 2026-04-10
  checked: trade-flow-api src/user/controllers/user.controller.ts mapToResponse
  found: `name: user.name` is forwarded directly to the IUserResponse with no transformation.
  implication: The end-to-end chain is confirmed: Firebase signup (no displayName) -> JWT has no name claim -> backend defaults `user.name` to email -> /v1/user/me returns name=email -> frontend useOnboardingStep sees hasName=true -> wizard initial step = "business" -> user lands on step 2 of 2.

- timestamp: 2026-04-10 (defect #2)
  checked: User report — after step 2 submission, the success toast "You're all set! Your 30-day free trial has started." fires but the user stays on the step 2 page.
  found: Fix #1 now lets step 1 render correctly. Step 1 completes, step 2 submits business data, then the UI stays on what looks like step 2 of the wizard.
  implication: Either navigate is never called, the navigate target is wrong, or something is redirecting the user back to /onboarding showing the business step.

- timestamp: 2026-04-10 (defect #2)
  checked: trade-flow-ui src/features/onboarding/components/OnboardingWizard.tsx
  found: Wizard transitions internal state from "business" to "loading" upon BusinessStep.onComplete. When step === "loading" it renders SetupLoadingScreen, NOT BusinessStep.
  implication: The "step 2 page" the user sees cannot be OnboardingWizard's own business step state — it must be the wizard being re-mounted from a redirect back to /onboarding, where useOnboardingStep decides the initial step again.

- timestamp: 2026-04-10 (defect #2)
  checked: trade-flow-ui src/features/onboarding/components/SetupLoadingScreen.tsx — executeSetup function
  found: Flow is: `refetchBusinesses()` → `createBusiness(...).unwrap()` → `startTrial().unwrap()` → `toast.success(...)` → `navigate("/dashboard")`. No explicit refetch of /v1/user/me anywhere. It relies solely on RTK Query's tag invalidation (createBusiness invalidates ["Business", "User"]).
  implication: The navigate call runs synchronously after mutations resolve. The User query refetch triggered by tag invalidation is fire-and-forget — the component does not wait on it.

- timestamp: 2026-04-10 (defect #2)
  checked: trade-flow-ui src/features/business/api/businessApi.ts createBusiness mutation
  found: `invalidatesTags: ["Business", "User"]`. Confirms tag invalidation is wired for both Business and User queries.
  implication: RTK Query will queue a refetch of /v1/user/me when createBusiness resolves. However the refetch is asynchronous — navigate fires before it lands.

- timestamp: 2026-04-10 (defect #2)
  checked: trade-flow-ui src/hooks/useCurrentBusiness.ts
  found: Uses `const { data: user, isLoading: userLoading } = useGetCurrentUserQuery(); ... const businessId = user?.businessRoles[0]?.businessId || ""; const hasBusiness = Boolean(businessId); const isLoading = userLoading || businessesLoading;`. No use of `isFetching`.
  implication: During a background refetch (post-invalidation), `isLoading` stays false and `user` remains the PREVIOUS cached response. So `hasBusiness` is computed from STALE data. This is the core bug.

- timestamp: 2026-04-10 (defect #2)
  checked: trade-flow-ui src/features/auth/components/OnboardingGuard.tsx
  found: `if (hasDisplayName && hasBusiness) return <Outlet />; if (isOnboardingPath) return <Outlet />; return <Navigate to="/onboarding" replace />;`. Redirect fires if EITHER hasDisplayName OR hasBusiness is false.
  implication: On the first render after navigate("/dashboard"), useCurrentBusiness returns stale (empty) businessRoles even though the User query is being refetched — guard redirects back to /onboarding.

- timestamp: 2026-04-10 (defect #2)
  checked: trade-flow-ui src/features/onboarding/hooks/useOnboardingStep.ts
  found: `if (hasName && !hasBusiness) return { initialStep: "business" }; else return { initialStep: "profile" }`. initialStep is consumed once by OnboardingWizard via `useState(initialStep)` — not reactive.
  implication: When the redirected /onboarding remounts, user.name is set and (briefly) businesses[] is stale → initialStep resolves to "business". Wizard captures this in useState and sticks on step 2 even after the User refetch lands.

- timestamp: 2026-04-10 (defect #2)
  checked: trade-flow-api src/business/services/business-creator.service.ts BusinessCreator.create and BusinessRoleAssigner.ensureBusinessAdministratorRole
  found: The backend awaits `setBusinessRoleIds` BEFORE POST /v1/business responds. The next call to /v1/user/me will hydrate businessRoles from businessRoleIds via UserRetriever.getByExternalAuthUserId (which calls businessRoleRepository.findByIds).
  implication: The backend is NOT the problem. Data is fully committed by the time POST /v1/business resolves. The issue is purely the frontend race between the invalidation-driven refetch and the synchronous navigate call.

- timestamp: 2026-04-10 (defect #2)
  checked: trade-flow-ui src/features/subscription/contexts/SubscriptionProvider.tsx and useSubscription.ts
  found: SubscriptionProvider reads useGetCurrentUserQuery and useGetSubscriptionQuery. Because it is mounted in AuthenticatedLayout (above OnboardingGuard), it persists across the /onboarding → /dashboard navigation. It does NOT clear on route change.
  implication: This is fine — SubscriptionProvider isn't the cause; the startTrial invalidation of ["Subscription"] correctly refetches. The only problem is the User query race.

- timestamp: 2026-04-10 (defect #3)
  checked: User re-test produced a 500 from POST /v1/subscription/trial. API logs show MongoDB E11000 on subscriptions collection, index userId_1, dup key userId="KBb7szqrWRdkjtW56E5fYyQuGPP2". The error is logged with the "Internal server error encountered" wrapper (fallback branch in handle-error.utility.ts), proving the error is NOT an InvalidRequestError/ResourceNotFoundError/etc — it's a raw Mongo error that escaped the service.
  found: The 422 path in SubscriptionTrialCreator (findByUserId → throw InvalidRequestError) is not being hit. The code is reaching upsertByStripeCustomerId and hitting the unique-index violation on userId.
  implication: Either (a) a pre-existing subscription row is missing despite the re-test scenario, OR (b) the findByUserId-then-upsert sequence is non-atomic and a concurrent call creates the row between the check and the insert. I need to figure out which.

- timestamp: 2026-04-10 (defect #3)
  checked: trade-flow-api src/subscription/services/subscription-trial-creator.service.ts SubscriptionTrialCreator.create
  found: Flow is: findByUserId(userId) → if existing throw InvalidRequestError → stripe.customers.create (NEW customer each call) → stripe.subscriptions.create → upsertByStripeCustomerId(customer.id, { userId, ... }). The upsert is keyed on stripeCustomerId (NOT userId), so when a new Stripe customer id is generated per call, each call produces an insert against the Mongo collection, which is guarded by a unique index on userId. The whole flow has NO try/catch — any Stripe or Mongo error bubbles up to the controller, which maps non-domain errors to HTTP 500 via the fallback in createHttpError.
  implication: This service is vulnerable to any race or retry: the findByUserId check is NOT atomic with the insert. Two calls arriving at approximately the same time both see "null" from findByUserId, both proceed, both create a new Stripe customer, and the second insert fails with E11000. Unlike SubscriptionCreator.createCheckoutSession (which uses upsertByUserId — atomic against the userId unique index) this code uses upsertByStripeCustomerId.

- timestamp: 2026-04-10 (defect #3)
  checked: trade-flow-api src/subscription/repositories/subscription.repository.ts ensureIndexes + upsertByStripeCustomerId
  found: ensureIndexes creates unique index on {userId:1}. upsertByStripeCustomerId uses findOneAndUpdate({stripeCustomerId}, ..., {upsert:true}) — this is atomic ONLY against the stripeCustomerId filter, not against userId. When no doc has that stripeCustomerId the upsert branches to $setOnInsert which inserts a new doc; if that insert collides with another doc that already has the same userId, the unique index triggers E11000.
  implication: Confirms the mechanism. The error the user saw is exactly what happens when two calls race against a user with no subscription row (or when the second call is separated enough to miss a concurrent insert but still early enough to race the index).

- timestamp: 2026-04-10 (defect #3)
  checked: trade-flow-ui src/features/onboarding/components/SetupLoadingScreen.tsx (useEffect at line 130) and src/main.tsx
  found: SetupLoadingScreen uses `useEffect(() => { void executeSetup(depsRef.current); }, [])` — empty deps — inside a <StrictMode> tree (main.tsx imports StrictMode from react and wraps App). React 19 StrictMode intentionally mounts, unmounts, and re-mounts every component in dev mode, so effects with empty deps fire TWICE on mount. Critically, executeSetup is async and not guarded by any "has already run" ref. Both invocations proceed in parallel and both reach the startTrial() mutation. Because the new Stripe customer is created per call, both calls do hit the backend.
  implication: This is the trigger for defect #3 in DEV — two concurrent POST /v1/subscription/trial calls race the non-atomic check-then-insert. The first call wins and creates the sub row; the second call's findByUserId already ran (returning null before the first call's upsert had landed) so it proceeds to create a second Stripe customer and attempts a conflicting insert, triggering E11000. The same bug would also bite any genuine network retry in production (e.g., a double-click or a React Router re-mount).

- timestamp: 2026-04-10 (defect #3)
  checked: trade-flow-ui src/features/onboarding/components/SetupLoadingScreen.tsx executeSetup try/catch around startTrial (lines 63-74)
  found: Frontend ALREADY expects the 422 path: `if (!isAlreadyExists) { setHasError(true); return; }`. The comment says "Trial may already exist (422 from D-09) -- treat as success". So the contract the frontend expects is "startTrial throws a 422 if a trial already exists" — but the backend today is throwing a 500 (or, on the happy path, a successful 201). A clean fix would be to have the backend return 201 with the existing subscription (idempotent, matching REST semantics for "PUT-like" retries), and the frontend can drop the 422-handling hack as a future cleanup (not required for this bug fix).
  implication: I'll (a) make SubscriptionTrialCreator idempotent (return existing if any), (b) wrap the upsert in a try/catch that maps E11000 → re-fetch the sub by userId → return it (belt-and-braces race protection), and (c) keep the frontend's 422 handling in place for backward compatibility even though new backend will only emit 201 on both paths.

- timestamp: 2026-04-10 (defect #3)
  checked: Prior debug session checkout-duplicate-subscription-key.md (related pattern)
  found: A previous defect in SubscriptionCreator.createCheckoutSession had exactly the same class of bug (insert-only pattern colliding with unique index on userId). The fix there was to switch to upsertByUserId. user-duplicate-creation-race-condition.md further shows UserRepository.findOrCreate uses $or match + $setOnInsert for atomic idempotency.
  implication: Idempotent/atomic writes are the codified pattern in this codebase. The trial creator is the only subscription creation path that still uses the fragile find-then-insert pattern.

- timestamp: 2026-04-10 (defect #4)
  checked: End-user re-test after defects #1/#2/#3 fixes. User reports "It worked, but it looks like once step two is completed two simultaneous POST requests are being sent to the business endpoint so two of the same business are being created with two different IDs". API logs confirm: TWO concurrent POST /v1/business for "testing-user-123@gmail.com" producing businessIds 69d9547a390f5ecfa1c6044e and 69d9547a390f5ecfa1c60450, both completing 201 within ~10ms of each other (responseTime 1046 and 1056). Both seed default tax rates, items, bundles, job types, visit types, quote settings — full duplicate resource set. User's businessIds=["69d9547a390f5ecfa1c6044e","69d9547a390f5ecfa1c60450"]. Two POST /v1/subscription/trial fire in parallel too; fix #3 recovery kicks in on the second one (finds the first-created sub by userId after E11000) so trial ends up with ONE row — but both calls created Stripe customers, so one Stripe customer is leaked.
  implication: The defect #3 fix is working as intended for the trial endpoint, but there is no analogous uniqueness constraint for businesses, so the root trigger (two parallel executeSetup invocations) produces two genuinely independent businesses. Root cause must be upstream of both endpoints — something is firing executeSetup twice.

- timestamp: 2026-04-10 (defect #4)
  checked: trade-flow-ui/src/features/onboarding/components/SetupLoadingScreen.tsx lines 117-132, end-to-end re-read.
  found: The useEffect at line 130 is `useEffect(() => { void executeSetup(depsRef.current); }, []);` with empty deps and no hasRunRef guard. The preceding useEffect at line 117 only updates depsRef.current — it's not an effect-skip guard. executeSetup's internal refetchBusinesses early-return exists but is non-atomic: both parallel executeSetup invocations resolve refetchBusinesses BEFORE either createBusiness lands, so both see the empty businesses list and both fire createBusiness. This is exactly the same non-atomic find-then-insert pattern that defect #3 suffered from, but at the component-effect layer instead of the service layer.
  implication: The component layer must be made idempotent with a ref guard. The defect #3 server-side idempotency is necessary but insufficient because business creation has no unique constraint. A useRef hasRun guard is the standard React 18+/19 pattern for non-idempotent effect bodies under StrictMode.

- timestamp: 2026-04-10 (defect #4)
  checked: trade-flow-ui/src/main.tsx
  found: `createRoot(...).render(<StrictMode><App /></StrictMode>)` — StrictMode is enabled at the root, confirming the development-mode double-invocation environment. Re-confirmed from earlier defect #3 evidence (already known, re-verified per investigation hint).
  implication: Any useEffect with non-idempotent side effects and empty deps is guaranteed to fire twice in dev. The fix must be correct under StrictMode, which means a ref guard (not a state flag — state is reset across the StrictMode unmount/remount cycle; refs persist across it).

- timestamp: 2026-04-10 (defect #4)
  checked: trade-flow-ui/src/features/onboarding/components/__tests__/SetupLoadingScreen.test.tsx
  found: All existing tests render <SetupLoadingScreen /> directly (not wrapped in <StrictMode>). This means StrictMode's double-invocation has never been exercised by the test suite — the regression that shipped was invisible to the existing tests by construction.
  implication: The fix must include a new test that wraps SetupLoadingScreen in React.StrictMode and asserts createBusiness and startTrial are each called exactly once. Existing tests should continue to pass unchanged (the ref guard has no effect on a single mount).

## Resolution

root_cause: |
  Two independent defects in the onboarding flow, both manifesting as "wizard lands on step 2".

  Defect #1 (FIXED and verified by user):
  In trade-flow-api JwtAuthGuard.canActivate (src/auth/auth.guard.ts), newly created users defaulted
  `name` to `userInfo.email`, so email/password signups without a Firebase displayName persisted with
  `name = email` — non-empty and never null. On the frontend, useOnboardingStep treats any non-empty
  trimmed name as "profile step already complete" and starts the wizard at the business step (step 2),
  skipping step 1 entirely.

  Defect #2 (FIXED and verified by user):
  In trade-flow-ui SetupLoadingScreen.executeSetup, after createBusiness and startTrial resolve the
  code immediately calls navigate("/dashboard"). createBusiness.invalidatesTags includes "User", which
  schedules a refetch of /v1/user/me — but this refetch is fire-and-forget. navigate fires
  synchronously before the refetch response lands. On the next render, OnboardingGuard reads
  useCurrentBusiness() which uses RTK Query's isLoading flag (false during background refetches, not
  true), so it computes hasBusiness from STALE cached user data (businessRoles=[]). The guard then
  redirects with <Navigate to="/onboarding" replace />. OnboardingPage remounts OnboardingWizard,
  useOnboardingStep sees hasName=true and (still) hasBusiness=false, and initialStep="business" is
  captured in useState. Because OnboardingWizard's step state is not reactive to useOnboardingStep
  changes, the user stays on step 2 even after the refetch eventually updates the cache.

  Defect #3 (NEW):
  In trade-flow-api SubscriptionTrialCreator.create, the service uses a non-atomic find-then-insert
  sequence: it calls findByUserId() first and throws an InvalidRequestError if a sub exists,
  otherwise it creates a brand-new Stripe customer and calls upsertByStripeCustomerId (keyed on
  stripeCustomerId, not userId). Because the collection has a unique index on subscriptions.userId,
  any second concurrent or retried call that passes the null check will insert a second document
  for the same userId and trigger a MongoDB E11000 duplicate key error. Nothing in the service or
  controller catches E11000, so the raw Mongo error falls through createHttpError's instanceof
  ladder and is mapped to HTTP 500 with the raw errorResponse payload leaked in the response body.

  The trigger in DEV is React StrictMode in trade-flow-ui/src/main.tsx: <StrictMode> double-invokes
  useEffect hooks, so SetupLoadingScreen fires executeSetup twice on mount, producing two parallel
  POST /v1/subscription/trial calls. In PROD the same class of bug would be exposed by any genuine
  retry (double-click, router remount, client-side retry).

  Defect #4 (NEW):
  The StrictMode double-invocation that triggered defect #3 on the trial endpoint ALSO fires two
  parallel POST /v1/business calls on the business endpoint. The component-level executeSetup has
  an early-return when refetchBusinesses returns a non-empty list, but the two StrictMode
  invocations run in parallel — both resolve their refetchBusinesses against an empty collection
  BEFORE either createBusiness call has landed, so both proceed past the early-return and both
  call createBusiness. Unlike subscriptions, businesses have no user-scoped uniqueness constraint
  (a user may legitimately own multiple businesses in the future), so the backend happily creates
  two independent business documents, attaches both businessIds to the user, and seeds the full
  default resource set (tax rates, items, bundles, job types, visit types, quote settings) for
  each. The trial endpoint also gets called twice in parallel; defect #3's E11000-recovery path
  absorbs the second insert so the trial row ends up deduplicated — but both calls still create
  their own Stripe customer before the second one discovers it has lost the race, leaking one
  Stripe customer per duplicate onboarding attempt.

  The correct fix is not server-side idempotency (businesses are legitimately multi-per-user) but
  component-level idempotency: guard executeSetup with a useRef "has run" flag so the effect body
  runs exactly once per component mount regardless of StrictMode. Refs (unlike useState) persist
  across StrictMode's synchronous unmount/remount cycle, making this the canonical React 18+/19
  pattern for non-idempotent effects.

fix: |
  Defect #1 (applied and verified):
  In trade-flow-api/src/auth/auth.guard.ts, change the newUser `name` fallback from
  `userInfo.name || userInfo.displayName || userInfo.email` to
  `userInfo.name || userInfo.displayName || null`.

  Defect #2 (applied and verified):
  In trade-flow-ui/src/features/onboarding/components/SetupLoadingScreen.tsx, executeSetup now
  awaits both refetchUser() AND refetchBusinesses() after startTrial resolves, and only calls
  navigate("/dashboard") once the refetched user has at least one businessRole. If the refetched
  user is still missing a businessRole (a real backend failure, not a cache race), the component
  surfaces the error state and shows Try again instead of silently bouncing back to /onboarding.
  Specifically:
  1. Added useGetCurrentUserQuery + refetchUser to the component hooks and depsRef.
  2. executeSetup now calls `Promise.all([refetchUser(), refetchBusinesses()])` after startTrial.
  3. Guards against the refetched user having businessRoles.length === 0 → setHasError(true).
  4. Unit tests updated: new mock refetchUser, new test asserting refetch-before-navigate order,
     new test asserting error state when the refetched user has no businessRole.

  Defect #3 (applied, pending user verification):
  In trade-flow-api/src/subscription/services/subscription-trial-creator.service.ts, make the
  SubscriptionTrialCreator.create method idempotent and race-safe:
  1. Short-circuit idempotency: if findByUserId returns an existing sub, return it directly
     (previously threw InvalidRequestError). This matches the documented UserRepository.findOrCreate
     pattern in the codebase and aligns with the frontend's existing "treat existing as success"
     expectation.
  2. Belt-and-braces race protection: wrap the final upsertByStripeCustomerId in try/catch. If a
     duplicate-key error occurs (error with typeof === "object" and "code" in error && code === 11000),
     log a warning and re-fetch the subscription by userId, returning that. This closes the race
     window between the initial findByUserId check and the upsert.
  3. Unit tests rewritten to cover both new paths: (a) existing sub returned without Stripe calls,
     (b) E11000 during upsert is recovered by re-fetching by userId. The old test asserting
     InvalidRequestError is replaced with the new idempotent return path test.
  The frontend contract is unchanged — the endpoint now reliably returns 201 with a subscription
  DTO on both first-time and retry/concurrent calls, so SetupLoadingScreen's 422 handling becomes
  dead code but is left in place for backwards compatibility and extra defensiveness.

  Defect #4 (applied, pending user verification):
  In trade-flow-ui/src/features/onboarding/components/SetupLoadingScreen.tsx, add a hasRunRef
  useRef flag and guard the mount useEffect with it so executeSetup runs at most once per
  component mount regardless of React StrictMode's intentional double-invocation:
  1. `const hasRunRef = useRef(false);` declared alongside depsRef.
  2. Mount useEffect body is now `if (hasRunRef.current) return; hasRunRef.current = true;
     void executeSetup(depsRef.current);`.
  3. Retry handler also sets hasRunRef.current = true (redundant but explicit — the Try again
     path was already safe because it calls executeSetup directly and bypasses the guarded
     useEffect).
  4. New regression test: "runs executeSetup exactly once even when wrapped in <StrictMode>".
     Mounts SetupLoadingScreen inside <StrictMode> and asserts that createBusiness and
     startTrial are each called exactly once, and navigate("/dashboard") is still reached.
     This test would have caught the defect if it had been present during the defect #2 fix.
  Ref is used (not state) because refs persist across StrictMode's synchronous unmount/remount
  cycle while state is reset. No backend changes are required. Defect #3's server-side
  E11000 recovery path is still valuable as belt-and-braces protection against any genuine
  network retry in production (double-click, router remount, explicit retry).

  Known follow-up (out of scope for this session):
  Defect #3's E11000 recovery path in SubscriptionTrialCreator.create does not delete the
  losing Stripe customer when a race is detected, so any code path that still produces two
  concurrent trial calls will leak one Stripe customer. Defect #4's ref guard eliminates the
  StrictMode-induced race, so in practice this leak will only occur on a genuine network
  retry. A clean-up (either delete the losing Stripe customer in the recovery branch, or
  create the Mongo row first and attach the Stripe customer after) is recommended as a
  follow-up.

verification:
- Defect #1 verified by end user on 2026-04-10: "After registering the brand-new user, it took me to step 1, which I completed, and then I went to step 2 which I completed."
- trade-flow-api `npm run ci` PASSED for defect #1: 386 tests (57 suites), 0 lint errors, prettier clean, typecheck clean.
- trade-flow-ui `npm run ci` PASSED for defect #2: 71 tests (9 suites) — 2 new tests added, 0 lint errors, prettier clean, typecheck clean, exit 0.
- Defect #2 verified by end user on 2026-04-10 (implicit — the user's re-test reached startTrial, proving step 1 → step 2 → trial creation is working end-to-end; the remaining problem is defect #3 at the trial endpoint itself).
- trade-flow-api `npm run ci` PASSED for defect #3: 389 tests (57 suites, +3 new tests), 0 lint errors, prettier clean, typecheck clean. New tests cover: (a) idempotent return when existing sub found; (b) E11000 recovery by re-fetching by userId; (c) non-duplicate errors still propagate; (d) InternalServerError when E11000 but no existing row found on recovery.
- trade-flow-ui `npm run ci` re-run for defect #3 regression check: 71 tests (9 suites), 0 errors, prettier clean, typecheck clean. No UI changes were required for defect #3 — the frontend's existing "treat 422 as success" path is preserved for safety but is now dead code on the happy path because the backend returns 201 on both first-time and repeat calls.
- Defect #3 verified implicitly by end user on 2026-04-10: the re-test showed trial endpoint hitting the E11000 recovery path on the second StrictMode invocation and ending up with ONE subscription row per user ("fix #3 DOES kick in on the trial: first call creates sub_..., second hits E11000 and recovery finds the existing sub — so trial ends up with ONE row").
- trade-flow-ui `npm run ci` PASSED for defect #4: 72 tests (9 suites, +1 new StrictMode regression test), 0 lint errors (2 pre-existing warnings: one for react-hook-form's watch() in BusinessStep.tsx and one for exhaustive-deps on the dep-sync useEffect in SetupLoadingScreen.tsx — neither introduced by this fix), prettier clean, typecheck clean.
- Defect #4 verified by end user on 2026-04-10: "confirmed fixed". The user re-tested the onboarding flow end-to-end with a brand-new registration; exactly one POST /v1/business and one POST /v1/subscription/trial are fired per step-2 submission, a single businessId is attached to the user, and a single subscription row is created. Final session state: all four defects resolved and user-verified.

files_changed:
  - trade-flow-api/src/auth/auth.guard.ts (defect #1)
  - trade-flow-ui/src/features/onboarding/components/SetupLoadingScreen.tsx (defects #2 and #4)
  - trade-flow-ui/src/features/onboarding/components/__tests__/SetupLoadingScreen.test.tsx (defects #2 and #4 tests)
  - trade-flow-api/src/subscription/services/subscription-trial-creator.service.ts (defect #3)
  - trade-flow-api/src/subscription/test/services/subscription-trial-creator.service.spec.ts (defect #3 tests)
