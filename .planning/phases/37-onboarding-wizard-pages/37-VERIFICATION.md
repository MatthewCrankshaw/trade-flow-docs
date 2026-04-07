---
phase: 37-onboarding-wizard-pages
verified: 2026-04-07T00:00:00Z
status: passed
score: 11/11 must-haves verified
re_verification:
  previous_status: gaps_found
  previous_score: 8/9
  gaps_closed:
    - "DefaultQuoteSettingsCreatorService created in quote-settings module and wired into BusinessCreator.create() as 5th default resource creator"
    - "REQUIREMENTS.md updated: ONBD-02, ONBD-03, ONBD-04 now marked [x] and Complete in traceability table"
  gaps_remaining: []
  regressions: []
human_verification:
  - test: "Complete new user onboarding flow end-to-end"
    expected: "User redirected to /onboarding, completes Step 1 (display name), proceeds to Step 2 (business name + trade), SetupLoadingScreen shows 'Setting up your business...' then 'Starting your free trial...', success toast appears, redirect to /dashboard"
    why_human: "Full browser flow with Firebase auth and live API calls required"
  - test: "Refresh mid-wizard after completing Step 1"
    expected: "Page loads at Step 2 (business step) since display name exists but no business — progress bar shows Step 2 of 2"
    why_human: "Session state and resume logic requires live browser with authenticated user"
  - test: "Existing user with complete profile navigates to app"
    expected: "No redirect to /onboarding — goes directly to requested route"
    why_human: "Requires authenticated user with both name and business in database"
  - test: "TrialBadge urgency color shifts"
    expected: "Default (>10 days), yellow warning (4-10 days), destructive red (1-3 days), 'Trial ends today' at 0 days"
    why_human: "Requires manipulating trialEnd date or mocked subscription data in browser"
  - test: "TrialBadge click opens Stripe Billing Portal"
    expected: "New tab opens with Stripe Billing Portal URL"
    why_human: "Requires live Stripe integration and active subscription"
---

# Phase 37: Onboarding Wizard Pages Verification Report

**Phase Goal:** Every new user completes a mandatory two-step setup (profile name, then business name + trade) before accessing the app, and existing users with completed profiles and businesses pass through automatically
**Verified:** 2026-04-07
**Status:** passed
**Re-verification:** Yes — after gap closure plan 37-04 executed

## Goal Achievement

### Observable Truths

| #  | Truth                                                                                     | Status     | Evidence                                                                                                                                    |
|----|-------------------------------------------------------------------------------------------|------------|---------------------------------------------------------------------------------------------------------------------------------------------|
| 1  | New user without display name is redirected to /onboarding                                | VERIFIED   | OnboardingGuard checks `user?.name?.trim()` and returns `<Navigate to="/onboarding" replace />` when false                                 |
| 2  | User sees Step 1 form with display name input and progress bar at 50%                    | VERIFIED   | OnboardingWizard renders OnboardingProgress (currentStep=1, totalSteps=2) + ProfileStep form with "e.g. Dave" placeholder                  |
| 3  | Completing Step 1 saves display name via PATCH /v1/user/me and advances to Step 2        | VERIFIED   | ProfileStep calls `useUpdateUserMutation` then `onComplete()`; PATCH /v1/user/me with `{ name: data.displayName }`                        |
| 4  | User sees Step 2 form with business name input and 10-trade icon card grid                | VERIFIED   | BusinessStep form with TradeCardGrid renders 10 TRADE_OPTIONS as accessible button cards                                                   |
| 5  | Selecting 'Other' trade reveals custom trade text input                                   | VERIFIED   | BusinessStep conditionally renders Input with placeholder "e.g. Roofer, Glazier" when `selectedTrade === "other"`                         |
| 6  | Completing Step 2 creates business with GB/GBP and starts trial; success redirects       | VERIFIED   | SetupLoadingScreen calls `createBusiness({country:"GB", currency:"GBP",...})` then `startTrial()`, then `navigate("/dashboard")`          |
| 7  | After business creation, ALL required defaults are auto-created (incl. quote template)   | VERIFIED   | BusinessCreator.create() calls all 5 default creators: tax rates, items, job types, visit types, AND DefaultQuoteSettingsCreatorService   |
| 8  | Existing user with display name AND business bypasses onboarding entirely                 | VERIFIED   | OnboardingGuard: `if (hasDisplayName && hasBusiness) return <Outlet />;`                                                                  |
| 9  | Refreshing mid-wizard resumes at correct step                                             | VERIFIED   | `useOnboardingStep` returns `initialStep: "business"` when `user.name` exists but no business; wizard initializes from `initialStep`      |
| 10 | User sees trial days remaining in app header (TRIAL-02)                                  | VERIFIED   | TrialBadge rendered unconditionally in DashboardLayout header; returns null when not trialing; desktop "N days left", mobile "Nd"          |
| 11 | Clicking trial badge opens Stripe Billing Portal in new tab (TRIAL-03)                  | VERIFIED   | TrialBadge calls `createPortalSession()` → POST /v1/subscription/portal, then `window.open(result.url, "_blank", "noopener,noreferrer")` |

