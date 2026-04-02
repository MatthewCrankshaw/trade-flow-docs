# Phase 39: Welcome Dashboard and Final Cleanup - Research

**Researched:** 2026-04-02
**Domain:** React dashboard widget, onboarding progress tracking, old system removal
**Confidence:** HIGH

## Summary

Phase 39 delivers a personalised welcome experience on the dashboard for new users who have just completed the Phase 37 onboarding wizard, and removes all remnants of the pre-Phase 37 onboarding system. The phase has two distinct work streams: (1) a welcome message and getting-started checklist widget at the top of the dashboard page, and (2) a clean sweep removal of the old onboarding infrastructure (Redux slice, middleware, context providers, dialog components, and related hooks).

The welcome widget uses the user's display name (saved in Phase 37 Step 1) for a personalised greeting and shows two checklist items: "Create your first job" and "Send your first quote." Per D-07, completion state comes from the onboarding progress record -- however, the existing `IOnboardingProgressDto` only has a `completedAt` field and does not track granular step completion for "first job" or "first quote." This is a critical implementation gap: either the API needs new fields on the onboarding progress record, or the frontend must determine completion from existing data (job/quote counts). D-07 explicitly says NOT to use direct job/quote count queries, so the API likely needs extension. This is flagged as an open question.

The old onboarding removal follows the same systematic pattern as Phase 38's soft paywall removal (D-12 through D-15 in 38-CONTEXT.md). The files to remove are well-documented in the existing codebase structure.

**Primary recommendation:** Build the welcome widget as a self-contained `WelcomeSection` component in `src/features/dashboard/components/`, conditionally rendered on the dashboard page when onboarding checklist items are incomplete. Address the onboarding progress data gap first (API extension or alternative approach), then build the UI, then perform the old onboarding clean sweep last.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- D-01: Name-based greeting using display name from onboarding Step 1 -- e.g., "Welcome, Matt! Let's get your business up and running."
- D-02: Casual and direct tone -- matches existing tradesperson audience tone established in Phase 36 D-01 and Phase 37 D-08
- D-03: Welcome message disappears when all checklist items are complete -- tied to widget lifecycle, not tracked separately
- D-04: Card component with checkbox-style items -- checked items show completion. Uses existing Card component pattern.
- D-05: Two checklist items: "Create your first job" and "Send your first quote" -- each links to the relevant page
- D-06: Each item has a title + short description (e.g., "Create your first job" with subtitle "Add a job to start tracking your work")
- D-07: Completion detection uses the onboarding progress record -- NOT direct job/quote count queries or new backend fields
- D-08: Widget disappears entirely once all checklist items are complete -- no celebration animation, no collapsed state
- D-09: Remove the old pre-Phase 37 onboarding system -- old wizard components, old routes, old onboarding state
- D-10: Researcher should discover the full removal inventory from the codebase -- no pre-known list of files
- D-11: Welcome message and checklist card positioned at the top of the dashboard page, above any existing content
- D-12: Below the welcome widget, existing dashboard content (including empty states) renders as normal -- welcome widget is additive, not a takeover
- D-13: Once checklist is complete, dashboard is the standard dashboard with no welcome remnants -- returning users see nothing special

### Claude's Discretion
- Welcome message exact copy (subtitle wording, greeting format)
- Checklist item descriptions exact wording
- Card dimensions, padding, spacing relative to existing dashboard content
- Checkbox visual styling (icon, color, animation on completion)
- How links work on checklist items (whole row clickable vs button)
- Responsive layout of the checklist card (mobile vs desktop)
- Transition when widget disappears (instant vs fade)
- Order of old onboarding removal operations

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| DASH-01 | New user sees a personalised welcome message on first dashboard visit | WelcomeSection component with greeting using displayName from user profile (D-01, D-11) |
| DASH-02 | Dashboard shows getting-started widget with "Create your first job" and "Send your first quote" checklist items | GettingStartedChecklist component with two ChecklistItem rows linked to /jobs and /quotes (D-04, D-05, D-06) |
| DASH-03 | Getting-started checklist items link to relevant pages and show completion state | ChecklistItem wraps incomplete items in Link, completed items show strikethrough with check icon; completion from onboarding progress record (D-05, D-07, D-08) |
</phase_requirements>

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| React | 19.2.0 | UI framework | Already installed, project standard |
| react-router-dom | 7.13.0 | Link navigation for checklist items | Already installed |
| @reduxjs/toolkit | 2.11.2 | RTK Query for fetching onboarding progress | Already installed |
| lucide-react | 0.563.0 | Check, Circle, Briefcase, FileText, ChevronRight icons | Already installed |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| tailwind-merge | 3.4.0 | Merging conditional Tailwind classes | Already installed, used via cn() |
| clsx | 2.1.1 | Conditional class names | Already installed, used via cn() |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Onboarding progress API query | Direct job/quote count queries | Explicitly rejected by D-07; progress record is single source of truth |

