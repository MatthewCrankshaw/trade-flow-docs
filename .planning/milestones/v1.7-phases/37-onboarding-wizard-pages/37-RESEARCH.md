# Phase 37: Onboarding Wizard Pages - Research

**Researched:** 2026-04-02
**Domain:** React multi-step wizard, route guards, trial badge UI
**Confidence:** HIGH

## Summary

Phase 37 implements a mandatory two-step onboarding wizard (profile name, then business name + trade selection) that gates app access for new users. The phase is frontend-only (`trade-flow-ui`), building on the route guard architecture established in Phase 36 (OnboardingGuard pass-through shell) and consuming the trial API endpoint from Phase 35 (POST /v1/subscription/trial). All API endpoints already exist -- this phase wires up the frontend wizard flow, guard redirect logic, and trial status badge.

The implementation requires: (1) populating the OnboardingGuard with redirect logic based on user profile and business existence checks, (2) building the wizard page with two steps using React Hook Form + Valibot validation, (3) a setup loading screen that orchestrates two sequential API calls (create business, start trial), and (4) a TrialBadge component in the DashboardLayout header with mobile compact format and urgency color shifts. The existing old onboarding system (onboardingSlice, OnboardingDialogs, onboardingMiddleware) coexists and is removed in Phase 39.

**Primary recommendation:** Use simple React state (`useState`) to manage wizard step progression (not React Router nested routes), with React Hook Form for each step's form and RTK Query mutations for API calls. The OnboardingGuard reads from existing `useAuth` and `useCurrentBusiness` hooks to determine redirect.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- D-01: Centered card layout on a clean background -- single card, one step at a time
- D-02: Subtle gradient background matching existing loading page style
- D-03: Trade Flow logo centered above the wizard card
- D-04: Progress bar indicator -- horizontal bar filling left-to-right with "Step 1 of 2" / "Step 2 of 2" text
- D-05: No back button on Step 2 -- profile is saved after Step 1
- D-06: No animation between steps -- instant content swap
- D-07: Full-width on mobile (with padding), max-width ~480px on desktop
- D-08: Friendly and direct copy tone -- "What's your name?" / "Tell us about your business"
- D-09: Short subtitle under each step heading
- D-10: Icon card grid with 9 predefined trades + "Other" card -- NOT a dropdown
- D-11: Trade enum matches API: PLUMBER, ELECTRICIAN, HEATING_GAS_ENGINEER, CARPENTER_JOINER, BUILDER_GENERAL_CONTRACTOR, PAINTER_DECORATOR, TILER_FLOORING, LANDSCAPING_GARDENING, HANDYMAN, OTHER
- D-12: Plain Lucide icons above trade names -- no colored circles or badges
- D-13: Selecting "Other" highlights card and reveals text input BELOW the grid
- D-14: Trade field already exists on the business entity -- no schema changes needed
- D-15: Full-screen loading state replaces wizard card -- centered spinner + "Setting up your business..."
- D-16: Two sequential API calls: 1) POST /v1/business, 2) POST /v1/subscription/trial
- D-17: Step 1 saves display name immediately via PATCH /v1/user when user clicks Continue
- D-18: Error handling: replace loading state with error message + "Try again" retry button; retry handles partial completion
- D-19: Success: sonner toast "You're all set! Your 30-day free trial has started." then redirect to dashboard
- D-20: Desktop trial badge already exists -- this phase adds mobile compact variant and urgency color shifts
- D-21: Mobile compact format: "28d" (abbreviated number + "d")
- D-22: Urgency color shifts: default/muted normally, yellow/warning at <=10 days, red/destructive at <=3 days
- D-23: Clicking trial badge opens Stripe Billing Portal (calls manage endpoint, opens in new tab)
- D-24: Badge only visible when subscription status is "trialing"
- D-25: Data source: existing GET /v1/subscription returns trialEnd, frontend calculates days remaining
- D-26: OnboardingGuard gets redirect logic: check displayName AND business, redirect to /onboarding if missing
- D-27: Wizard resumes at correct step on refresh: if displayName exists but no business, start Step 2
- D-28: Existing users with completed profile and business bypass onboarding entirely

