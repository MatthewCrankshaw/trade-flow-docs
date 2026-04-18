---
phase: 48-di-token-fix-cleanup
reviewed: 2026-04-16T00:00:00Z
depth: standard
files_reviewed: 1
files_reviewed_list:
  - trade-flow-ui/src/features/estimates/components/EstimateActionStrip.tsx
findings:
  critical: 0
  warning: 0
  info: 2
  total: 2
status: issues_found
---

# Phase 48: Code Review Report (Gap-Closure Re-review, Plan 48-03)

**Reviewed:** 2026-04-16T00:00:00Z
**Depth:** standard
**Files Reviewed:** 1
**Status:** issues_found

## Summary

This is a gap-closure re-review covering Plan 48-03, which tightens the status-gate constants in `EstimateActionStrip.tsx`. Only this file changed since the previous phase-48 review; the API module change, reviser spec, estimate type narrowing, and other estimate components were reviewed in the prior pass and are unchanged.

The change in scope:

- Three tuples (`CONVERTIBLE_STATUSES`, `LOSABLE_STATUSES`, `REVISABLE_STATUSES`) were converted from `[...] as const` plus an `as readonly string[]` widening cast to an explicit `readonly EstimateStatus[]` annotation.
- The stale `"site_visit_requested"` literal was removed from `LOSABLE_STATUSES`.

I verified the `EstimateStatus` union in `trade-flow-ui/src/types/estimate.ts`: it enumerates `draft | sent | viewed | responded | converted | declined | expired | lost | deleted`, and `site_visit_requested` is correctly absent. Every literal in the three typed tuples (`"sent"`, `"viewed"`, `"responded"`) is a valid member, so TypeScript will now catch typos or stale members at declaration time rather than silently widening through an `as` cast. The `.includes(estimate.status)` calls on lines 50, 51, and 53 continue to type-check: `readonly EstimateStatus[]` gives `includes` an `EstimateStatus` element type that matches the `estimate.status` argument exactly, so no widening is needed and no warning is emitted.

This closes the prior warnings WR-01 (dead `"site_visit_requested"` literal) and IN-01 (`as readonly string[]` widening) from the original 48-REVIEW.md. The prior WR-02 (`convertedToQuoteId!` non-null assertion) was explicitly out of scope for Plan 48-03 and remains in the file; it is not re-flagged here because the gap-closure scope is confined to the type-tightening change.

Two residual `info` items remain — both are the same anti-pattern the plan set out to remove, still present in sibling expressions that were not covered by 48-03's scope.

## Info

### IN-01: `canResend` still uses an inline untyped tuple

**File:** `trade-flow-ui/src/features/estimates/components/EstimateActionStrip.tsx:52`
**Issue:** Line 52 reads:

```ts
const canResend = ["sent", "viewed", "responded"].includes(estimate.status);
```

The inline array literal is inferred as `string[]` here, so `.includes` accepts any string and the compiler will not catch a typo or stale literal. The tuple is identical in contents to the new `REVISABLE_STATUSES` constant on line 15, and the very next line (`canRevise` on line 53) already uses the typed constant. This is the same weakness Plan 48-03 removed from the other three gates, left behind on this sibling line.

**Fix:** Reuse the existing typed constant (they are byte-identical today), or introduce a dedicated typed constant if the two concepts may diverge later:

```ts
const canResend = REVISABLE_STATUSES.includes(estimate.status);
```

Or, if a separate concept is preferred:

```ts
const RESENDABLE_STATUSES: readonly EstimateStatus[] = ["sent", "viewed", "responded"];
// ...
const canResend = RESENDABLE_STATUSES.includes(estimate.status);
```

### IN-02: `isTerminal` inline tuple bypasses union narrowing

**File:** `trade-flow-ui/src/features/estimates/components/EstimateActionStrip.tsx:49`
**Issue:** Line 49 reads:

```ts
const isTerminal = ["converted", "declined", "expired", "lost", "deleted"].includes(estimate.status);
```

Same anti-pattern as IN-01: the literal array is inferred as `string[]`, so a typo like `"converte d"` or a stale literal would not be caught. Strictly outside the scope of Plan 48-03 (which targeted the three gate tuples), but flagged for completeness since it is the exact class of issue the plan aimed to eliminate.

**Fix:** Promote to a typed constant alongside the others:

```ts
const TERMINAL_STATUSES: readonly EstimateStatus[] = ["converted", "declined", "expired", "lost", "deleted"];
// ...
const isTerminal = TERMINAL_STATUSES.includes(estimate.status);
```

---

_Reviewed: 2026-04-16T00:00:00Z_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: standard_
