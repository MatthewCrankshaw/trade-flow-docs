# Phase 49: Revision Frontend UI - Research

**Researched:** 2026-04-15
**Domain:** React RTK Query mutations, Radix Collapsible UI, estimate revision lifecycle frontend
**Confidence:** HIGH

## Summary

Phase 49 closes two gap-closure requirements (REV-02, REV-04) identified in the v1.8 milestone audit. The backend is fully shipped: `POST /v1/estimates/:id/revisions` (clone-only, no body) and `GET /v1/estimates/:id/revisions` (returns full `IEstimateDto[]` oldest-first) both exist in `EstimateController` since Phase 42. The frontend has zero revision awareness -- no RTK Query hooks for revision endpoints, no revision fields on the `Estimate` TypeScript type, no "Edit and resend" button, and no history section.

The work is entirely frontend (trade-flow-ui). It requires: (1) extending the `Estimate` type with revision fields already returned by the API, (2) adding two new RTK Query endpoints to `estimateApi.ts`, (3) adding an "Edit and resend" button to `EstimateActionStrip`, and (4) building a collapsible History section on `EstimateDetailPage`. The Radix `Collapsible` primitive is already installed and wrapped in `src/components/ui/collapsible.tsx`.

**Primary recommendation:** Add two RTK Query endpoints (revise mutation + revisions query), extend the Estimate type, wire the button and history UI into existing components.

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| REV-02 | User can revise a Sent estimate via "Edit and resend" which creates a new revision under the same E-YYYY-NNN number and marks previous as non-current | Backend endpoint `POST /v1/estimates/:id/revisions` exists (Phase 42). Frontend needs: RTK Query mutation, button on EstimateActionStrip for allowed statuses, navigation to detail page after revision created. |
| REV-04 | Estimate detail shows collapsed History section listing previous revisions with send/view timestamps (trader-only) | Backend endpoint `GET /v1/estimates/:id/revisions` exists (Phase 42, returns `IEstimateResponse[]` oldest-first). Frontend needs: RTK Query hook, collapsible History card on EstimateDetailPage showing revision number, sentAt, firstViewedAt for each revision. |
</phase_requirements>

## Standard Stack

### Core (already installed -- no new dependencies)

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| @reduxjs/toolkit (RTK Query) | 2.11.2 | API state & mutations | Project standard for all API calls [VERIFIED: estimateApi.ts] |
| @radix-ui/react-collapsible | installed | Collapsible primitive | Already wrapped in src/components/ui/collapsible.tsx [VERIFIED: codebase] |
| lucide-react | 0.563.0 | Icons (Edit, ChevronDown, History) | Project standard icon library [VERIFIED: CLAUDE.md] |
| react-router-dom | 7.13.0 | Navigation after revision created | Project standard routing [VERIFIED: CLAUDE.md] |

**Installation:** No new packages needed. All dependencies are already installed.

## Architecture Patterns

### Existing File Structure (estimates feature)

```
trade-flow-ui/src/
  features/estimates/
    api/estimateApi.ts          # RTK Query endpoints -- ADD 2 new endpoints here
    components/
      EstimateActionStrip.tsx   # Action buttons -- ADD "Edit and resend" button here
      index.ts                  # Barrel exports -- ADD new component export
    lib/                        # Feature utilities
  pages/
    EstimateDetailPage.tsx      # Detail page -- ADD History section here
  types/
    estimate.ts                 # Types -- ADD revision fields here
  components/ui/
    collapsible.tsx             # Radix Collapsible wrapper (already exists)
```

### Pattern 1: RTK Query Mutation (mirror existing estimateApi patterns)

**What:** Add `reviseEstimate` mutation and `getEstimateRevisions` query to the existing `estimateApi` slice.
**When to use:** Calling Phase 42's revision endpoints from the frontend.
**Example:**

```typescript
// Source: trade-flow-ui/src/features/estimates/api/estimateApi.ts [VERIFIED]
// Mirror the existing sendEstimate/convertEstimate/markEstimateLost mutation pattern

reviseEstimate: builder.mutation<Estimate, { estimateId: string }>({
  query: ({ estimateId }) => ({
    url: `/v1/estimates/${estimateId}/revisions`,
    method: "POST",
  }),
  transformResponse: unwrapSingle,
  invalidatesTags: (_result, _error, { estimateId }) => [
    { type: "Estimate", id: estimateId },
    { type: "Estimate", id: "LIST" },
  ],
}),

getEstimateRevisions: builder.query<Estimate[], string>({
  query: (estimateId) => `/v1/estimates/${estimateId}/revisions`,
  transformResponse: (response: StandardResponse<Estimate>) => response.data,
  providesTags: (_result, _error, estimateId) => [
    { type: "Estimate", id: `${estimateId}-revisions` },
  ],
}),
```