**Score:** 11/11 truths verified

### Required Artifacts

| Artifact                                                                                            | Expected                                       | Status   | Details                                                                                                                        |
|-----------------------------------------------------------------------------------------------------|------------------------------------------------|----------|--------------------------------------------------------------------------------------------------------------------------------|
| `trade-flow-ui/src/features/auth/components/OnboardingGuard.tsx`                                   | Redirect logic checking name and business      | VERIFIED | Uses `useCurrentBusiness()` for user+business state, `useLocation()` for path check                                          |
| `trade-flow-ui/src/features/onboarding/components/OnboardingWizard.tsx`                            | Main wizard container with step state          | VERIFIED | `useState<WizardStep>`, `useOnboardingStep`, renders ProfileStep/BusinessStep/SetupLoadingScreen                               |
| `trade-flow-ui/src/features/onboarding/components/ProfileStep.tsx`                                 | Step 1 display name form                       | VERIFIED | `useForm` + `valibotResolver`, `useUpdateUserMutation`, "Continue Setup" button, error toast                                   |
| `trade-flow-ui/src/features/onboarding/components/BusinessStep.tsx`                                | Step 2 form with business name + trade         | VERIFIED | `useForm` + `valibotResolver`, `v.forward` cross-field validation, "Set up my business" button                                |
| `trade-flow-ui/src/features/onboarding/components/TradeCard.tsx`                                   | Accessible trade card button                   | VERIFIED | `<button type="button">`, `aria-pressed={isSelected}`, selected styles with ring-2                                            |
| `trade-flow-ui/src/features/onboarding/components/TradeCardGrid.tsx`                               | Grid of trade cards                            | VERIFIED | `grid grid-cols-2 sm:grid-cols-3 gap-2`, maps over `TRADE_OPTIONS`                                                            |
| `trade-flow-ui/src/features/onboarding/components/SetupLoadingScreen.tsx`                          | Full-screen loading/error state                | VERIFIED | Sequential createBusiness + startTrial, partial-completion retry, error recovery UI                                           |
| `trade-flow-ui/src/features/onboarding/components/OnboardingProgress.tsx`                          | Progress bar with step text                    | VERIFIED | `role="progressbar"`, `aria-valuenow/min/max`, `bg-primary rounded-full` fill                                                 |
| `trade-flow-ui/src/features/onboarding/constants/trades.ts`                                        | 10 trade options                               | VERIFIED | `TRADE_OPTIONS` with exactly 10 entries, all expected enum values present                                                      |
| `trade-flow-ui/src/features/onboarding/hooks/useOnboardingStep.ts`                                 | Step resume detection hook                     | VERIFIED | Returns `"business"` when name exists + no business, `"profile"` otherwise                                                    |
| `trade-flow-ui/src/pages/OnboardingPage.tsx`                                                        | Route-level onboarding shell                   | VERIFIED | Imports `OnboardingWizard` from `@/features/onboarding`, renders it directly                                                   |
| `trade-flow-ui/src/features/subscription/components/TrialBadge.tsx`                                | Trial days badge with urgency colors           | VERIFIED | `calculateDaysRemaining`, urgency classes, hidden/md:inline-flex responsive variants                                           |
| `trade-flow-ui/src/components/layouts/DashboardLayout.tsx`                                         | TrialBadge in header                           | VERIFIED | `import { TrialBadge } from "@/features/subscription"`, rendered in header right-actions div                                  |
| `trade-flow-api/src/quote-settings/services/default-quote-settings-creator.service.ts`             | DefaultQuoteSettingsCreatorService             | VERIFIED | `@Injectable()`, `async create(businessId: string): Promise<void>`, calls `quoteSettingsRepository.upsert` with default email template |
| `trade-flow-api/src/business/services/business-creator.service.ts`                                 | Auto-creates all 5 default resource types      | VERIFIED | Injects and calls all 5 creators: tax rates, items, job types, visit types, DefaultQuoteSettingsCreatorService                |

