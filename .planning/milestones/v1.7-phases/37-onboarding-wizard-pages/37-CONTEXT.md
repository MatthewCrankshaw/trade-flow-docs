# Phase 37: Onboarding Wizard Pages - Context

**Gathered:** 2026-04-01
**Status:** Ready for planning

<domain>
## Phase Boundary

Mandatory two-step onboarding wizard (profile name, then business name + trade) that gates app access for new users. Implements OnboardingGuard redirect logic (Phase 36 created the pass-through shell). Existing users with completed profiles and businesses bypass automatically. After business setup, auto-creates defaults and starts the 30-day no-card trial (Phase 35 endpoint). Trial status badge in the app header with mobile compact variant and urgency color shifts. Old onboarding system coexists until Phase 39 removes it.

</domain>

<decisions>
## Implementation Decisions

### Wizard Visual Design
- **D-01:** Centered card layout on a clean background — single card, one step at a time, focused and simple
- **D-02:** Subtle gradient background matching the existing loading page style
- **D-03:** Trade Flow logo centered above the wizard card
- **D-04:** Progress bar indicator — horizontal bar filling left-to-right with "Step 1 of 2" / "Step 2 of 2" text
- **D-05:** No back button on Step 2 — profile is saved after Step 1, user can edit display name later in Settings
- **D-06:** No animation between steps — instant content swap when advancing from Step 1 to Step 2
- **D-07:** Full-width on mobile (with padding), max-width ~480px on desktop — standard responsive pattern
- **D-08:** Friendly and direct copy tone — "What's your name?" / "Tell us about your business" — casual, conversational, matches tradesperson audience (Phase 36 D-01)
- **D-09:** Short subtitle under each step heading, e.g., "We just need a couple of things to get you started."

### Trade Selection (Step 2)
- **D-10:** Icon card grid with 9 predefined trades + "Other" card — NOT a dropdown
- **D-11:** Trade enum (already exists in API): PLUMBER, ELECTRICIAN, HEATING_GAS_ENGINEER, CARPENTER_JOINER, BUILDER_GENERAL_CONTRACTOR, PAINTER_DECORATOR, TILER_FLOORING, LANDSCAPING_GARDENING, HANDYMAN, OTHER
- **D-12:** Plain Lucide icons above trade names — no colored circles or badges, consistent with existing app icon usage
- **D-13:** Selecting "Other" highlights the card and reveals a text input BELOW the grid for custom trade name
- **D-14:** Trade field already exists on the business entity in the API — no schema changes needed for trade storage

### Post-Setup Loading
- **D-15:** Full-screen loading state replaces the wizard card — centered spinner + "Setting up your business..." message
- **D-16:** Two sequential API calls from frontend: 1) POST /v1/business (creates business + all defaults in one orchestrated endpoint), 2) POST /v1/subscription/trial (starts 30-day trial)
- **D-17:** Step 1 saves display name immediately via PATCH /v1/user when user clicks Continue — name persisted even if user abandons Step 2
- **D-18:** Error handling: replace loading state with error message + "Try again" retry button. Business may already exist, trial call may be what failed — retry should handle partial completion
- **D-19:** Success: sonner toast notification "You're all set! Your 30-day free trial has started." then redirect to dashboard

### Trial Status Display
- **D-20:** Desktop trial badge already exists in the header — this phase adds mobile compact variant and urgency color shifts
- **D-21:** Mobile compact format: "28d" (abbreviated number + "d") — must be visible but not obstruct other header elements
- **D-22:** Urgency color shifts: default/muted color normally, yellow/warning at <=10 days, red/destructive at <=3 days
- **D-23:** Clicking the trial badge opens Stripe Billing Portal directly (calls existing manage endpoint for portal URL, opens in new tab) — satisfies TRIAL-03
- **D-24:** Badge only visible when subscription status is "trialing" — disappears once user has active paid subscription
- **D-25:** Data source: existing GET /v1/subscription endpoint returns trialEnd date, frontend calculates days remaining

