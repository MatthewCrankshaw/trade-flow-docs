---
phase: 49-revision-frontend-ui
reviewed: 2026-04-16T00:00:00Z
depth: standard
files_reviewed: 8
files_reviewed_list:
  - trade-flow-ui/src/features/estimates/components/EstimateRevisionHistory.tsx
  - trade-flow-ui/src/features/estimates/api/__tests__/estimateApi.revisions.test.ts
  - trade-flow-ui/src/types/estimate.ts
  - trade-flow-ui/src/features/estimates/api/estimateApi.ts
  - trade-flow-ui/src/features/estimates/components/EstimateActionStrip.tsx
  - trade-flow-ui/src/features/estimates/components/index.ts
  - trade-flow-ui/src/features/estimates/components/__tests__/SendEstimateForm.test.tsx
  - trade-flow-ui/src/pages/EstimateDetailPage.tsx
findings:
  critical: 0
  warning: 4
  info: 5
  total: 9
status: issues_found
---

# Phase 49: Code Review Report

**Reviewed:** 2026-04-16
**Depth:** standard
**Files Reviewed:** 8
**Status:** issues_found

## Summary

Phase 49 adds revision history UI for estimates: a new `EstimateRevisionHistory` collapsible card, an "Edit and resend" action that creates a new draft revision, and supporting RTK Query endpoints (`reviseEstimate`, `getEstimateRevisions`). Additions to the `Estimate` type expose `parentEstimateId`, `rootEstimateId`, `revisionNumber`, and `isCurrent`.

Overall the implementation follows project conventions (feature-module organisation, RTK Query patterns, `@/` alias imports, `import type`, Luxon-based date formatting via `date-helpers`). No critical security or correctness issues were found.

The warnings centre on a status-guard inconsistency in `EstimateActionStrip` (the new `REVISABLE_STATUSES` list includes `"site_visit_requested"`, which is not a member of the `EstimateStatus` type), a pair of test assertions that appear to contradict the behaviour they describe, a non-null assertion in `EstimateDetailPage.tsx`, and `as unknown as ...` type-assertion casts in the revisions test that the project's feedback memory explicitly discourages. Info items cover naming, dead assertions, and minor readability tweaks.

## Warnings

### WR-01: `REVISABLE_STATUSES` contains a value outside the `EstimateStatus` union

**File:** `trade-flow-ui/src/features/estimates/components/EstimateActionStrip.tsx:15`
**Issue:** `REVISABLE_STATUSES` is declared as
```ts
const REVISABLE_STATUSES = ["sent", "viewed", "responded", "site_visit_requested"] as const;
```
`"site_visit_requested"` is not a member of the `EstimateStatus` union declared in `trade-flow-ui/src/types/estimate.ts:3-12` (`"draft" | "sent" | "viewed" | "responded" | "converted" | "declined" | "expired" | "lost" | "deleted"`). Because `canRevise` performs the check via `(REVISABLE_STATUSES as readonly string[]).includes(estimate.status)`, the `as readonly string[]` cast silently loses the type-narrowing safety that the other `*_STATUSES` constants enjoy, and dead code that will never match is retained.

This is load-bearing for the "Edit and resend" action: if a `"site_visit_requested"` status is genuinely expected (e.g. from a sibling phase), it is missing from the shared `EstimateStatus` union and every downstream consumer (`statusColors` map in `EstimateDetailPage.tsx:44-54`, `CONVERTIBLE_STATUSES`, `LOSABLE_STATUSES`) will silently treat it as unhandled. If it is not expected, the string is dead.

**Fix:** Decide which case applies and update both sides together. If `site_visit_requested` is in-scope, add it to `EstimateStatus` and extend `statusColors`:
```ts
// src/types/estimate.ts
export type EstimateStatus =
  | "draft"
  | "sent"
  | "viewed"
  | "responded"
  | "site_visit_requested"
  | "converted"
  | "declined"
  | "expired"
  | "lost"
  | "deleted";
```
…and in `EstimateDetailPage.tsx:44`, add a colour mapping. Otherwise, drop the string from `REVISABLE_STATUSES`. Either way, type the constant against `EstimateStatus` so the compiler catches future drift:
```ts
const REVISABLE_STATUSES: readonly EstimateStatus[] = ["sent", "viewed", "responded"] as const;
const canRevise = REVISABLE_STATUSES.includes(estimate.status);
```
This removes the `as readonly string[]` cast (consistent with project guidance to avoid `as`).

