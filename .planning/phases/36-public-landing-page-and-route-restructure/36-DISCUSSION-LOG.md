# Phase 36: Public Landing Page and Route Restructure - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-01
**Phase:** 36-public-landing-page-and-route-restructure
**Areas discussed:** Landing page content, Route guard architecture, Bundle isolation, Navigation and CTA flow

---

## Landing Page Content

### Hero Tone

| Option | Description | Selected |
|--------|-------------|----------|
| Direct and practical | Speak like a tradesperson thinks. No jargon, short sentences. | ✓ |
| Professional and polished | More formal, credibility-focused. | |
| Friendly and encouraging | Warm, empathy-forward. | |
| You decide | Claude picks based on target audience. | |

**User's choice:** Direct and practical
**Notes:** None

### Feature Highlights

| Option | Description | Selected |
|--------|-------------|----------|
| Jobs, Quotes, Scheduling | Three pillars of daily trade work. Maps to shipped features. | ✓ |
| Customers, Quotes, Invoices | Customer-to-payment flow. Invoicing not built yet. | |
| You decide | Claude selects most compelling features. | |

**User's choice:** Jobs, Quotes, Scheduling
**Notes:** None

### Pricing Section

| Option | Description | Selected |
|--------|-------------|----------|
| Single card | One plan: £6/month, 30 days free, no card required. No decision fatigue. | ✓ |
| Free vs Pro comparison | Show free tier alongside Pro. Misleading — no free tier exists. | |
| You decide | Claude designs based on single-plan model. | |

**User's choice:** Single card
**Notes:** None

### Visual Elements

| Option | Description | Selected |
|--------|-------------|----------|
| Icons only | Lucide icons for each feature. Clean, fast-loading, no maintenance. | ✓ |
| Product screenshots | More compelling but requires maintenance. | |
| Illustrations | Most polished but requires design assets. | |
| You decide | Claude chooses based on available assets. | |

**User's choice:** Icons only
**Notes:** None

---

## Route Guard Architecture

### Guard Nesting

| Option | Description | Selected |
|--------|-------------|----------|
| ProtectedRoute > OnboardingGuard > PaywallGuard | Three nested layers. Landing/login outside all guards. | ✓ |
| Flat guards with redirect chain | Independent guards. Risk of redirect loops. | |
| You decide | Claude designs architecture. | |

**User's choice:** ProtectedRoute > OnboardingGuard > PaywallGuard
**Notes:** User selected the preview showing the full route tree structure

### Shell Implementation

| Option | Description | Selected |
|--------|-------------|----------|
| Pass-through shells | Render <Outlet /> only. Phase 37/38 add real logic. | ✓ |
| Stub with feature flags | Env-var toggleable. More complex. | |
| You decide | Claude decides based on dependency chain. | |

**User's choice:** Pass-through shells
**Notes:** None

### Existing SubscriptionGatedLayout

| Option | Description | Selected |
|--------|-------------|----------|
| Keep as-is, Phase 38 replaces | Don't touch soft paywall. Phase 38 replaces with hard paywall. | ✓ |
| Restructure now | Move logic into PaywallGuard shell. Risks breaking soft paywall. | |
| You decide | Claude decides based on risk. | |

**User's choice:** Keep as-is, Phase 38 replaces
**Notes:** None

---

## Bundle Isolation

### Isolation Approach

| Option | Description | Selected |
|--------|-------------|----------|
| React.lazy + code splitting | Lazy-loaded route. Vite auto code-splits. No store/features/hooks imports. | ✓ |
| Separate entry point | Multi-page Vite app. Maximum isolation but adds build complexity. | |
| You decide | Claude chooses approach. | |

**User's choice:** React.lazy + code splitting
**Notes:** User selected the preview showing the import pattern

### Verification Method

| Option | Description | Selected |
|--------|-------------|----------|
| Vite build + visualizer | Manual chunk inspection. One-time verification. | ✓ |
| Import lint rule | ESLint rule blocking forbidden imports. Enforced on every build. | |
| You decide | Claude chooses verification approach. | |

**User's choice:** Vite build + visualizer
**Notes:** None

---

## Navigation and CTA Flow

### CTA Target

| Option | Description | Selected |
|--------|-------------|----------|
| Go to /login (sign-up tab) | Navigate to existing login page in sign-up mode. After sign-up, onboarding wizard flow. | ✓ |
| Go to dedicated /signup page | New page. Duplicates AuthForm. | |
| You decide | Claude decides based on existing auth flow. | |

**User's choice:** Go to /login (sign-up tab)
**Notes:** User selected the preview showing the full user flow from landing to dashboard

### Landing Page Header

| Option | Description | Selected |
|--------|-------------|----------|
| Minimal header | Logo left, Log in + Start Trial buttons right. Separate from DashboardLayout. | ✓ |
| No header | Full-page layout. CTA in sections only. Log in less discoverable. | |
| You decide | Claude designs layout. | |

**User's choice:** Minimal header
**Notes:** User selected the ASCII mockup preview

### Auth Redirect

| Option | Description | Selected |
|--------|-------------|----------|
| Check auth in LandingPage component | Firebase onAuthStateChanged directly. Navigate to /dashboard if authenticated. Brief spinner. | ✓ |
| Route-level redirect in App.tsx | Wrapper component checks auth. Keeps LandingPage pure. | |
| You decide | Claude decides redirect mechanism. | |

**User's choice:** Check auth in LandingPage component
**Notes:** User selected the preview showing the Firebase auth check pattern

---

## Claude's Discretion

- Exact hero section copy and headline
- Feature highlight card layout
- Pricing card visual treatment
- Landing page footer content
- Suspense fallback design
- Mobile responsive breakpoints
- Sign-up mode mechanism for /login navigation

## Deferred Ideas

None — discussion stayed within phase scope.