**Installation:** No new packages needed. All dependencies already installed.

## Architecture Patterns

### Recommended Project Structure
```
src/
├── features/
│   └── dashboard/
│       ├── components/
│       │   ├── WelcomeSection.tsx           # Container: greeting + checklist card
│       │   ├── GettingStartedChecklist.tsx   # Card with checklist items
│       │   ├── ChecklistItem.tsx             # Individual checklist row
│       │   └── index.ts                     # Barrel exports
│       ├── hooks/
│       │   ├── useGettingStarted.ts          # Fetches onboarding progress, derives checklist state
│       │   └── index.ts
│       └── index.ts                          # Feature barrel
│
├── pages/
│   └── DashboardPage.tsx                     # MODIFY: add WelcomeSection at top
```

### Pattern 1: Conditional Welcome Widget
**What:** Dashboard page conditionally renders welcome section based on checklist completion state
**When to use:** When a widget should appear only for new users who haven't completed getting-started items
**Example:**
```typescript
// DashboardPage.tsx
export function DashboardPage() {
  const { user } = useAuth();
  const { isChecklistComplete, isLoading, items } = useGettingStarted();

  return (
    <div className="space-y-6">
      {!isLoading && !isChecklistComplete && (
        <WelcomeSection
          displayName={user?.displayName ?? ""}
          items={items}
        />
      )}
      {/* Existing dashboard content below */}
    </div>
  );
}
```

### Pattern 2: Checklist Item with Link/Static Split
**What:** Incomplete items are clickable Links; complete items are non-interactive divs
**When to use:** Checklist rows where completed items should not be navigable
**Example:**
```typescript
// ChecklistItem.tsx (per UI-SPEC)
interface ChecklistItemProps {
  title: string;
  description: string;
  icon: LucideIcon;
  isComplete: boolean;
  linkTo: string;
}

export function ChecklistItem({ title, description, icon: Icon, isComplete, linkTo }: ChecklistItemProps) {
  if (isComplete) {
    return (
      <li className="flex items-center gap-4 p-3 rounded-lg">
        <div className="flex items-center justify-center h-8 w-8 rounded-full bg-primary/10">
          <Check className="h-4 w-4 text-primary" />
        </div>
        <div className="flex-1 min-w-0">
          <p className="text-sm font-medium text-muted-foreground line-through">{title}</p>
        </div>
      </li>
    );
  }

  return (
    <li>
      <Link to={linkTo} className="flex items-center gap-4 p-3 rounded-lg hover:bg-muted/50 transition-colors cursor-pointer">
        <div className="flex items-center justify-center h-8 w-8 rounded-full border border-muted-foreground/40">
          <Icon className="h-4 w-4 text-muted-foreground" />
        </div>
        <div className="flex-1 min-w-0">
          <p className="text-sm font-medium text-foreground">{title}</p>
          <p className="text-sm text-muted-foreground">{description}</p>
        </div>
        <ChevronRight className="h-4 w-4 text-muted-foreground" />
      </Link>
    </li>
  );
}
```

