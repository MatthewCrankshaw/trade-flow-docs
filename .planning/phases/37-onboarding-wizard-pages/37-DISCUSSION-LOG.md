# Phase 37: Onboarding Wizard Pages - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-01
**Phase:** 37-onboarding-wizard-pages
**Areas discussed:** Wizard visual design, Trade selection, Post-setup loading, Trial status display

---

## Wizard Visual Design

### Layout Style

| Option | Description | Selected |
|--------|-------------|----------|
| Centered card | Single card centered on a clean background, one step at a time. Simple, focused. | ✓ |
| Full-width page | Form fields take the full content area, no card wrapper. More spacious but less focused. | |
| Split layout | Left side: branding/illustration area. Right side: form. More visually rich but heavier to build. | |

**User's choice:** Centered card
**Notes:** None

### Progress Indicator

| Option | Description | Selected |
|--------|-------------|----------|
| Progress bar | Simple horizontal bar filling left-to-right. "Step 1 of 2" text above. Minimal, works for 2 steps. | ✓ |
| Numbered circles | Two numbered circles connected by a line, active step highlighted. May be overkill for 2 steps. | |
| You decide | Let Claude pick | |

**User's choice:** Progress bar
**Notes:** None

### Back Button (Step 2)

| Option | Description | Selected |
|--------|-------------|----------|
| Yes, back button | User can go back to change display name before finishing. More forgiving. | |
| No back button | Profile is saved after Step 1. User can edit later in Settings. Simpler flow. | ✓ |
| You decide | Let Claude pick | |

**User's choice:** No back button
**Notes:** None

### Background

| Option | Description | Selected |
|--------|-------------|----------|
| Plain background | Clean bg-background color, no decoration. | |
| Subtle gradient | Gentle gradient or slight pattern behind the card. | ✓ |
| You decide | Let Claude pick | |

**User's choice:** Subtle gradient similar to the loading page
**Notes:** User specified it should match the existing loading page gradient style

### Branding

| Option | Description | Selected |
|--------|-------------|----------|
| Logo above the card | Trade Flow name/logo centered above wizard card. | ✓ |
| No branding | Just the wizard card, no logo. | |
| You decide | Let Claude pick | |

**User's choice:** Logo above the card
**Notes:** None

### Step Transitions

| Option | Description | Selected |
|--------|-------------|----------|
| No animation | Instant swap to Step 2 content. Simple, fast. | ✓ |
| Subtle fade/slide | Content fades or slides left-to-right between steps. | |
| You decide | Let Claude choose | |

**User's choice:** No animation
**Notes:** None

### Responsive Width

| Option | Description | Selected |
|--------|-------------|----------|
| Full-width mobile, max-width desktop | Card fills screen on mobile, constrained to ~480px on desktop. | ✓ |
| You decide | Let Claude handle responsive breakpoints | |

**User's choice:** Full-width mobile, max-width desktop
**Notes:** None

### Copy Tone

| Option | Description | Selected |
|--------|-------------|----------|
| Friendly and direct | "What's your name?" / "Tell us about your business" — casual, conversational | ✓ |
| Formal | "Enter your display name" / "Business details" — more corporate | |
| You decide | Let Claude pick | |

**User's choice:** Friendly and direct
**Notes:** None

### Welcome Text

| Option | Description | Selected |
|--------|-------------|----------|
| Short subtitle | One line under heading, e.g., "We just need a couple of things to get you started." | ✓ |
| No subtitle | Just heading and form fields. Ultra-minimal. | |
| You decide | Let Claude decide | |

**User's choice:** Short subtitle
**Notes:** None

---

## Trade Selection

### Trade UI Style

| Option | Description | Selected |
|--------|-------------|----------|
| Dropdown select | Standard select/combobox dropdown. Compact, works for 10-20+ trades. | |
| Icon card grid | Grid of cards with Lucide icons, one per trade. Visual and engaging. | ✓ |
| Searchable combobox | Type-ahead search with dropdown. Good for long lists. | |

**User's choice:** Icon card grid
**Notes:** None

### Trade List

| Option | Description | Selected |
|--------|-------------|----------|
| Core 6 + Other | Plumber, Electrician, Builder, Carpenter, Painter & Decorator, Roofer, plus Other | |
| Expanded 10+ trades | Add Locksmith, Landscaper, Plasterer, Tiler, Gas Engineer, etc. | |
| You decide | Let Claude pick | |

**User's choice:** 9 trades + Other (custom enum provided)
**Notes:** User provided exact enum: PLUMBER, ELECTRICIAN, HEATING_GAS_ENGINEER, CARPENTER_JOINER, BUILDER_GENERAL_CONTRACTOR, PAINTER_DECORATOR, TILER_FLOORING, LANDSCAPING_GARDENING, HANDYMAN, OTHER

### Custom Trade Input

| Option | Description | Selected |
|--------|-------------|----------|
| Yes, text input on select | Selecting "Other" shows text field for custom trade name | ✓ |
| No, just "Other" | "Other" is label only, no custom input | |

**User's choice:** Yes, text input on select
**Notes:** None

### API Storage

| Option | Description | Selected |
|--------|-------------|----------|
| Store on business | Add trade field to business entity in API | ✓ |
| Frontend-only | Trade only used during onboarding, not persisted | |