---

### WR-02: Test case description and assertion do not match the described behaviour

**File:** `trade-flow-ui/src/features/estimates/components/__tests__/SendEstimateForm.test.tsx:103-110`
**Issue:** The test is titled `"'Save email to customer record' checkbox is checked by default when customerEmail is null"` but the body never asserts on the checkbox at all; it only asserts `toInput` is empty. The preceding comment ("Type an email so the checkbox becomes visible") implies an interaction that the test never performs. This is a silent-passing test that provides no coverage of the described behaviour.

The sibling test at `:146-151` has the inverse problem: it is titled `"'Save email to customer record' checkbox is checked by default when customer has an email on file"` but asserts `expect(checkbox).not.toBeChecked()` — the assertion contradicts the title.

**Fix:** Align each test's body with its title. For the null-email case, type an email and assert the checkbox is present and checked:
```tsx
it("'Save email to customer record' checkbox is checked by default when customerEmail is null", async () => {
  const user = userEvent.setup();
  render(<SendEstimateForm {...defaultProps} customerEmail={null} />);

  const toInput = screen.getByLabelText(/^to$/i);
  await user.type(toInput, "jane@example.com");

  const checkbox = await screen.findByRole("checkbox", { name: /save email to customer record/i });
  expect(checkbox).toBeChecked();
});
```
For the on-file case, either change the title to reflect the actual assertion ("…is unchecked by default when customer has an email on file") or change the assertion to match the intended behaviour.

---

### WR-03: Non-null assertion on `estimateId` route param

**File:** `trade-flow-ui/src/pages/EstimateDetailPage.tsx:321`
**Issue:**
```ts
const { data: estimate, isLoading, isError } = useGetEstimateQuery(estimateId!, { skip: !estimateId });
```
The non-null assertion (`estimateId!`) bypasses strict null checks. While the `skip: !estimateId` option prevents the query from firing with an empty arg, the asserted value is still passed into RTK Query's cache key. If the router ever mounts this component at a malformed path, the cache key becomes `"undefined"` and misbehaves silently. Project memory records a preference against `as` assertions and non-null assertions follow the same principle.

**Fix:** Render the not-found branch when `estimateId` is missing, and pass a narrowed value to the query:
```ts
const { estimateId } = useParams<{ estimateId: string }>();
const { data: estimate, isLoading, isError } = useGetEstimateQuery(estimateId ?? "", { skip: !estimateId });
```
Or, preferred (avoids a fallback string that ends up in the cache):
```ts
if (!estimateId) {
  return <EstimateNotFound />;
}
const { data: estimate, isLoading, isError } = useGetEstimateQuery(estimateId);
```
Applies equally to `businessId!` at `:323`.

Also see `EstimateActionStrip.tsx:80`:
```tsx
return <EstimateConvertedLink convertedToQuoteId={estimate.convertedToQuoteId!} />;
```
Same pattern. Narrow explicitly and render a fallback when the value is absent:
```tsx
if (estimate.convertedToQuoteId) {
  return <EstimateConvertedLink convertedToQuoteId={estimate.convertedToQuoteId} />;
}
```

---

### WR-04: `as unknown as` double cast in revisions test

**File:** `trade-flow-ui/src/features/estimates/api/__tests__/estimateApi.revisions.test.ts:7-18`
**Issue:**
```ts
expect(typeof (estimateApi as unknown as { useReviseEstimateMutation?: unknown }).useReviseEstimateMutation).toBe(
  "function",
);
```
`as unknown as {...}` is a double cast that bypasses type checking entirely. The project memory explicitly calls out "Avoid `as` type assertions" (`feedback_no_type_assertions.md`). It also obscures what is being verified — the hooks are already exported named constants destructured at the bottom of `estimateApi.ts`, so a direct import would be both type-safe and more meaningful.

**Fix:** Import the hooks directly and assert on their existence without casts:
```ts
import { useReviseEstimateMutation, useGetEstimateRevisionsQuery } from "../estimateApi";

it("exposes useReviseEstimateMutation hook", () => {
  expect(typeof useReviseEstimateMutation).toBe("function");
});

it("exposes useGetEstimateRevisionsQuery hook", () => {
  expect(typeof useGetEstimateRevisionsQuery).toBe("function");
});
```
The remaining endpoint assertions (`estimateApi.endpoints.reviseEstimate`) are already type-safe and need no change.