### Claude's Discretion
- Exact wizard card dimensions and padding
- Specific Lucide icon choices for each trade card
- Progress bar visual styling (color, height, border-radius)
- Loading spinner style and animation
- Exact copy for step subtitles and error messages
- Trade card grid responsive layout (columns per breakpoint)
- Urgency color exact values (use existing warning/destructive semantic tokens)
- Whether to use React Router nested routes or simple state for wizard steps

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| ONBD-01 | New user required to enter display name before accessing app | OnboardingGuard redirect logic (D-26) + ProfileStep form (D-17) |
| ONBD-02 | New user required to enter business name and select trade before accessing app | BusinessStep form with trade card grid (D-10, D-11) |
| ONBD-03 | Country defaults to UK and currency defaults to GBP (no user input) | POST /v1/business payload hardcodes country: "GB", currency: "GBP" |
| ONBD-04 | Tax rates, job types, visit types, items, quote email template auto-created on business setup | POST /v1/business endpoint already orchestrates all defaults (D-16, existing API) |
| ONBD-05 | User can see progress indication during onboarding steps | OnboardingProgress component with progress bar (D-04) |
| ONBD-06 | Wizard resumes at correct step on refresh/return | OnboardingGuard + wizard step detection based on existing user data (D-27) |
| ONBD-07 | Existing users with completed profile and business bypass onboarding | OnboardingGuard pass-through when displayName AND business exist (D-28) |
| TRIAL-02 | User can see trial days remaining in app header | TrialBadge component with desktop/mobile variants (D-20-D-25) |
| TRIAL-03 | User can add payment method via Stripe Billing Portal at any time | Trial badge click handler calls manage endpoint, opens portal in new tab (D-23) |
</phase_requirements>

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| React | 19.2.0 | UI framework | Already installed, project standard |
| react-hook-form | 7.71.1 | Form state management for wizard steps | Already installed, project standard for all forms |
| @hookform/resolvers | 5.2.2 | Valibot schema integration | Already installed |
| valibot | 1.2.0 | Schema validation for form fields | Already installed, project standard (not Zod) |
| react-router-dom | 7.13.0 | Navigation (Navigate, useNavigate) | Already installed |
| @reduxjs/toolkit | 2.11.2 | RTK Query for API calls | Already installed |
| lucide-react | 0.563.0 | Icons for trade cards and UI elements | Already installed |
| sonner | 2.0.7 | Toast notifications (success/error) | Already installed |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| class-variance-authority | 0.7.1 | Variant styling for trial badge urgency states | Already installed, use for badge variant logic |
| tailwind-merge | 3.4.0 | Merging conditional Tailwind classes | Already installed, used via cn() utility |
| clsx | 2.1.1 | Conditional class names | Already installed, used via cn() utility |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| useState for wizard steps | React Router nested routes (/onboarding/step-1, /onboarding/step-2) | Router routes add URL-based navigation but increase complexity; useState is simpler for a 2-step wizard with instant swap and no back button. Step detection on load handles resume. |

**Installation:** No new packages needed. All dependencies already installed.

## Architecture Patterns

