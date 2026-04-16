---
phase: 48-di-token-fix-cleanup
reviewed: 2026-04-16T00:00:00Z
depth: standard
files_reviewed: 9
files_reviewed_list:
  - trade-flow-api/src/estimate/estimate.module.ts
  - trade-flow-api/src/estimate/services/estimate-to-quote-converter.service.ts
  - trade-flow-api/src/estimate/test/services/estimate-reviser.service.spec.ts
  - trade-flow-ui/src/features/estimates/components/EstimateActionStrip.tsx
  - trade-flow-ui/src/features/estimates/components/EstimatesCardList.tsx
  - trade-flow-ui/src/features/estimates/components/EstimatesTable.tsx
  - trade-flow-ui/src/pages/EstimateDetailPage.tsx
  - trade-flow-ui/src/pages/EstimatesPage.tsx
  - trade-flow-ui/src/types/estimate.ts
findings:
  critical: 0
  warning: 2
  info: 4
  total: 6
status: issues_found
---

# Phase 48: Code Review Report

**Reviewed:** 2026-04-16T00:00:00Z
**Depth:** standard
**Files Reviewed:** 9
**Status:** issues_found

## Summary

Phase 48 had two cleanup plans: 48-01 removed an API-side shadowed `NoopEstimateFollowupCanceller` provider from `EstimateModule` and fixed prettier drift in `estimate-to-quote-converter.service.ts`; 48-02 narrowed the UI `EstimateStatus` union by removing the dead `"site_visit_requested"` literal and purged references from pages/components.

The DI change in `EstimateModule` is clean and correct: the Noop shadow is gone, the real BullMQ canceller bound in `EstimateFollowupsModule` (via `useExisting`) is now the only provider visible to consumers, and the `NoopEstimateFollowupCanceller` class is retained as a test utility. The reviser spec wires the Noop via `{ provide: ESTIMATE_FOLLOWUP_CANCELLER, useClass: NoopEstimateFollowupCanceller }` at the test-module level, which is properly isolated from the production module's binding and therefore unaffected by the module cleanup.

However, the UI type-narrowing cleanup (48-02) is **incomplete**. One live source-code reference to `"site_visit_requested"` survives in `EstimateActionStrip.tsx` inside the `REVISABLE_STATUSES` tuple. The 48-02 plan and summary both claim zero remaining references, but a grep of `trade-flow-ui/src/` at the time of this review returns one match. This is dead data rather than a runtime bug (no estimate can ever carry that status after the backend enum removal in Phase 45), but it contradicts the phase's success criteria and hides behind a deliberate widening cast.

Additional quality observations include an `as readonly string[]` widening pattern that silently disables literal-union narrowing for all four status-gating checks in `EstimateActionStrip`, a non-null assertion on `convertedToQuoteId`, and duplication of the `statusColors` table across three files.

## Warnings

### WR-01: Dead `"site_visit_requested"` literal survives in `REVISABLE_STATUSES`

**File:** `trade-flow-ui/src/features/estimates/components/EstimateActionStrip.tsx:15`
**Issue:** After 48-02 narrowed `EstimateStatus` to remove `"site_visit_requested"`, this tuple still contains it:

```ts
const REVISABLE_STATUSES = ["sent", "viewed", "responded", "site_visit_requested"] as const;
```

The phase summary (`48-02-SUMMARY.md`) and plan (`48-02-PLAN.md`) both claim grep returns zero matches -- this is wrong. The literal survives because the tuple is later widened via `(REVISABLE_STATUSES as readonly string[]).includes(estimate.status)` (see line 53), which defeats any type check that would have caught the stale member. This is a latent correctness defect: a future reader may infer that `"site_visit_requested"` is a legitimate status and propagate it, and the file fails the phase's stated success criterion of zero residual references.

**Fix:** Remove the literal from the tuple and drop the widening cast so the compiler enforces exhaustiveness against `EstimateStatus`:

```ts
const REVISABLE_STATUSES: readonly EstimateStatus[] = ["sent", "viewed", "responded"] as const;
// ...
const canRevise = REVISABLE_STATUSES.includes(estimate.status);
```

This also eliminates the `as readonly string[]` widening called out in IN-01.

### WR-02: `convertedToQuoteId!` non-null assertion lacks guarantee

**File:** `trade-flow-ui/src/features/estimates/components/EstimateActionStrip.tsx:80`
**Issue:** When `isConverted` is true the component renders `<EstimateConvertedLink convertedToQuoteId={estimate.convertedToQuoteId!} />`. The `Estimate` type declares `convertedToQuoteId?: string` (optional). The runtime invariant "status === 'converted' implies `convertedToQuoteId` is present" is an API-level contract, not a type-level one, so the `!` silently hides any backend regression or in-flight state where the status has transitioned but the id has not yet been written (this exact race is a known hazard in the converter: `EstimateToQuoteConverter.convert` transitions status to CONVERTED at line 65 before writing `convertedToQuoteId` at line 73, and if a reader races between those writes the id could be missing while the status is already `converted`). User memory also flags avoiding `as`/non-null assertions in favour of type guards.

**Fix:** Guard explicitly and render a safe fallback, so the contract is enforced locally:

```tsx
if (isConverted) {
  if (!estimate.convertedToQuoteId) {
    return null; // or a muted "Conversion pending" badge
  }
  return <EstimateConvertedLink convertedToQuoteId={estimate.convertedToQuoteId} />;
}
```

## Info

### IN-01: `as readonly string[]` widening defeats exhaustiveness checks

**File:** `trade-flow-ui/src/features/estimates/components/EstimateActionStrip.tsx:50-53`
**Issue:** All four status-gating checks cast tuples to `readonly string[]` purely to satisfy the `.includes()` signature:

```ts
const canConvert = (CONVERTIBLE_STATUSES as readonly string[]).includes(estimate.status);
const canMarkLost = (LOSABLE_STATUSES as readonly string[]).includes(estimate.status);
const canResend = ["sent", "viewed", "responded"].includes(estimate.status);
const canRevise = (REVISABLE_STATUSES as readonly string[]).includes(estimate.status);
```

This pattern directly contradicts the project rule in user memory ("Avoid `as` type assertions -- prefer type guards/validation over `as` casts") and in `trade-flow-api/CLAUDE.md` ("Avoid `as` type assertions -- prefer type guards, mapping functions, or narrowing"). It also directly caused WR-01 to go undetected: a literal outside `EstimateStatus` sits in a tuple and no compiler check fires.

**Fix:** Type the tuples as `readonly EstimateStatus[]` at declaration and drop the casts; TS then narrows `.includes` correctly against `estimate.status`:

```ts
const CONVERTIBLE_STATUSES: readonly EstimateStatus[] = ["sent", "responded"];
const LOSABLE_STATUSES: readonly EstimateStatus[] = ["sent", "viewed", "responded"];
const RESENDABLE_STATUSES: readonly EstimateStatus[] = ["sent", "viewed", "responded"];
const REVISABLE_STATUSES: readonly EstimateStatus[] = ["sent", "viewed", "responded"];

const canConvert = CONVERTIBLE_STATUSES.includes(estimate.status);
const canMarkLost = LOSABLE_STATUSES.includes(estimate.status);
const canResend = RESENDABLE_STATUSES.includes(estimate.status);
const canRevise = REVISABLE_STATUSES.includes(estimate.status);
```

### IN-02: `statusColors` map duplicated across three files

**Files:**

- `trade-flow-ui/src/features/estimates/components/EstimatesCardList.tsx:19-29`
- `trade-flow-ui/src/features/estimates/components/EstimatesTable.tsx:19-29`
- `trade-flow-ui/src/pages/EstimateDetailPage.tsx:44-54`

**Issue:** The identical `Record<EstimateStatus, ...>` status-colour map is defined in three places. Because the record type is keyed on `EstimateStatus`, removing a status from the union (as 48-02 did) does force each copy to be updated -- which is a small safety net -- but the duplication remains a maintenance burden and is exactly the class of repetition the user-memory rule "Separation over DRY" *does not* exempt (the rule applies to entity-boundary document types, not presentational constants). A future status addition requires three coordinated edits, and a reviewer must check all three locations for consistency.

**Fix:** Extract a shared constant into the feature module:

```ts
// trade-flow-ui/src/features/estimates/constants/status-colors.ts
import type { EstimateStatus } from "@/types";

export const ESTIMATE_STATUS_COLORS: Record<EstimateStatus, "default" | "secondary" | "destructive" | "outline"> = {
  draft: "secondary",
  sent: "outline",
  viewed: "outline",
  responded: "default",
  converted: "default",
  declined: "destructive",
  expired: "secondary",
  lost: "destructive",
  deleted: "destructive",
};
```

Import in all three consumers.

### IN-03: Inline `isTerminal` array not typed to `EstimateStatus`

**File:** `trade-flow-ui/src/features/estimates/components/EstimateActionStrip.tsx:49`
**Issue:**

```ts
const isTerminal = ["converted", "declined", "expired", "lost", "deleted"].includes(estimate.status);
```

This inline literal array bypasses the same kind of union-narrowing and would not flag a typo such as `"converted "` or a stale value. Same root cause as IN-01.

**Fix:** Hoist as a typed constant alongside the others:

```ts
const TERMINAL_STATUSES: readonly EstimateStatus[] = ["converted", "declined", "expired", "lost", "deleted"];
const isTerminal = TERMINAL_STATUSES.includes(estimate.status);
```

### IN-04: Residual comment referencing a prior D-HOOK identifier

**File:** `trade-flow-api/src/estimate/test/services/estimate-reviser.service.spec.ts:121-122`
**Issue:**

```ts
// D-HOOK-03: followupCanceller is injected but not called
describe("D-HOOK-03: followupCanceller is injected but not called", () => {
```

The line 121 comment duplicates the `describe` block's string description verbatim -- per the project conventions this is a classic "comment restates what the code does" case and violates both the API and UI CLAUDE.md rules ("Never write comments that restate what the code does"). The design marker inside the describe string is sufficient as behavioural documentation.

**Fix:** Delete line 121.

---

_Reviewed: 2026-04-16T00:00:00Z_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: standard_