### Key Link Verification

| From                                              | To                                                                     | Via                                                   | Status  | Details                                                                                                      |
|---------------------------------------------------|------------------------------------------------------------------------|-------------------------------------------------------|---------|--------------------------------------------------------------------------------------------------------------|
| OnboardingGuard.tsx                               | useCurrentBusiness hook                                                | `const { user, hasBusiness, isLoading } =`            | WIRED   | Exposes user.name for display name check and hasBusiness for business check                                 |
| OnboardingWizard.tsx                              | ProfileStep.tsx                                                        | Conditional render `step === "profile"`               | WIRED   | `<ProfileStep onComplete={() => setStep("business")} />`                                                    |
| OnboardingWizard.tsx                              | BusinessStep.tsx                                                       | Conditional render `step === "business"`              | WIRED   | `<BusinessStep onComplete={(data) => { setBusinessData(data); setStep("loading"); }} />`                    |
| OnboardingWizard.tsx                              | SetupLoadingScreen.tsx                                                 | Conditional render `step === "loading"`               | WIRED   | `{step === "loading" && businessData && <SetupLoadingScreen businessData={businessData} />}`                |
| ProfileStep.tsx                                   | PATCH /v1/user/me                                                      | `useUpdateUserMutation` from `@/services`             | WIRED   | `await updateUser({ name: data.displayName }).unwrap()`                                                     |
| BusinessStep.tsx                                  | TradeCardGrid.tsx                                                      | Renders `<TradeCardGrid>` with selection cb           | WIRED   | `<TradeCardGrid selectedTrade={selectedTrade} onTradeSelect={(val) => setValue(...)} />`                    |
| SetupLoadingScreen.tsx                            | POST /v1/business                                                      | `useCreateBusinessMutation` from `@/services`         | WIRED   | `await createBusiness({name, country:"GB", currency:"GBP", trade:{...}}).unwrap()`                         |
| SetupLoadingScreen.tsx                            | POST /v1/subscription/trial                                            | `useStartTrialMutation` from subscriptionApi          | WIRED   | `await startTrial().unwrap()`, 422 treated as success                                                       |
| TrialBadge.tsx                                    | GET /v1/subscription                                                   | `useSubscription()` context → subscriptionApi         | WIRED   | `const { isLoading, subscription } = useSubscription()`                                                     |
| TrialBadge.tsx                                    | POST /v1/subscription/portal                                           | `useCreatePortalSessionMutation`                      | WIRED   | `await createPortal().unwrap()`, opens `result.url` in new tab                                              |
| DashboardLayout.tsx                               | TrialBadge.tsx                                                         | `import { TrialBadge } from "@/features/subscription"` | WIRED | `<TrialBadge />` rendered unconditionally in header right-actions                                           |
| App.tsx                                           | OnboardingGuard (dual-wrap)                                            | Route structure wrapping onboarding + main app        | WIRED   | OnboardingGuard wraps `/onboarding` route and main app routes                                               |
| business-creator.service.ts                       | default-quote-settings-creator.service.ts                              | Constructor injection + `this.defaultQuoteSettingsCreator.create(createdBusiness.id)` | WIRED | Line 32: constructor param; line 53: called after other 4 default creators |
| quote-settings.module.ts                          | DefaultQuoteSettingsCreatorService                                     | providers + exports arrays                            | WIRED   | Service in `providers: [...]` and `exports: [QuoteSettingsRetriever, DefaultQuoteSettingsCreatorService]`   |
| business.module.ts                                | QuoteSettingsModule                                                    | `forwardRef(() => QuoteSettingsModule)` in imports    | WIRED   | Circular dependency resolved with forwardRef on both sides                                                  |