### Recommended Project Structure
```
src/
├── features/
│   ├── onboarding/
│   │   ├── components/
│   │   │   ├── OnboardingWizard.tsx      # Main wizard container (manages step state)
│   │   │   ├── ProfileStep.tsx           # Step 1: display name form
│   │   │   ├── BusinessStep.tsx          # Step 2: business name + trade selection
│   │   │   ├── TradeCardGrid.tsx         # Grid of selectable trade cards
│   │   │   ├── TradeCard.tsx             # Individual selectable trade card
│   │   │   ├── OnboardingProgress.tsx    # Progress bar with step text
│   │   │   ├── SetupLoadingScreen.tsx    # Full-screen loading/error after Step 2
│   │   │   └── index.ts                 # Barrel exports
│   │   ├── hooks/
│   │   │   ├── useOnboardingStep.ts      # Determine current step from user/business state
│   │   │   └── index.ts
│   │   ├── constants/
│   │   │   └── trades.ts                 # Trade enum values, labels, and icon mappings
│   │   └── index.ts                      # Feature barrel
│   │
│   ├── auth/
│   │   └── components/
│   │       └── OnboardingGuard.tsx        # MODIFY: add redirect logic
│   │
│   └── subscription/
│       └── components/
│           ├── TrialBadge.tsx             # NEW: trial days badge
│           └── index.ts
│
├── pages/
│   └── OnboardingPage.tsx                # Route-level page shell
│
└── components/
    └── layouts/
        └── DashboardLayout.tsx            # MODIFY: add TrialBadge to header
```

### Pattern 1: Wizard Step Management with useState
**What:** Simple state-driven step progression for a 2-step wizard
**When to use:** When steps are sequential, no back navigation, and step count is small
**Example:**
```typescript
// OnboardingWizard.tsx
type WizardStep = "profile" | "business" | "loading";

export function OnboardingWizard() {
  const { user } = useAuth();
  const { data: business } = useGetBusinessQuery();

  // Resume at correct step (ONBD-06)
  const initialStep = user?.displayName ? "business" : "profile";
  const [step, setStep] = useState<WizardStep>(initialStep);

  if (step === "loading") {
    return <SetupLoadingScreen />;
  }

  return (
    <div className="flex flex-col items-center justify-center min-h-screen px-4 py-12 bg-gradient-to-br from-background to-muted/30">
      {/* Logo */}
      <OnboardingProgress currentStep={step === "profile" ? 1 : 2} totalSteps={2} />
      <Card className="w-full max-w-[480px]">
        {step === "profile" && <ProfileStep onComplete={() => setStep("business")} />}
        {step === "business" && <BusinessStep onComplete={() => setStep("loading")} />}
      </Card>
    </div>
  );
}
```

### Pattern 2: OnboardingGuard Redirect Logic
**What:** Route guard that checks user state and redirects to onboarding if incomplete
**When to use:** Wrapping authenticated routes that require completed onboarding
**Example:**
```typescript
// OnboardingGuard.tsx
export function OnboardingGuard() {
  const { user } = useAuth();
  const { data: business, isLoading: isBusinessLoading } = useGetBusinessQuery();

  if (isBusinessLoading) {
    return <LoadingSpinner />;
  }

  const hasDisplayName = Boolean(user?.displayName?.trim());
  const hasBusiness = Boolean(business);

  if (!hasDisplayName || !hasBusiness) {
    return <Navigate to="/onboarding" replace />;
  }

  return <Outlet />;
}
```

### Pattern 3: Sequential API Calls with Error Recovery
**What:** Two sequential mutations with partial completion handling
**When to use:** Setup flow where second call depends on first succeeding
**Example:**
```typescript
// SetupLoadingScreen.tsx -- handles D-16 and D-18
const [createBusiness] = useCreateBusinessMutation();
const [startTrial] = useStartTrialMutation();

async function handleSetup(businessData: BusinessPayload) {
  setPhase("business"); // "Setting up your business..."
  try {
    await createBusiness(businessData).unwrap();
  } catch {
    setError(true);
    return;
  }

  setPhase("trial"); // "Starting your free trial..."
  try {
    await startTrial().unwrap();
  } catch {
    setError(true);
    return;
  }

  toast.success("You're all set! Your 30-day free trial has started.");
  navigate("/dashboard");
}
```