### OnboardingGuard Logic
- **D-26:** OnboardingGuard (Phase 36 pass-through shell) gets redirect logic: check if user has display name AND business — if not, redirect to /onboarding
- **D-27:** Wizard resumes at correct step on refresh (ONBD-06): if display name exists but no business, start at Step 2; if neither, start at Step 1
- **D-28:** Existing users with completed profile and business bypass onboarding entirely (ONBD-07) — guard renders Outlet

### Claude's Discretion
- Exact wizard card dimensions and padding
- Specific Lucide icon choices for each trade card
- Progress bar visual styling (color, height, border-radius)
- Loading spinner style and animation
- Exact copy for step subtitles and error messages
- Trade card grid responsive layout (columns per breakpoint)
- Urgency color exact values (use existing warning/destructive semantic tokens)
- Whether to use React Router nested routes or simple state for wizard steps

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### v1.7 Requirements
- `.planning/REQUIREMENTS.md` — Full v1.7 requirements; Phase 37 covers ONBD-01 through ONBD-07, TRIAL-02, TRIAL-03

### Roadmap
- `.planning/ROADMAP.md` §Phase 37 — Phase goal, success criteria, dependencies on Phase 35 and 36

### Prior Phase Context (REQUIRED — decisions carry forward)
- `.planning/phases/35-no-card-trial-api-endpoint/35-CONTEXT.md` — Trial endpoint (POST /v1/subscription/trial), rejection rules, auto-cancel behavior
- `.planning/phases/36-public-landing-page-and-route-restructure/36-CONTEXT.md` — OnboardingGuard shell, route tree structure (D-05 through D-08), bundle isolation, CTA flow

### Codebase Conventions
- `.planning/codebase/CONVENTIONS.md` — Frontend file naming, component structure, styling patterns
- `.planning/codebase/STRUCTURE.md` — Frontend directory structure, existing onboarding components, store slices
- `.planning/codebase/ARCHITECTURE.md` — Frontend data flow, Redux patterns, context providers

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `OnboardingGuard` (`src/features/auth/components/OnboardingGuard.tsx`): Phase 36 pass-through shell — Phase 37 adds redirect logic
- `Card`, `Button`, `Input` and shadcn/ui components (`src/components/ui/`): reuse for wizard card and form fields
- `lucide-react`: 560+ icons for trade card grid
- `sonner`: toast system for success notification after setup completion
- `DashboardLayout` header: existing trial badge component to extend with mobile compact format and urgency colors
- `useAuth` hook: get current user for display name check
- `useCurrentBusiness` hook: check if business exists for guard logic
- RTK Query `userApi.ts` and `api.ts`: existing API integration patterns
- `React Hook Form` + `valibot`: form validation for display name and business name fields

### Established Patterns
- Feature-based module structure: `src/features/` with components, hooks, api barrels
- RTK Query for server state, Redux slices for client state
- `cn()` utility for conditional Tailwind classes
- Mobile-first responsive design with `md:` breakpoint prefix
- Existing `onboardingSlice.ts` and `onboardingMiddleware.ts` (OLD system — coexists, Phase 39 removes)

### Integration Points
- `OnboardingGuard`: add redirect logic (check user profile + business existence)
- `App.tsx`: `/onboarding/*` routes already inside OnboardingGuard (Phase 36 D-08)
- `DashboardLayout` header: extend trial badge with mobile compact format and urgency color logic
- `src/features/onboarding/` or `src/pages/OnboardingPage.tsx`: new wizard page components
- Subscription API hooks: use existing GET /v1/subscription for trial status data

</code_context>

<specifics>
## Specific Ideas

- Background gradient should match the existing loading page style (user specified this explicitly)
- Desktop trial badge already exists — this phase adds mobile compact variant ("28d") and urgency color shifts, not a full rebuild
- Trade enum is already defined in the API — frontend must match the exact enum values: plumber, electrician, heating_gas_engineer, carpenter_joiner, builder_general_contractor, painter_decorator, tiler_flooring, landscaping_gardening, handyman, other
- POST /v1/business endpoint already orchestrates business creation + all defaults (tax rates, job types, visit types, items, quote email template) in a single call

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 37-onboarding-wizard-pages*
*Context gathered: 2026-04-01*