## Info

### IN-01: Collapsible card has no controlled open state, so `isLoading` spinner is never visible

**File:** `trade-flow-ui/src/features/estimates/components/EstimateRevisionHistory.tsx:19`
**Issue:** `<Collapsible>` defaults to `open={false}`. Because the loading skeleton, error message, and revision list are all inside `<CollapsibleContent>`, the user must click the toggle before any of them render. During that first expand, the `useGetEstimateRevisionsQuery` hook is already fetching (the hook fires on mount regardless of collapsed state), so the skeleton at `:33-37` is unlikely to ever be shown. Consider either defaulting the section to open when `revisionNumber > 1` (the condition under which the card is rendered at `EstimateDetailPage.tsx:258`), or gating the query on `open` state so the network request only fires when the user expands the panel.
**Fix:** Default to open, since the card is only rendered when revisions exist:
```tsx
<Collapsible defaultOpen>
```

### IN-02: `orderedRevisions` name is ambiguous

**File:** `trade-flow-ui/src/features/estimates/components/EstimateRevisionHistory.tsx:15`
**Issue:** `const orderedRevisions = revisions ? [...revisions].reverse() : [];` — the name says "ordered" without indicating the direction. A reader has to infer from `.reverse()` that it is newest-first.
**Fix:** Rename to `revisionsNewestFirst` (or similar) for self-documenting intent:
```ts
const revisionsNewestFirst = revisions ? [...revisions].reverse() : [];
```

### IN-03: Duplicate inline status list for "resend"

**File:** `trade-flow-ui/src/features/estimates/components/EstimateActionStrip.tsx:52`
**Issue:**
```ts
const canResend = ["sent", "viewed", "responded"].includes(estimate.status);
```
This literal array is identical to `LOSABLE_STATUSES` at `:14` and overlaps with `REVISABLE_STATUSES`. Extracting a named constant keeps the declarative `*_STATUSES` style used elsewhere in the file and ensures future status additions are caught in one place.
**Fix:**
```ts
const RESENDABLE_STATUSES: readonly EstimateStatus[] = ["sent", "viewed", "responded"] as const;
const canResend = RESENDABLE_STATUSES.includes(estimate.status);
```

### IN-04: Revision "sentAt && viewedAt" guard is redundant

**File:** `trade-flow-ui/src/features/estimates/components/EstimateRevisionHistory.tsx:56`
**Issue:** `{revision.sentAt && revision.viewedAt && (...)}` — a revision logically cannot be `viewedAt` without being `sentAt` first. Checking `revision.viewedAt` alone is sufficient and reads more directly.
**Fix:**
```tsx
{revision.viewedAt && (
  <p className="text-xs text-muted-foreground">Viewed {formatDateTime(revision.viewedAt)}</p>
)}
```

### IN-05: Revisions cache tag scheme may cause stale list on create

**File:** `trade-flow-ui/src/features/estimates/api/estimateApi.ts:189-205`
**Issue:** `reviseEstimate` invalidates `{ type: "Estimate", id: estimateId }` and `{ type: "Estimate", id: "LIST" }`, but `getEstimateRevisions` provides `{ type: "Estimate", id: "${estimateId}-revisions" }`. A newly-created revision therefore will not invalidate the revisions list of either the source estimate or the new one, so the `EstimateRevisionHistory` card may show stale data after a user clicks "Edit and resend" and then navigates back to the source estimate.

Note: in the current UX (`EstimateActionStrip.tsx:69-77`) the user is navigated to `/estimates/${result.id}`, which is a new estimate so the history card at `EstimateDetailPage.tsx:258` renders for a fresh cache key — in practice the bug is latent rather than active. Worth fixing before new navigation flows land.
**Fix:** Have `reviseEstimate` also invalidate the revisions-list tag for both the source and the new estimate:
```ts
invalidatesTags: (result, _error, { estimateId }) => [
  { type: "Estimate", id: estimateId },
  { type: "Estimate", id: "LIST" },
  { type: "Estimate", id: `${estimateId}-revisions` },
  ...(result ? [{ type: "Estimate" as const, id: `${result.id}-revisions` }] : []),
],
```

---

_Reviewed: 2026-04-16_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: standard_