### Pattern 4: Trial Badge with Urgency Colors
**What:** Conditional badge styling based on days remaining
**When to use:** Trial status display in header
**Example:**
```typescript
function getTrialBadgeVariant(daysRemaining: number) {
  if (daysRemaining <= 3) return "destructive";
  if (daysRemaining <= 10) return "warning";
  return "secondary";
}

function calculateDaysRemaining(trialEnd: string): number {
  const end = new Date(trialEnd);
  const now = new Date();
  return Math.max(0, Math.ceil((end.getTime() - now.getTime()) / (1000 * 60 * 60 * 24)));
}
```

### Anti-Patterns to Avoid
- **Do NOT use React Router nested routes for wizard steps:** The wizard has only 2 steps with no back button and instant swap. useState is simpler and avoids URL management complexity.
- **Do NOT import from old onboarding system:** The existing `onboardingSlice.ts`, `OnboardingDialogs.tsx`, `onboardingMiddleware.ts`, `OnboardingProvider.tsx`, and `OnboardingContext.ts` are the OLD system. Phase 37 builds a NEW system. Phase 39 removes the old one. The two coexist.
- **Do NOT call both API endpoints in parallel:** Business must be created before trial can start. The trial endpoint requires a business to exist (user must have a business for the subscription to be meaningful).
- **Do NOT build a custom form library:** Use React Hook Form + Valibot resolver. Both are already installed and used throughout the project.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Form state & validation | Custom form state with useState | React Hook Form + Valibot | Already used everywhere in the project; handles dirty tracking, validation, submission state |
| Toast notifications | Custom notification system | Sonner (already installed) | Consistent with rest of app |
| API state management | Custom fetch + loading states | RTK Query mutations | Already used for all API calls; provides loading/error states, cache invalidation |
| Conditional CSS classes | Manual string concatenation | cn() utility from @/lib/utils | Already established pattern |
| Route guarding | Manual auth checks in components | OnboardingGuard + Navigate | Follows existing ProtectedRoute pattern |
| Date calculation | Manual date math | Simple Date arithmetic (no library needed) | Calculating days remaining from trialEnd is trivial -- just subtraction and ceiling |

**Key insight:** Every tool needed for this phase is already installed and in use. This is a pure feature implementation phase with no new dependencies.

## Common Pitfalls

### Pitfall 1: Old Onboarding System Interference
**What goes wrong:** New onboarding wizard conflicts with old OnboardingDialogs/onboardingMiddleware system.
**Why it happens:** Old system uses Redux middleware to auto-open dialogs when business or profile is missing. New system uses route guard redirect.
**How to avoid:** The two systems operate on different triggers. OnboardingGuard redirects to /onboarding BEFORE the user reaches DashboardLayout where the old system lives. The old dialogs only fire inside the dashboard. As long as the guard redirects first, there is no conflict.
**Warning signs:** If a user somehow reaches the dashboard without completing onboarding, the old dialog system may fire. This is actually a safety net, not a bug.

### Pitfall 2: RTK Query Cache Stale After Profile/Business Creation
**What goes wrong:** After PATCH /v1/user (Step 1) or POST /v1/business (Step 2), the cached user/business data in RTK Query is stale. OnboardingGuard still sees old state.
**Why it happens:** RTK Query caches responses. Mutations don't automatically invalidate related queries unless configured with tag invalidation.
**How to avoid:** Ensure the PATCH /v1/user mutation invalidates the user query tag. Ensure the POST /v1/business mutation invalidates the business query tag. Use `invalidatesTags` on the mutation definitions. After business creation + trial start, the navigate to /dashboard will trigger fresh queries.
**Warning signs:** User completes wizard but gets redirected back to /onboarding because guard sees stale data.

### Pitfall 3: Retry Logic for Partial Completion (D-18)
**What goes wrong:** User's business was created successfully but trial start failed. Clicking "Try again" attempts to create business again, which fails with a duplicate error.
**Why it happens:** The retry function naively re-runs both API calls.
**How to avoid:** On retry, check if business already exists (re-fetch business query). If business exists, skip straight to trial creation. The retry function should be resilient to partial completion state.
**Warning signs:** "Try again" button produces a 422 error about business already existing.