### Pattern 2: Action Button Visibility (mirror EstimateActionStrip)

**What:** Show "Edit and resend" only for statuses where revision is allowed.
**When to use:** EstimateActionStrip conditional rendering.
**Key detail from Phase 42 D-REV-02:** Allowed source statuses are `SENT, VIEWED, RESPONDED, SITE_VISIT_REQUESTED`. Phase 49 SC #3 says hidden for `Draft, Converted, Lost, Expired, Deleted`. The `site_visit_requested` status is still present in the frontend type (Phase 48 removes it but has not yet executed). Use the same `const REVISABLE_STATUSES = ["sent", "viewed", "responded", "site_visit_requested"] as const` pattern matching the existing `CONVERTIBLE_STATUSES` and `LOSABLE_STATUSES` arrays in EstimateActionStrip. [VERIFIED: EstimateActionStrip.tsx lines 13-14]

```typescript
// Source: EstimateActionStrip.tsx existing pattern [VERIFIED]
const REVISABLE_STATUSES = ["sent", "viewed", "responded", "site_visit_requested"] as const;
// ...
const canRevise = (REVISABLE_STATUSES as readonly string[]).includes(estimate.status);
```

### Pattern 3: Collapsible History Card (Radix Collapsible)

**What:** A collapsed-by-default Card showing revision history on the detail page.
**When to use:** EstimateDetailPage, below the action strip or above the response summary card.
**Example:**

```typescript
// Source: src/components/ui/collapsible.tsx [VERIFIED]
import { Collapsible, CollapsibleContent, CollapsibleTrigger } from "@/components/ui/collapsible";

// Renders inside EstimateDetailPage
<Card>
  <Collapsible>
    <CardHeader className="flex flex-row items-center justify-between">
      <CardTitle className="text-base">History</CardTitle>
      <CollapsibleTrigger asChild>
        <Button variant="ghost" size="sm">
          <ChevronsUpDown className="h-4 w-4" />
        </Button>
      </CollapsibleTrigger>
    </CardHeader>
    <CollapsibleContent>
      <CardContent>
        {/* revision list */}
      </CardContent>
    </CollapsibleContent>
  </Collapsible>
</Card>
```

### Pattern 4: Post-Mutation Navigation

**What:** After revise mutation succeeds, navigate to the newly created revision's detail page.
**Key insight:** The `reviseEstimate` mutation returns the new Estimate (the new Draft revision). Navigate using `navigate(\`/estimates/\${result.id}\`)` so the user lands on the new draft for editing. [VERIFIED: POST /v1/estimates/:id/revisions returns the new revision via `this.mapToResponse(revision)` in EstimateController]

### Anti-Patterns to Avoid

- **Do NOT add revision fields as optional with `?` when they always exist:** The API response always includes `parentEstimateId` (nullable for root), `rootEstimateId` (nullable for root), `revisionNumber` (always present, default 1), and `isCurrent` (always present, default true). Use `string | null` for nullable fields, not `string?`. [VERIFIED: estimate.responses.ts lines 69-72]
- **Do NOT create a separate RevisionHistory component file:** The history section is simple enough to inline in EstimateDetailPage or extract as a small component within the same file. Follow the existing pattern where EstimateDetailPage contains its own card sections inline.
- **Do NOT fetch revisions unconditionally:** Only fetch when `estimate.revisionNumber > 1` or when the user opens the collapsible (lazy fetch). Root estimates with `revisionNumber: 1` have no history to show.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Collapsible UI | Custom show/hide toggle | Radix Collapsible (already installed) | Handles animation, accessibility, state [VERIFIED: collapsible.tsx] |
| API state management | Manual fetch/loading/error state | RTK Query hooks | Project standard, handles caching and invalidation [VERIFIED: estimateApi.ts] |
| Date formatting | Custom date formatting | Luxon via `@/lib/date-helpers` | Project standard per CLAUDE.md [VERIFIED: CLAUDE.md Date/Time Standards] |

## Common Pitfalls

### Pitfall 1: Estimate type missing revision fields causes silent data loss

**What goes wrong:** The API already returns `parentEstimateId`, `rootEstimateId`, `revisionNumber`, `isCurrent` on every estimate response, but the frontend `Estimate` type does not include them. TypeScript silently drops unknown fields from typed objects in RTK Query transforms.
**Why it happens:** Phase 43 created the frontend types before Phase 42 shipped the revision fields.
**How to avoid:** Add revision fields to `Estimate` interface in `src/types/estimate.ts` BEFORE building any UI.
**Warning signs:** Estimate objects missing `revisionNumber` at runtime despite API returning them.