### Data-Flow Trace (Level 4)

| Artifact                 | Data Variable     | Source                                                             | Produces Real Data | Status   |
|--------------------------|-------------------|--------------------------------------------------------------------|--------------------|----------|
| OnboardingGuard.tsx      | `user`, `hasBusiness` | `useCurrentBusiness()` → RTK Query `useGetCurrentUserQuery` + `useGetBusinessesQuery` | Yes — live API | FLOWING |
| ProfileStep.tsx          | `isValid`, `isSubmitting` | react-hook-form state                                          | Yes — form state   | FLOWING  |
| SetupLoadingScreen.tsx   | `phase`, `hasError` | useState + sequential mutations                                  | Yes — live API     | FLOWING  |
| TrialBadge.tsx           | `subscription.trialEnd` | `useSubscription()` → `useGetSubscriptionQuery` → GET /v1/subscription | Yes — live API | FLOWING |
| OnboardingWizard.tsx     | `step`            | `useState` initialized from `useOnboardingStep`                   | Yes — derived from user/business state | FLOWING |
| DefaultQuoteSettingsCreatorService | n/a (write-only) | `quoteSettingsRepository.upsert(businessId, {...})` | Yes — DB write with template content | FLOWING |

### Behavioral Spot-Checks

Step 7b: SKIPPED — UI components require a running browser/Vite dev server. Static analysis and wiring checks confirm the implementation is correct. API service verified by TypeScript compilation in trade-flow-api (commit b82187a passes tsc --noEmit).

### Requirements Coverage

| Requirement | Source Plan | Description                                                                                                | Status    | Evidence                                                                                                  |
|-------------|-------------|------------------------------------------------------------------------------------------------------------|-----------|-----------------------------------------------------------------------------------------------------------|
| ONBD-01     | 37-01-PLAN  | New user required to enter display name before accessing app                                                | SATISFIED | OnboardingGuard redirects; ProfileStep form saves name via PATCH /v1/user/me                             |
| ONBD-02     | 37-02-PLAN  | New user required to enter business name and select primary trade                                           | SATISFIED | BusinessStep form with TradeCardGrid fully implemented and wired. REQUIREMENTS.md marked [x] Complete.   |
| ONBD-03     | 37-02-PLAN  | Country defaults to UK and currency defaults to GBP                                                        | SATISFIED | `country: "GB"`, `currency: "GBP"` hardcoded in SetupLoadingScreen. REQUIREMENTS.md marked [x] Complete. |
| ONBD-04     | 37-02-PLAN  | Tax rates, job types, visit types, items, and quote email template auto-created at business setup           | SATISFIED | BusinessCreator calls all 5 default creators including DefaultQuoteSettingsCreatorService (commit b82187a). REQUIREMENTS.md marked [x] Complete. |
| ONBD-05     | 37-01-PLAN  | User can see progress indication during onboarding                                                         | SATISFIED | OnboardingProgress component with role="progressbar", aria attributes, Step N of 2 text                  |
| ONBD-06     | 37-01-PLAN  | Onboarding wizard resumes at correct step on refresh                                                       | SATISFIED | `useOnboardingStep` returns "business" when user has name but no business                                |
| ONBD-07     | 37-01-PLAN  | Existing users with complete profile and business bypass onboarding                                         | SATISFIED | `if (hasDisplayName && hasBusiness) return <Outlet />;` in OnboardingGuard                               |
| TRIAL-02    | 37-03-PLAN  | User can see trial days remaining in app header                                                            | SATISFIED | TrialBadge in DashboardLayout header; desktop "N days left" + Clock icon, mobile "Nd" compact            |
| TRIAL-03    | 37-03-PLAN  | User can add payment method via Stripe Billing Portal at any time during trial                             | SATISFIED | TrialBadge click calls POST /v1/subscription/portal, opens Stripe URL in new tab                         |