### Pitfall 4: Guard Loading State Race Condition
**What goes wrong:** OnboardingGuard renders before user/business data is loaded, causing a flash redirect to /onboarding for existing users.
**Why it happens:** RTK Query queries are initially in loading state. If the guard redirects when data is undefined (loading), it incorrectly redirects completed users.
**How to avoid:** Show a loading spinner while data is being fetched. Only make the redirect decision when both user and business queries have resolved (not loading).
**Warning signs:** Existing users briefly see the onboarding page before being redirected to dashboard.

### Pitfall 5: Trial Days Calculation Off-by-One
**What goes wrong:** Badge shows "0 days left" when there's still time remaining, or "1 day left" when trial just expired.
**Why it happens:** Timezone differences between server (UTC) and client local time, plus integer rounding issues.
**How to avoid:** Use `Math.ceil` for days remaining calculation. Compare timestamps in UTC. When result is 0 or negative, display "Trial ends today" (0d on mobile).
**Warning signs:** Badge shows incorrect day count near midnight.

### Pitfall 6: Business POST Payload Missing Required Fields
**What goes wrong:** POST /v1/business returns 400/422 because payload is missing country or currency fields.
**Why it happens:** ONBD-03 says country defaults to UK and currency to GBP without user input, but developer forgets to hardcode these in the payload.
**How to avoid:** The business creation payload MUST include `country: "GB"` and `currency: "GBP"` hardcoded. These are not user inputs. The form only collects business name and trade.
**Warning signs:** API returns validation error on business creation.

## Code Examples

### Trade Constants File
```typescript
// src/features/onboarding/constants/trades.ts
import type { LucideIcon } from "lucide-react";
import {
  Wrench, Zap, Flame, Hammer, HardHat,
  PaintBucket, Grid3x3, Trees, Drill, MoreHorizontal,
} from "lucide-react";

export interface TradeOption {
  value: string;
  label: string;
  icon: LucideIcon;
}

export const TRADE_OPTIONS: TradeOption[] = [
  { value: "plumber", label: "Plumber", icon: Wrench },
  { value: "electrician", label: "Electrician", icon: Zap },
  { value: "heating_gas_engineer", label: "Heating & Gas", icon: Flame },
  { value: "carpenter_joiner", label: "Carpenter", icon: Hammer },
  { value: "builder_general_contractor", label: "Builder", icon: HardHat },
  { value: "painter_decorator", label: "Painter", icon: PaintBucket },
  { value: "tiler_flooring", label: "Tiler & Flooring", icon: Grid3x3 },
  { value: "landscaping_gardening", label: "Landscaping", icon: Trees },
  { value: "handyman", label: "Handyman", icon: Drill },
  { value: "other", label: "Other", icon: MoreHorizontal },
];
```

### Valibot Schema for Profile Step
```typescript
// ProfileStep validation
import * as v from "valibot";

const profileSchema = v.object({
  displayName: v.pipe(
    v.string(),
    v.trim(),
    v.minLength(1, "Display name is required"),
  ),
});
```

### Valibot Schema for Business Step
```typescript
// BusinessStep validation
import * as v from "valibot";

const businessSchema = v.object({
  name: v.pipe(
    v.string(),
    v.trim(),
    v.minLength(1, "Business name is required"),
  ),
  primaryTrade: v.pipe(
    v.string(),
    v.minLength(1, "Please select your trade"),
  ),
  customTrade: v.optional(v.string()),
});
```

### RTK Query Mutation for Trial Start
```typescript
// In subscription API or a new onboarding API file
startTrial: builder.mutation<SubscriptionResponse, void>({
  query: () => ({
    url: "/v1/subscription/trial",
    method: "POST",
  }),
  invalidatesTags: ["Subscription"],
}),
```