### Pitfall 2: Navigation after revise goes to wrong estimate

**What goes wrong:** After calling the revise mutation, navigating to the OLD estimate ID instead of the NEW revision's ID. The API auto-resolves non-current IDs to the current revision (D-DET-01), so the user would see the new draft anyway, but the URL would be stale.
**Why it happens:** Forgetting that `reviseEstimate` returns the NEW estimate, not the old one.
**How to avoid:** Use `const result = await reviseEstimate({ estimateId }).unwrap(); navigate(\`/estimates/\${result.id}\`);`

### Pitfall 3: History section shows on root estimates with no history

**What goes wrong:** Rendering an empty "History" card on estimates that have never been revised.
**Why it happens:** Not checking `revisionNumber > 1` before showing the section.
**How to avoid:** Conditionally render the History card only when `revisionNumber > 1`. Alternatively, always render but show "No previous revisions" when the list is empty.

### Pitfall 4: Cache invalidation after revision creation

**What goes wrong:** After creating a revision, the estimate list still shows the old estimate as current.
**Why it happens:** Not invalidating the right cache tags.
**How to avoid:** The `reviseEstimate` mutation must invalidate both the specific estimate tag AND the LIST tag. The backend changes `isCurrent` on the old row and creates a new row, so the list cache is stale. [VERIFIED: all existing mutations follow this pattern in estimateApi.ts]

### Pitfall 5: site_visit_requested status handling

**What goes wrong:** Phase 48 removes `site_visit_requested` from frontend types but has not yet executed. Phase 49 depends only on Phase 42 and 43.
**Why it happens:** Execution order -- Phase 48 and 49 are independent gap closures.
**How to avoid:** Include `site_visit_requested` in the `REVISABLE_STATUSES` array to match the backend's allowed statuses (D-REV-02). If Phase 48 executes first and removes it, the planner should note this dependency. If Phase 49 executes first, keep it.

## Code Examples

### Backend Endpoint Signatures (for RTK Query endpoint design)

```typescript
// Source: trade-flow-api/src/estimate/controllers/estimate.controller.ts [VERIFIED]

// Revise -- POST, no body, returns new revision
@Post("estimates/:id/revisions")
public async revise(
  @Req() request: { user: IUserDto; params: { id: string } }
): Promise<IResponse<IEstimateResponse>>

// History -- GET, returns IEstimateResponse[] oldest-first
@Get("estimates/:id/revisions")
public async findRevisions(
  @Req() request: { user: IUserDto; params: { id: string } }
): Promise<IResponse<IEstimateResponse>>
```

### API Response Shape (revision fields)

```typescript
// Source: trade-flow-api/src/estimate/responses/estimate.responses.ts [VERIFIED]
export interface IEstimateResponse {
  // ... existing fields ...
  parentEstimateId?: string | null;   // null for root estimate
  rootEstimateId?: string | null;     // null for root estimate
  revisionNumber: number;             // 1 for root, 2+ for revisions
  isCurrent: boolean;                 // true for active revision
}
```

### Existing EstimateActionStrip Pattern

```typescript
// Source: trade-flow-ui/src/features/estimates/components/EstimateActionStrip.tsx [VERIFIED]
const CONVERTIBLE_STATUSES = ["sent", "responded"] as const;
const LOSABLE_STATUSES = ["sent", "viewed", "responded"] as const;

// New revision status array follows same pattern:
const REVISABLE_STATUSES = ["sent", "viewed", "responded", "site_visit_requested"] as const;
```

### Date Helper Usage