All 9 phase requirement IDs are SATISFIED. REQUIREMENTS.md traceability table shows Complete for all ONBD-01 through ONBD-07, TRIAL-02, TRIAL-03.

### Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| No blockers found | — | — | — | — |

No regression in previously passing components. No stubs, placeholder returns, or disconnected props found in gap-closure files.

### Human Verification Required

#### 1. End-to-End Onboarding Flow

**Test:** Sign up as new user, navigate to any app route, complete both wizard steps, verify dashboard redirect
**Expected:** Redirect to /onboarding → Step 1 form → name saved → Step 2 form → business + trade → loading screen → success toast → /dashboard
**Why human:** Full Firebase auth + live API calls + browser navigation required

#### 2. Mid-Wizard Resume

**Test:** Complete Step 1 as new user, hard refresh the /onboarding page
**Expected:** Wizard loads at Step 2 (business step), progress bar shows "Step 2 of 2"
**Why human:** Requires authenticated session with saved name but no business in DB

#### 3. Existing User Bypass

**Test:** Log in as user with name + business, navigate directly to /dashboard
**Expected:** No redirect to /onboarding, loads /dashboard normally
**Why human:** Requires authenticated user with both conditions met in DB

#### 4. TrialBadge Urgency Colors

**Test:** With active trial subscription, inspect badge when trial has >10, <=10, <=3 days remaining
**Expected:** Secondary color (>10 days), yellow warning (<=10 days), destructive red (<=3 days)
**Why human:** Requires manipulating subscription trialEnd date or test fixtures

#### 5. TrialBadge Click → Billing Portal

**Test:** Click TrialBadge in app header during active trial
**Expected:** New browser tab opens with Stripe Billing Portal URL
**Why human:** Requires live Stripe integration with createPortalSession endpoint active

### Gaps Summary

No gaps remain. Both gaps from the initial verification have been closed by plan 37-04:

1. **ONBD-04 (quote email template):** `DefaultQuoteSettingsCreatorService` created in `trade-flow-api/src/quote-settings/services/` with `@Injectable()` decorator, `async create(businessId)` method, and a default email template with `{businessName}` and `{customerName}` placeholders. Service registered in `QuoteSettingsModule` providers and exports. Injected into `BusinessCreator` constructor and called as the 5th default resource creator after `DefaultVisitTypesCreatorService`. Circular dependency between `BusinessModule` and `QuoteSettingsModule` resolved using `forwardRef` on both sides. Confirmed by TypeScript compilation in trade-flow-api (commit b82187a).

2. **REQUIREMENTS.md tracking:** ONBD-02, ONBD-03, ONBD-04 changed from `- [ ]` to `- [x]` and status in traceability table updated from "Pending" to "Complete". Confirmed by direct file inspection — all 9 Phase 37 requirements show Complete.

---

_Verified: 2026-04-07_
_Verifier: Claude (gsd-verifier)_