### Pattern 3: Onboarding Progress Hook
**What:** Custom hook that fetches onboarding progress and derives checklist completion state
**When to use:** Dashboard page needs to know if checklist items are complete
**Example:**
```typescript
// useGettingStarted.ts
export function useGettingStarted() {
  // Fetch onboarding progress from API
  // The exact query depends on how the API exposes this data
  // See Open Questions section for the data model gap

  return {
    isLoading,
    isChecklistComplete: hasCreatedFirstJob && hasSentFirstQuote,
    items: [
      { key: "job", isComplete: hasCreatedFirstJob },
      { key: "quote", isComplete: hasSentFirstQuote },
    ],
  };
}
```

### Anti-Patterns to Avoid
- **Do NOT query job/quote collections directly for completion state:** D-07 explicitly says to use the onboarding progress record, not count queries. Even though counting jobs > 0 would work, it contradicts the decision.
- **Do NOT show the welcome widget while data is loading:** Per UI-SPEC interaction states, the welcome section is not rendered during loading to avoid layout shift.
- **Do NOT import from old onboarding system in new dashboard components:** The old `onboardingSlice`, `OnboardingDialogs`, `onboardingMiddleware` are being removed in this phase. New components must not depend on them.
- **Do NOT animate widget disappearance:** D-08 says no celebration animation, no collapsed state. Widget simply stops rendering.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Checklist state management | Custom localStorage tracking | RTK Query + onboarding progress API | Server-side state persists across devices and sessions |
| Icon rendering | Custom SVG components | lucide-react (Check, Circle, Briefcase, FileText, ChevronRight) | Already installed with 560+ icons |
| Conditional class names | Manual string concatenation | cn() utility from @/lib/utils | Established project pattern |
| Navigation links | Manual onClick with navigate | react-router-dom Link component | Standard for client-side navigation |
| Card layout | Custom card styling | shadcn/ui Card, CardHeader, CardContent | Already installed and used throughout app |

**Key insight:** This phase introduces no new libraries. Every component pattern already exists in the codebase.

## Old Onboarding System Removal Inventory

Per D-09 and D-10, the old pre-Phase 37 onboarding system must be removed. Based on codebase structure analysis:

### Files to Remove (OLD system)

| File | Type | What It Does |
|------|------|-------------|
| `src/components/onboarding/OnboardingDialogs.tsx` | Component | Manages which onboarding dialog to show (old wizard) |
| `src/components/onboarding/PrerequisiteAlert.tsx` | Component | Warning for missing prerequisites |
| `src/components/onboarding/` directory | Directory | All old onboarding dialog components |
| `src/store/slices/onboardingSlice.ts` | Redux slice | Old onboarding dialog state (isOpen, currentStep) |
| `src/store/middleware/onboardingMiddleware.ts` | Middleware | Watches auth/business changes to auto-open dialogs |
| `src/contexts/OnboardingProvider.tsx` | Provider | Old onboarding context provider |
| `src/contexts/OnboardingContext.ts` | Context | Old onboarding context type |
| `src/contexts/OnboardingContext.types.ts` | Types | Old onboarding type definitions |
| `src/hooks/useOnboarding.ts` | Hook | Get/set old onboarding dialog state |
| `src/store/utils/onboarding-storage.ts` | Utility | Old onboarding localStorage persistence |

### Files to Modify (remove old onboarding references)

| File | Modification |
|------|-------------|
| `src/App.tsx` | Remove `OnboardingDialogs` import and render |
| `src/store/index.ts` | Remove `onboardingSlice` reducer and `onboardingMiddleware` from store config |
| Any feature component importing `openOnboarding()` | Remove dispatch calls and imports |

### Verification After Removal

- `onboardingSlice` not imported anywhere
- `onboardingMiddleware` not imported anywhere
- `OnboardingDialogs` not imported anywhere
- `OnboardingProvider` not imported anywhere
- `OnboardingContext` not imported anywhere
- `useOnboarding` not imported anywhere
- Redux store shape no longer has `onboarding` key
- TypeScript compilation passes
- No references to old onboarding storage utility

**Confidence:** HIGH -- file locations from STRUCTURE.md and ARCHITECTURE.md codebase analysis. Exact file list should be verified during implementation by grepping for imports.