```typescript
// Source: CLAUDE.md Date/Time Standards [VERIFIED]
import { formatDate, formatDateTime } from "@/lib/date-helpers";

// For revision history display:
// sentAt and firstViewedAt are ISO 8601 UTC strings from the API
formatDate(revision.sentAt)      // e.g., "15 Apr 2026"
formatDateTime(revision.sentAt)  // e.g., "15 Apr 2026, 10:30"
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Estimate type without revision fields | Must add revision fields to match API | Phase 42 (backend) | Frontend silently drops revision data |
| No revision RTK Query endpoints | Must add revise mutation + revisions query | Phase 42 (backend) | Cannot call revision API from UI |
| EstimateActionStrip: Send/Convert/Lost/Delete only | Must add "Edit and resend" button | This phase | Revision workflow inaccessible from UI |

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | The History section should be collapsed by default (Collapsible, not Accordion) | Architecture Patterns | Low -- UX preference, easily changed |
| A2 | History section placement: below action strip, near existing cards | Architecture Patterns | Low -- layout position is flexible |
| A3 | Lazy-fetch revisions only when `revisionNumber > 1` | Anti-Patterns | Low -- could always fetch and show empty state instead |

## Open Questions

1. **Should the revisions query be lazy-loaded on collapsible open or eager-loaded on page mount?**
   - What we know: RTK Query supports `skip` parameter for conditional fetching. Root estimates (revisionNumber 1) have no history.
   - What's unclear: Whether the user expects instant history display or is ok with a brief loading state on expand.
   - Recommendation: Skip fetch when `revisionNumber <= 1`. For revised estimates, eager-fetch on page mount since the data is small (few revisions per chain). This avoids a loading flash when expanding the collapsible.

2. **Should the "Edit and resend" button include a confirmation dialog?**
   - What we know: The backend POST is a destructive-ish action (marks old revision as non-current, creates new draft).
   - What's unclear: Whether a one-tap action is acceptable or if the trader should confirm.
   - Recommendation: No confirmation dialog. The action creates a Draft that the trader must explicitly send. The "send" step is the real commitment. This matches the existing "Convert to Quote" button which also has no confirmation.

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | Vitest 4.1.3 |
| Config file | trade-flow-ui/vite.config.ts (mergeConfig) |
| Quick run command | `cd trade-flow-ui && npm run test` |
| Full suite command | `cd trade-flow-ui && npm run ci` |

### Phase Requirements to Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| REV-02 | "Edit and resend" button renders for sent/viewed/responded statuses and is hidden for others | unit | `cd trade-flow-ui && npx vitest run src/features/estimates/components/__tests__/EstimateActionStrip.test.tsx -t "Edit and resend"` | Depends on existing test file |
| REV-04 | History section renders revision list with send/view dates when revisionNumber > 1 | unit | `cd trade-flow-ui && npx vitest run src/features/estimates/components/__tests__/EstimateRevisionHistory.test.tsx` | Wave 0 |

### Wave 0 Gaps

- [ ] `src/features/estimates/components/__tests__/EstimateRevisionHistory.test.tsx` -- covers REV-04 (new component)
- [ ] Verify existing `EstimateActionStrip` test file exists or create one for REV-02 coverage

## Security Domain

### Applicable ASVS Categories

| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | yes | Firebase JWT via existing prepareHeaders in apiSlice [VERIFIED] |
| V3 Session Management | no | N/A -- no new session logic |
| V4 Access Control | yes | Backend EstimatePolicy enforces ownership; frontend is not trust boundary [VERIFIED: Phase 42 D-CONC-02] |
| V5 Input Validation | no | POST /revisions takes no body; GET /revisions takes only route param |
| V6 Cryptography | no | N/A |

### Known Threat Patterns

| Pattern | STRIDE | Standard Mitigation |
|---------|--------|---------------------|
| IDOR on revision endpoint | Tampering | Backend EstimatePolicy checks ownership via JWT; frontend cannot bypass [VERIFIED] |
| Stale cache after revision | Information Disclosure | RTK Query tag invalidation ensures list + detail caches refresh [VERIFIED: existing pattern] |

## Sources

### Primary (HIGH confidence)
- trade-flow-api/src/estimate/controllers/estimate.controller.ts -- revision endpoint signatures verified
- trade-flow-api/src/estimate/responses/estimate.responses.ts -- IEstimateResponse revision fields verified
- trade-flow-ui/src/features/estimates/api/estimateApi.ts -- existing RTK Query patterns verified
- trade-flow-ui/src/features/estimates/components/EstimateActionStrip.tsx -- action button patterns verified
- trade-flow-ui/src/types/estimate.ts -- current type (missing revision fields) verified
- trade-flow-ui/src/components/ui/collapsible.tsx -- Radix Collapsible wrapper verified
- .planning/phases/42-revisions/42-CONTEXT.md -- D-REV-02 allowed statuses, D-HIST-01..05 history query
- .planning/v1.8-MILESTONE-AUDIT.md -- gap analysis confirming REV-02/REV-04 partial status

### Secondary (MEDIUM confidence)
- trade-flow-ui/CLAUDE.md -- date/time standards, conventions

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- all libraries already installed and patterns well-established
- Architecture: HIGH -- direct extension of existing components with verified API endpoints
- Pitfalls: HIGH -- based on verified codebase analysis showing exact gaps

**Research date:** 2026-04-15
**Valid until:** 2026-05-15 (stable -- no dependency changes expected)