### Business Creation Payload (hardcoded defaults for ONBD-03)
```typescript
const businessPayload = {
  name: formData.name,
  primaryTrade: formData.primaryTrade === "other" ? formData.customTrade : formData.primaryTrade,
  country: "GB",
  currency: "GBP",
};
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Old onboarding (modal dialogs + Redux middleware) | New onboarding (route guard + wizard page) | Phase 37 (this phase) | New users get proper wizard; old system coexists for safety until Phase 39 removes it |

**Deprecated/outdated:**
- `onboardingSlice.ts`, `onboardingMiddleware.ts`, `OnboardingDialogs.tsx`, `OnboardingProvider.tsx`, `OnboardingContext.ts`: OLD onboarding system. Do NOT import from these. Phase 39 removes them.

## Open Questions

1. **Existing subscription API hook for trial start**
   - What we know: POST /v1/subscription/trial is being built in Phase 35. The frontend needs an RTK Query mutation for it.
   - What's unclear: Whether Phase 35 has already added this mutation to the subscription API file, or if Phase 37 needs to create it.
   - Recommendation: Check if `startTrial` mutation exists in subscription API. If not, create it as part of this phase.

2. **Existing user PATCH mutation**
   - What we know: PATCH /v1/user exists for updating display name. There should be an existing RTK Query mutation in `userApi.ts`.
   - What's unclear: Exact shape of the existing mutation and whether it properly invalidates user data cache.
   - Recommendation: Verify the existing mutation and ensure it has proper tag invalidation for the user query.

3. **Business creation mutation existence**
   - What we know: POST /v1/business endpoint exists. There should be an RTK Query mutation in `businessApi.ts`.
   - What's unclear: Whether the existing mutation accepts the `primaryTrade` field.
   - Recommendation: Verify the existing mutation payload shape. The `trade` field is already on the API entity (D-14), so the mutation should already support it.

## Project Constraints (from CLAUDE.md)

- **Tech stack:** Must follow existing NestJS (API) and React/Vite (UI) patterns
- **Two repos:** This phase is frontend-only (trade-flow-ui), but must call API endpoints from trade-flow-api
- **Frontend conventions:** Feature-based module structure, RTK Query for API data, React Hook Form + Valibot for forms, Tailwind CSS with semantic tokens, cn() utility, mobile-first responsive design
- **No type assertions:** Prefer type guards/validation over `as` casts (from user memory)
- **Code style:** Functional components only, `import type` for type imports, interfaces for props, camelCase functions/variables, PascalCase components
- **Styling:** Use existing semantic color tokens (primary, destructive, muted-foreground, etc.), shadcn/ui components (Card, Button, Input, Label, Badge)

## Sources

### Primary (HIGH confidence)
- `.planning/phases/37-onboarding-wizard-pages/37-CONTEXT.md` -- all 28 implementation decisions
- `.planning/phases/37-onboarding-wizard-pages/37-UI-SPEC.md` -- full visual and interaction contract
- `.planning/phases/36-public-landing-page-and-route-restructure/36-CONTEXT.md` -- route guard architecture (D-05 through D-08)
- `.planning/phases/35-no-card-trial-api-endpoint/35-CONTEXT.md` -- trial endpoint specification
- `.planning/codebase/CONVENTIONS.md` -- frontend coding conventions
- `.planning/codebase/STRUCTURE.md` -- frontend directory structure
- `.planning/REQUIREMENTS.md` -- phase requirement definitions

### Secondary (MEDIUM confidence)
- Trade enum values inferred from D-11 and API codebase structure (`src/business/enums/primary-trade.enum.ts`)

### Tertiary (LOW confidence)
- Exact existence and shape of RTK Query mutations for user PATCH, business POST, and subscription trial POST -- needs verification during implementation

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- all libraries already installed, no new dependencies
- Architecture: HIGH -- follows established feature-based module pattern, all integration points clearly documented
- Pitfalls: HIGH -- based on concrete analysis of data flow and state management patterns in the codebase

**Research date:** 2026-04-02
**Valid until:** 2026-05-02 (stable -- no external dependency changes expected)