## Common Pitfalls

### Pitfall 1: Onboarding Progress Data Model Gap
**What goes wrong:** The welcome checklist needs "has created first job" and "has sent first quote" boolean states, but the existing `IOnboardingProgressDto` only has `completedAt: DateTime | null`.
**Why it happens:** The original onboarding progress record was designed for the business-creation wizard flow, not for post-onboarding getting-started tracking.
**How to avoid:** Investigate the actual API onboarding progress endpoint during implementation. Either (a) the `onboarding-step.enum.ts` already includes granular steps like JOB_CREATED/QUOTE_SENT that the `OnboardingProgressUpdater` tracks, or (b) the API needs new fields added. If (b), this becomes a two-repo change requiring API work before UI work.
**Warning signs:** RTK Query returns a progress object without job/quote completion fields.

### Pitfall 2: Layout Shift When Welcome Widget Disappears
**What goes wrong:** When both checklist items are completed and the widget disappears, the dashboard content jumps up, causing a jarring visual shift.
**Why it happens:** The welcome section occupies vertical space; removing it shifts everything up.
**How to avoid:** Per UI-SPEC, the widget disappears entirely with no animation (D-08). The shift is expected and intentional -- don't try to prevent it. The welcome section is only shown to new users who haven't completed the checklist, so disappearance happens once. No smooth transition needed.
**Warning signs:** Attempting to add fade-out animation (contradicts D-08).

### Pitfall 3: Stale Progress Data After Job/Quote Creation
**What goes wrong:** User creates their first job on the /jobs page, navigates back to dashboard, but the checklist still shows the item as incomplete.
**Why it happens:** RTK Query caches the onboarding progress data. If the API updates the progress record when a job is created, the cached data on the frontend may be stale.
**How to avoid:** Either (a) invalidate the onboarding progress cache when jobs/quotes are created (add tag invalidation to job/quote mutation endpoints), or (b) refetch progress data when the dashboard page mounts (use `refetchOnMountOrArgChange` or manual refetch). Option (a) is cleaner.
**Warning signs:** User must manually refresh the page to see updated checklist state.

### Pitfall 4: Removing Old System Breaks Existing Users
**What goes wrong:** Removing the old onboarding Redux slice causes runtime errors because existing code references `state.onboarding`.
**Why it happens:** Not all references to the old system were found and removed.
**How to avoid:** Grep for ALL references to old onboarding imports before removing files. Check: `onboardingSlice`, `openOnboarding`, `closeOnboarding`, `OnboardingDialogs`, `OnboardingProvider`, `OnboardingContext`, `useOnboarding`, `onboardingMiddleware`, `onboarding-storage`. Remove all consumers before removing the providers.
**Warning signs:** TypeScript compilation errors after removal (which is actually the desired feedback -- let the compiler guide the cleanup).

### Pitfall 5: Welcome Widget Shown to Existing Users
**What goes wrong:** Users who had businesses and jobs before v1.7 see the welcome widget on their dashboard.
**Why it happens:** If the onboarding progress record is only created during the Phase 37 wizard flow, existing users who bypassed onboarding (D-28) may not have a progress record at all.
**How to avoid:** The `useGettingStarted` hook must handle the case where no onboarding progress record exists. For existing users who bypassed onboarding (they already have a business), treat "no progress record" as "checklist complete" -- don't show the welcome widget. Only show it when a progress record exists AND has incomplete items.
**Warning signs:** Long-time users suddenly see "Welcome, Matt!" on their dashboard after the update.

## Code Examples