**User's choice:** Store on business — trade field already exists in the API
**Notes:** No API schema changes needed

### Icon Style

| Option | Description | Selected |
|--------|-------------|----------|
| Icon in colored circle | Soft-colored circle background behind Lucide icon | |
| Plain icon | Just Lucide icon above trade name, no decoration | ✓ |
| You decide | Let Claude pick | |

**User's choice:** Plain icon
**Notes:** None

### "Other" Text Input Placement

| Option | Description | Selected |
|--------|-------------|----------|
| Text field below grid | Selecting "Other" highlights card, reveals text input below grid | ✓ |
| Inline in card | Card expands to include text input | |
| You decide | Let Claude pick | |

**User's choice:** Text field below grid
**Notes:** None

---

## Post-Setup Loading

### Loading UX

| Option | Description | Selected |
|--------|-------------|----------|
| Full-screen loading | Replace wizard with centered loading state — spinner + message | ✓ |
| Stay on wizard + button loading | Keep wizard visible, disable button with spinner | |
| Animated progress steps | Checklist ticking off each operation | |

**User's choice:** Full-screen loading
**Notes:** None

### API Flow

| Option | Description | Selected |
|--------|-------------|----------|
| Sequential frontend calls | Frontend calls POST /v1/business then POST /v1/subscription/trial | ✓ |
| Single orchestration endpoint | New backend endpoint handles everything in one call | |
| You decide | Let Claude determine | |

**User's choice:** POST /v1/business already orchestrates business + defaults in one call. Then POST /v1/subscription/trial.
**Notes:** User clarified the create business endpoint already handles orchestration

### Error Handling

| Option | Description | Selected |
|--------|-------------|----------|
| Show error + retry button | Replace loading with error + "Try again" button | ✓ |
| Toast error + redirect | Show toast, redirect to dashboard anyway | |
| You decide | Let Claude pick | |

**User's choice:** Show error + retry button
**Notes:** None

### Success Notification

| Option | Description | Selected |
|--------|-------------|----------|
| Toast notification | Sonner toast: "You're all set! Your 30-day free trial has started." | ✓ |
| No notification | Silent redirect, Phase 39 welcome handles greeting | |
| You decide | Let Claude decide | |

**User's choice:** Toast notification
**Notes:** None

### Step 1 Save Timing

| Option | Description | Selected |
|--------|-------------|----------|
| Save immediately on Continue | PATCH /v1/user when clicking Continue on Step 1 | ✓ |
| Batch with business | Hold name in state, send together with business creation | |

**User's choice:** Save immediately on Continue
**Notes:** None

---

## Trial Status Display

### Header Presentation

| Option | Description | Selected |
|--------|-------------|----------|
| Text badge | Small badge/pill: "Trial: 28 days left" — clicking opens billing portal | ✓ |
| Banner below header | Thin strip below header with trial info and CTA | |
| Icon + tooltip | Clock icon, hover shows details | |
| You decide | Let Claude pick | |

**User's choice:** Text badge
**Notes:** Desktop trial badge already exists

### Badge Click Behavior

| Option | Description | Selected |
|--------|-------------|----------|
| Open Stripe Billing Portal | Calls manage endpoint, opens portal in new tab | ✓ |
| Open dropdown/popover | Shows popover with info + button to portal | |
| Navigate to /settings | Goes to settings/subscription page | |

**User's choice:** Open Stripe Billing Portal directly
**Notes:** None

### Urgency Color Shifts

| Option | Description | Selected |
|--------|-------------|----------|
| Yes, color shift | Default normally, yellow at <=10 days, red at <=3 days | ✓ |
| No color change | Same style regardless of days remaining | |
| You decide | Let Claude decide thresholds | |

**User's choice:** Yes, color shift
**Notes:** None

### Visibility Scope

| Option | Description | Selected |
|--------|-------------|----------|
| Trial only | Badge shows only when status is "trialing" | ✓ |
| Always visible with status | Shows subscription status for all users | |

**User's choice:** Trial only
**Notes:** None

### Mobile Format

| Option | Description | Selected |
|--------|-------------|----------|
| Short text "28d" | Abbreviated number + "d" on mobile. Full text on desktop. | ✓ |
| Icon + number | Clock icon + "28" with no text | |
| You decide | Let Claude pick | |

**User's choice:** Short text "28d"
**Notes:** User noted desktop badge already exists; this phase adds mobile compact variant. Must be visible but not obstruct other elements.

### Data Source

| Option | Description | Selected |
|--------|-------------|----------|
| Existing GET /v1/subscription | Returns trialEnd date, frontend calculates days remaining | ✓ |
| You decide | Let Claude determine | |

**User's choice:** Existing GET /v1/subscription
**Notes:** None

---

## Claude's Discretion

- Exact wizard card dimensions and padding
- Specific Lucide icon choices for each trade card
- Progress bar visual styling (color, height, border-radius)
- Loading spinner style and animation
- Exact copy for step subtitles and error messages
- Trade card grid responsive layout (columns per breakpoint)
- Urgency color exact values
- Whether to use React Router nested routes or simple state for wizard steps

## Deferred Ideas

None — discussion stayed within phase scope.