### WelcomeSection Component
```typescript
// src/features/dashboard/components/WelcomeSection.tsx
// Source: UI-SPEC layout specification

import { Card, CardHeader, CardContent } from "@/components/ui/card";
import { GettingStartedChecklist } from "./GettingStartedChecklist";

interface WelcomeSectionProps {
  displayName: string;
  items: Array<{ key: string; isComplete: boolean }>;
}

export function WelcomeSection({ displayName, items }: WelcomeSectionProps) {
  return (
    <div className="space-y-4">
      <div>
        <h1 className="text-lg font-semibold">Welcome, {displayName}!</h1>
        <p className="text-sm text-muted-foreground mt-1">
          Let's get your business up and running.
        </p>
      </div>
      <Card>
        <CardHeader className="px-4 md:px-6 pt-4 md:pt-6 pb-0">
          <h2 className="text-sm font-semibold">Getting started</h2>
        </CardHeader>
        <CardContent className="px-4 md:px-6 py-4 md:py-6">
          <GettingStartedChecklist items={items} />
        </CardContent>
      </Card>
    </div>
  );
}
```

### GettingStartedChecklist Component
```typescript
// src/features/dashboard/components/GettingStartedChecklist.tsx
import { Briefcase, FileText } from "lucide-react";
import { ChecklistItem } from "./ChecklistItem";

const CHECKLIST_CONFIG = [
  {
    key: "job",
    title: "Create your first job",
    description: "Add a job to start tracking your work",
    icon: Briefcase,
    linkTo: "/jobs",
  },
  {
    key: "quote",
    title: "Send your first quote",
    description: "Send a quote to a customer for approval",
    icon: FileText,
    linkTo: "/quotes",
  },
] as const;

interface GettingStartedChecklistProps {
  items: Array<{ key: string; isComplete: boolean }>;
}

export function GettingStartedChecklist({ items }: GettingStartedChecklistProps) {
  return (
    <ul className="space-y-2">
      {CHECKLIST_CONFIG.map((config) => {
        const item = items.find((i) => i.key === config.key);
        return (
          <ChecklistItem
            key={config.key}
            title={config.title}
            description={config.description}
            icon={config.icon}
            isComplete={item?.isComplete ?? false}
            linkTo={config.linkTo}
          />
        );
      })}
    </ul>
  );
}
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Old onboarding (modal dialogs + Redux middleware + context providers) | New onboarding (route guard + wizard page, Phase 37) | Phase 37 | Phase 39 removes the old system entirely |
| No dashboard getting-started experience | Welcome widget with personalised greeting + checklist | Phase 39 (this phase) | New users get guided first steps |

**Deprecated/outdated:**
- `onboardingSlice.ts`: OLD system -- being removed in this phase
- `onboardingMiddleware.ts`: OLD system -- being removed in this phase
- `OnboardingDialogs.tsx`: OLD system -- being removed in this phase
- `OnboardingProvider.tsx`: OLD system -- being removed in this phase
- `OnboardingContext.ts` / `OnboardingContext.types.ts`: OLD system -- being removed in this phase
- `useOnboarding.ts`: OLD system -- being removed in this phase
- `onboarding-storage.ts`: OLD system -- being removed in this phase

## Open Questions

1. **Onboarding Progress Record Data Model**
   - What we know: The existing `IOnboardingProgressDto` has `completedAt: DateTime | null`. The `OnboardingProgressUpdater` service is called during business creation. There is an `onboarding-step.enum.ts` in the API.
   - What's unclear: Does the onboarding progress record already track granular steps like "first job created" and "first quote sent"? Or does it only track business-setup completion? D-07 says to use this record for checklist completion, but we don't know if it has the right fields.
   - Recommendation: During implementation, the executor MUST read the actual API files: `trade-flow-api/src/user/enums/onboarding-step.enum.ts`, `trade-flow-api/src/user/data-transfer-objects/onboarding-progress.dto.ts`, `trade-flow-api/src/user/utilities/onboarding-progress-updater.service.ts`, and `trade-flow-api/src/user/repositories/onboarding-progress.repository.ts`. If the record does NOT have fields for job/quote completion, the planner must add an API extension task (new fields on the progress record + update the progress updater to mark them when jobs/quotes are created). This would make Phase 39 a two-repo phase.

2. **Dashboard Page Current Content**
   - What we know: `DashboardPage.tsx` exists at `src/pages/DashboardPage.tsx`.
   - What's unclear: What content currently exists on the dashboard page. The welcome widget goes "at the top, above any existing content" (D-11).
   - Recommendation: Executor reads the existing `DashboardPage.tsx` before modifying it to understand current layout and where to insert the welcome section.

3. **RTK Query Endpoint for Onboarding Progress**
   - What we know: The API has an onboarding progress record accessible via the user module.
   - What's unclear: Whether an RTK Query endpoint already exists for fetching onboarding progress, or if one needs to be created.
   - Recommendation: Check `src/services/userApi.ts` and any onboarding-related API files for existing progress query endpoints. If none exist, create one.

## UI-SPEC Reference

Phase 39 has a full UI-SPEC at `.planning/phases/39-welcome-dashboard-and-final-cleanup/39-UI-SPEC.md`. The planner MUST reference this for exact layout specifications, copy, interaction states, spacing, color tokens, and accessibility requirements. Key specifications:

- Components: `WelcomeSection`, `GettingStartedChecklist`, `ChecklistItem`
- Icons: `Check`, `Circle`, `Briefcase`, `FileText`, `ChevronRight`
- Copy: "Welcome, {displayName}!" / "Let's get your business up and running." / "Getting started" / "Create your first job" / "Send your first quote"
- Layout: Detailed pseudo-JSX in UI-SPEC Layout Specifications section
- Accessibility: `<h1>` for greeting, `<h2>` for card heading, `<ul>/<li>` for checklist, `aria-label` for check/incomplete icons, Link wrapping for keyboard navigation
- Responsive: Mobile px-4 card padding, desktop px-6 card padding

## Project Constraints (from CLAUDE.md)

- **Tech stack:** Must follow existing NestJS (API) and React/Vite (UI) patterns
- **Two repos:** This phase is primarily frontend (trade-flow-ui), but MAY require API changes if onboarding progress record needs new fields
- **Frontend conventions:** Feature-based module structure, RTK Query for API data, Tailwind CSS with semantic tokens, cn() utility, mobile-first responsive design
- **No type assertions:** Prefer type guards/validation over `as` casts (from user memory)
- **Code style:** Functional components only, `import type` for type imports, interfaces for props, camelCase functions/variables, PascalCase components
- **Styling:** Use existing semantic color tokens (primary, muted-foreground, etc.), shadcn/ui Card components
- **GSD workflow:** Use GSD entry points for code changes, not direct edits

## Sources

### Primary (HIGH confidence)
- `.planning/phases/39-welcome-dashboard-and-final-cleanup/39-CONTEXT.md` -- all 13 implementation decisions
- `.planning/phases/39-welcome-dashboard-and-final-cleanup/39-UI-SPEC.md` -- full visual and interaction contract
- `.planning/phases/37-onboarding-wizard-pages/37-CONTEXT.md` -- display name (D-17), onboarding guard (D-26-D-28)
- `.planning/phases/36-public-landing-page-and-route-restructure/36-CONTEXT.md` -- route tree structure (D-05-D-08)
- `.planning/phases/38-hard-paywall-and-soft-paywall-removal/38-CONTEXT.md` -- removal pattern (D-12-D-15)
- `.planning/codebase/STRUCTURE.md` -- frontend directory structure, old onboarding file locations
- `.planning/codebase/ARCHITECTURE.md` -- Redux store shape, OnboardingProgressUpdater usage
- `.planning/REQUIREMENTS.md` -- DASH-01, DASH-02, DASH-03 definitions

### Secondary (MEDIUM confidence)
- `.planning/quick/1-enforce-luxon-datetime-usage-in-dtos-for/1-PLAN.md` -- IOnboardingProgressDto shape (completedAt field)

### Tertiary (LOW confidence)
- Exact onboarding progress record fields for job/quote completion -- needs verification from actual API source code during implementation

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- all libraries already installed, no new dependencies
- Architecture: HIGH -- follows established feature-based module pattern with well-defined UI-SPEC
- Pitfalls: HIGH -- based on concrete analysis of data flow, state management, and removal patterns
- Onboarding progress data model: LOW -- the exact fields available for checklist completion need verification from API source code

**Research date:** 2026-04-02
**Valid until:** 2026-05-02 (stable -- no external dependency changes expected)
