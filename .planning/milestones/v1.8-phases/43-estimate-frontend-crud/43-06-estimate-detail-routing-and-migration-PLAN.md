---
phase: 43-estimate-frontend-crud
plan: 06
type: execute
wave: 4
depends_on:
  - 43-01
  - 43-02
  - 43-03
  - 43-04
  - 43-05
files_modified:
  - trade-flow-ui/src/pages/EstimateDetailPage.tsx
  - trade-flow-ui/src/App.tsx
  - trade-flow-ui/src/config/navigation.ts
  - trade-flow-ui/src/pages/JobDetailPage.tsx
  - trade-flow-ui/src/pages/QuotesPage.tsx
  - trade-flow-ui/src/features/quotes/components/CreateQuoteDialog.tsx
  - trade-flow-ui/src/features/quotes/components/index.ts
autonomous: true
requirements:
  - CONT-03
  - CONT-04

must_haves:
  truths:
    - "EstimateDetailPage renders at /estimates/:estimateId and displays line items, contingency percent, formatted price range, status, customer info, response summary placeholder, and Delete action for Draft"
    - "Draft detail page inline-edits contingencyPercent, displayMode, uncertaintyReasons, uncertaintyNotes, notes via PATCH"
    - "Non-Draft detail page renders all controls as disabled / read-only"
    - "/estimates and /estimates/:estimateId routes exist in App.tsx under the same guards as /quotes"
    - "Estimates sidebar link appears in navigation.ts between Quotes and Items"
    - "JobDetailPage 'Create Quote' button renamed to 'Create Document' and routes through CreateDocumentDialog (no defaultType)"
    - "QuotesPage now uses CreateDocumentDialog defaultType=\"quote\""
    - "CreateQuoteDialog.tsx file is DELETED and removed from the feature barrel"
    - "CI gate (cd trade-flow-ui && npm run ci) passes"
  artifacts:
    - path: "trade-flow-ui/src/pages/EstimateDetailPage.tsx"
      provides: "Detail page with inline-editable Draft controls and delete"
      contains: "useGetEstimateQuery"
    - path: "trade-flow-ui/src/App.tsx"
      provides: "Routing for /estimates and /estimates/:estimateId"
      contains: "EstimatesPage"
    - path: "trade-flow-ui/src/config/navigation.ts"
      provides: "Estimates sidebar link"
      contains: "Estimates"
  key_links:
    - from: "trade-flow-ui/src/App.tsx"
      to: "EstimatesPage + EstimateDetailPage"
      via: "Route elements under PaywallGuard"
      pattern: "path=\"/estimates"
    - from: "trade-flow-ui/src/pages/JobDetailPage.tsx"
      to: "CreateDocumentDialog"
      via: "replaces CreateQuoteDialog usage"
      pattern: "CreateDocumentDialog"
    - from: "trade-flow-ui/src/pages/QuotesPage.tsx"
      to: "CreateDocumentDialog"
      via: "defaultType=\"quote\""
      pattern: "defaultType=\"quote\""
---

<objective>
Ship the EstimateDetailPage, wire routing and navigation, migrate all existing CreateQuoteDialog consumers to CreateDocumentDialog, delete CreateQuoteDialog.tsx, and run the full trade-flow-ui CI gate.

Purpose: This is the final integration plan that makes Phase 43 end-to-end operational. The user can now click the sidebar, see the estimates list, open a detail page, create an estimate from QuotesPage / EstimatesPage / JobDetailPage, inline-edit a Draft, and delete it — all against Phase 41's real backend.

Output: One new detail page, three file edits wiring routes/nav/JobDetailPage, two file edits migrating QuotesPage + deleting CreateQuoteDialog, barrel cleanup, passing CI gate.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/phases/43-estimate-frontend-crud/43-CONTEXT.md
@trade-flow-ui/src/pages/QuoteDetailPage.tsx
@trade-flow-ui/src/pages/QuotesPage.tsx
@trade-flow-ui/src/pages/JobDetailPage.tsx
@trade-flow-ui/src/App.tsx
@trade-flow-ui/src/config/navigation.ts
@trade-flow-ui/src/features/quotes/components/CreateQuoteDialog.tsx
@trade-flow-ui/src/features/quotes/components/index.ts
@trade-flow-ui/src/components/CreateDocumentDialog.tsx
@trade-flow-ui/src/features/estimates/api/estimateApi.ts
@trade-flow-ui/src/features/estimates/components/ContingencySlider.tsx
@trade-flow-ui/src/features/estimates/components/UncertaintyChipGroup.tsx
@trade-flow-ui/src/features/estimates/components/EstimateLineItemsCard.tsx
@trade-flow-ui/src/features/estimates/components/EstimateActionStrip.tsx
@trade-flow-ui/src/lib/currency.ts

<interfaces>
<!-- Locked decisions specific to detail page + migration -->

Phase 43 D-EDIT-01 — draft detail inline-edit fields (via PATCH, pessimistic with spinner default):
- contingencyPercent (ContingencySlider onValueChange → useUpdateEstimateMutation)
- displayMode (pill toggle → PATCH)
- uncertaintyReasons (UncertaintyChipGroup onValueChange → PATCH)
- uncertaintyNotes (Textarea onBlur → PATCH)
- notes (Textarea onBlur → PATCH)

Phase 43 D-EDIT-03 — non-Draft: every control disabled, delete button hidden

Phase 43 D-EDIT-04 — delete UX: AlertDialog with "Delete E-YYYY-NNN? This cannot be undone." + confirm triggers useDeleteEstimateMutation + navigate back to /estimates

Phase 43 D-RESP-01 — response summary is always null in Phase 43; render as a "No response yet" placeholder card

Phase 43 D-ROUTE-01/02 — routes under AuthenticatedLayout > OnboardingGuard > PaywallGuard, sidebar link between Quotes and Items using `ClipboardList` icon from lucide-react (planner's discretion — D-ROUTE-02 allows FileText / Receipt / ClipboardList / ClipboardPen; chose ClipboardList as visually distinct from Quote's FileText)

Phase 43 D-ROUTE-03 — JobDetailPage "Create Quote" button renamed to "Create Document" and routes via CreateDocumentDialog with prefilledJobId/prefilledJobTitle and NO defaultType (toggle visible)

From trade-flow-ui/src/pages/QuoteDetailPage.tsx — structural template for detail page (199 lines). Key sections: back link, header (number + status badge + customer + job link + date), action strip, totals card, line items card, notes card, loading skeleton.

From trade-flow-ui/src/pages/QuotesPage.tsx — current create dialog usage to migrate:
```typescript
import { ..., CreateQuoteDialog } from "@/features/quotes";
// ...
{businessId && <CreateQuoteDialog open={createOpen} onOpenChange={setCreateOpen} businessId={businessId} />}
```
becomes:
```typescript
import { CreateDocumentDialog } from "@/components/CreateDocumentDialog";
// ...
{businessId && <CreateDocumentDialog open={createOpen} onOpenChange={setCreateOpen} businessId={businessId} defaultType="quote" />}
```

From trade-flow-ui/src/pages/JobDetailPage.tsx (line 23, line 214):
```typescript
import { useGetQuotesQuery, CreateQuoteDialog } from "@/features/quotes";
// ...
<CreateQuoteDialog open={createQuoteOpen} onOpenChange={setCreateQuoteOpen} businessId={businessId} prefilledJobId={jobId} prefilledJobTitle={job?.title} />
```
Button text `"Create Quote"` renames to `"Create Document"`; dialog becomes `<CreateDocumentDialog ... prefilledJobId={jobId} prefilledJobTitle={job?.title} />` with NO defaultType.
</interfaces>
</context>

<tasks>

<task type="auto" tdd="false">
  <name>Task 1: Build EstimateDetailPage with inline-edit Draft controls + delete</name>
  <files>trade-flow-ui/src/pages/EstimateDetailPage.tsx</files>
  <read_first>
    - trade-flow-ui/src/pages/QuoteDetailPage.tsx (full file — structural template)
    - trade-flow-ui/src/features/estimates/api/estimateApi.ts (for useGetEstimateQuery, useUpdateEstimateMutation, useDeleteEstimateMutation)
    - trade-flow-ui/src/features/estimates/components/ContingencySlider.tsx
    - trade-flow-ui/src/features/estimates/components/UncertaintyChipGroup.tsx
    - trade-flow-ui/src/features/estimates/components/EstimateLineItemsCard.tsx
    - trade-flow-ui/src/features/estimates/components/EstimateActionStrip.tsx
    - trade-flow-ui/src/lib/currency.ts (for formatRange)
    - trade-flow-ui/src/components/ui/alert-dialog.tsx
    - .planning/phases/43-estimate-frontend-crud/43-CONTEXT.md D-EDIT-01 through D-EDIT-06, D-MOD-02/03, D-SLD-04, D-BADGE-01/02, D-RESP-01
  </read_first>
  <behavior>
    - Draft status: every field inline-editable, each PATCH pessimistic with a small Loader2 spinner during the request
    - Non-Draft status: every field rendered read-only (Slider disabled, chip buttons disabled, textareas disabled, no line-item picker, no delete button)
    - Header shows: estimate.number, status badge using the local statusColors map, customer name, job link, estimate date
    - Main body renders in order: price range card (formatRange + displayMode toggle + contingency slider), uncertainty card (chips + notes textarea), line items card (EstimateLineItemsCard), internal notes card (Textarea), response summary card (placeholder "No response yet"), action strip (delete only in Phase 43)
    - Delete flow: AlertDialog with "Delete {estimate.number}? This cannot be undone." → confirm → useDeleteEstimateMutation → toast + navigate("/estimates")
  </behavior>
  <action>
Create NEW file `trade-flow-ui/src/pages/EstimateDetailPage.tsx`. Structure based on QuoteDetailPage.tsx. Concrete requirements:

1. Route params via `useParams<{ estimateId: string }>()`; fetch via `useGetEstimateQuery(estimateId!, { skip: !estimateId })`
2. Define the exact `statusColors` map inline (from the interfaces block of plan 05 — 10 entries for EstimateStatus)
3. Derived state:
   ```typescript
   const isDraft = estimate?.status === "draft";
   const isEditable = isDraft; // narrower than quotes per D-EDIT-02
   ```
4. Local UI state:
   ```typescript
   const [notesDraft, setNotesDraft] = useState(estimate?.notes ?? "");
   const [uncertaintyNotesDraft, setUncertaintyNotesDraft] = useState(estimate?.uncertaintyNotes ?? "");
   const [estimateToDelete, setEstimateToDelete] = useState<Estimate | null>(null);
   const [updateEstimate, { isLoading: isUpdating }] = useUpdateEstimateMutation();
   const [deleteEstimate] = useDeleteEstimateMutation();
   ```
   Sync `notesDraft`/`uncertaintyNotesDraft` from the fetched estimate via `useEffect` so the draft mirrors server state on mount and after invalidation.

5. PATCH handlers (pessimistic, single-field patches):
   ```typescript
   const handleContingencyChange = async (percent: number) => {
     if (!businessId || !estimate) return;
     try {
       await updateEstimate({ businessId, estimateId: estimate.id, data: { contingencyPercent: percent } }).unwrap();
     } catch {
       toast.error("Failed to update contingency");
     }
   };
   const handleDisplayModeChange = async (mode: EstimateDisplayMode) => { /* same pattern */ };
   const handleUncertaintyReasonsChange = async (reasons: UncertaintyReason[]) => { /* same pattern */ };
   const handleUncertaintyNotesBlur = async () => { /* only patches if draft value differs from server */ };
   const handleNotesBlur = async () => { /* same */ };
   ```

6. Price range card:
   ```jsx
   <Card>
     <CardHeader><CardTitle>Price</CardTitle></CardHeader>
     <CardContent className="space-y-4">
       <div className="text-2xl font-bold">
         {formatRange(estimate.priceRange.low, estimate.priceRange.high, estimate.displayMode, currencyCode)}
       </div>
       {/* display-mode toggle: two pill Buttons */}
       <div className="flex gap-2">
         <Button
           type="button"
           size="sm"
           variant={estimate.displayMode === "range" ? "default" : "outline"}
           onClick={() => handleDisplayModeChange("range")}
           disabled={!isEditable || isUpdating}
           aria-pressed={estimate.displayMode === "range"}
         >
           Range (£X - £Y)
         </Button>
         <Button
           type="button"
           size="sm"
           variant={estimate.displayMode === "from" ? "default" : "outline"}
           onClick={() => handleDisplayModeChange("from")}
           disabled={!isEditable || isUpdating}
           aria-pressed={estimate.displayMode === "from"}
         >
           From £X
         </Button>
       </div>
       <ContingencySlider
         value={estimate.contingencyPercent}
         onValueChange={handleContingencyChange}
         disabled={!isEditable || isUpdating}
       />
       <p className="text-sm text-muted-foreground">
         Base total: {currency.formatAmount(estimate.totals.total)} · API-computed range applies {estimate.contingencyPercent}% buffer.
       </p>
     </CardContent>
   </Card>
   ```

7. Uncertainty card: wrap UncertaintyChipGroup + Textarea for uncertaintyNotes in a Card; both disabled when `!isEditable`.

8. Line items card: `<EstimateLineItemsCard estimate={estimate} businessId={businessId} />` — the inner edit gate is already `status === "draft"`.

9. Internal notes card: Textarea with `value={notesDraft}`, `onChange={(e) => setNotesDraft(e.target.value)}`, `onBlur={handleNotesBlur}`, `disabled={!isEditable || isUpdating}`.

10. Response summary placeholder card:
    ```jsx
    <Card>
      <CardHeader><CardTitle>Customer response</CardTitle></CardHeader>
      <CardContent>
        <p className="text-sm text-muted-foreground">
          No customer response yet. Response handling ships in Phase 45.
        </p>
      </CardContent>
    </Card>
    ```

11. EstimateActionStrip integration:
    ```jsx
    {businessId && (
      <EstimateActionStrip
        estimate={estimate}
        businessId={businessId}
        customerEmail={customer?.email}
        customerName={customer?.name ?? estimate.customerName}
        onDelete={() => setEstimateToDelete(estimate)}
      />
    )}
    ```

12. Delete confirmation AlertDialog — same pattern as QuoteDetailPage / QuotesPage:
    ```jsx
    <AlertDialog open={estimateToDelete !== null} onOpenChange={(open) => { if (!open) setEstimateToDelete(null); }}>
      <AlertDialogContent>
        <AlertDialogHeader>
          <AlertDialogTitle>Delete {estimateToDelete?.number}?</AlertDialogTitle>
          <AlertDialogDescription>
            Delete {estimateToDelete?.number} for {estimateToDelete?.customerName}? This cannot be undone.
          </AlertDialogDescription>
        </AlertDialogHeader>
        <AlertDialogFooter>
          <AlertDialogCancel>Cancel</AlertDialogCancel>
          <AlertDialogAction className={buttonVariants({ variant: "destructive" })} onClick={handleDeleteConfirm}>
            Delete
          </AlertDialogAction>
        </AlertDialogFooter>
      </AlertDialogContent>
    </AlertDialog>
    ```
    where `handleDeleteConfirm` calls `deleteEstimate({ businessId, estimateId: estimateToDelete.id }).unwrap()`, toasts success `"Estimate deleted"`, navigates to `/estimates`, toasts error `"Failed to delete estimate"` on throw.

13. Loading skeleton + error state + not-found state — mirror QuoteDetailPage exactly with estimate wording.

14. Wrap in `<DashboardLayout user={user} isLoading={isLoading}>` with an inner `EstimateDetailContent()` function.

Run `cd trade-flow-ui && npm run ci` — fix any issues inline.
  </action>
  <verify>
    <automated>cd trade-flow-ui && npm run typecheck && npm run lint src/pages/EstimateDetailPage.tsx</automated>
  </verify>
  <acceptance_criteria>
    - File `trade-flow-ui/src/pages/EstimateDetailPage.tsx` exists
    - `grep -c "useGetEstimateQuery\|useUpdateEstimateMutation\|useDeleteEstimateMutation" trade-flow-ui/src/pages/EstimateDetailPage.tsx` returns at least 3
    - `grep -c "formatRange" trade-flow-ui/src/pages/EstimateDetailPage.tsx` returns at least 1
    - `grep -c "ContingencySlider" trade-flow-ui/src/pages/EstimateDetailPage.tsx` returns at least 1
    - `grep -c "UncertaintyChipGroup" trade-flow-ui/src/pages/EstimateDetailPage.tsx` returns at least 1
    - `grep -c "EstimateLineItemsCard" trade-flow-ui/src/pages/EstimateDetailPage.tsx` returns at least 1
    - `grep -c "EstimateActionStrip" trade-flow-ui/src/pages/EstimateDetailPage.tsx` returns at least 1
    - `grep -c "status === \"draft\"" trade-flow-ui/src/pages/EstimateDetailPage.tsx` returns at least 1
    - `grep -c "contingencyPercent\|displayMode\|uncertaintyReasons\|uncertaintyNotes" trade-flow-ui/src/pages/EstimateDetailPage.tsx` returns at least 6
    - `grep -c "No customer response yet" trade-flow-ui/src/pages/EstimateDetailPage.tsx` returns at least 1
    - `grep -c "Delete.*cannot be undone" trade-flow-ui/src/pages/EstimateDetailPage.tsx` returns at least 1
    - `grep -c "@ts-ignore\|@ts-expect-error\|@ts-nocheck\|eslint-disable" trade-flow-ui/src/pages/EstimateDetailPage.tsx` returns 0
    - `cd trade-flow-ui && npm run typecheck` exits 0
  </acceptance_criteria>
  <done>EstimateDetailPage renders everything the phase needs: price range card, contingency slider, display mode toggle, uncertainty card, line items card, notes card, response summary placeholder, delete confirm. Typecheck clean.</done>
</task>

<task type="auto" tdd="false">
  <name>Task 2: Wire /estimates routing, add sidebar link, migrate JobDetailPage + QuotesPage to CreateDocumentDialog</name>
  <files>trade-flow-ui/src/App.tsx, trade-flow-ui/src/config/navigation.ts, trade-flow-ui/src/pages/JobDetailPage.tsx, trade-flow-ui/src/pages/QuotesPage.tsx</files>
  <read_first>
    - trade-flow-ui/src/App.tsx (full file — where to add two new Route entries)
    - trade-flow-ui/src/config/navigation.ts (for the Main section items array)
    - trade-flow-ui/src/pages/JobDetailPage.tsx (current CreateQuoteDialog usage around line 214 + button rename around line ~100-180)
    - trade-flow-ui/src/pages/QuotesPage.tsx (current CreateQuoteDialog import and render around lines 22 and 103)
    - trade-flow-ui/src/components/CreateDocumentDialog.tsx (for the component API)
    - .planning/phases/43-estimate-frontend-crud/43-CONTEXT.md D-ROUTE-01/02/03, D-DLG-03/04/05
  </read_first>
  <action>
**Step 1 — `App.tsx`**. Add two new imports and two new routes. Insert import after the existing `QuotesPage` import:

```typescript
import EstimatesPage from "@/pages/EstimatesPage";
import EstimateDetailPage from "@/pages/EstimateDetailPage";
```

Add the routes immediately after the existing `/quotes` and `/quotes/:quoteId` routes, inside the same `<Route element={<PaywallGuard />}>` block:

```jsx
<Route path="/estimates" element={<EstimatesPage />} />
<Route path="/estimates/:estimateId" element={<EstimateDetailPage />} />
```

Do NOT reorder existing routes. Do NOT touch guards.

**Step 2 — `navigation.ts`**. Add an "Estimates" entry to the Main section items array, immediately after the "Quotes" entry and before "Items". Import `ClipboardList` from `lucide-react` at the top:

```typescript
import { Briefcase, Building2, ClipboardList, FileText, HelpCircle, Home, LayoutDashboard, Package, Settings, Users } from "lucide-react";
```

Add the new nav item:

```typescript
{
  title: "Estimates",
  href: "/estimates",
  icon: ClipboardList,
  description: "Rough-price estimates and follow-ups",
},
```

placed between the existing `Quotes` item and the `Items` item.

**Step 3 — `JobDetailPage.tsx`**:

1. Change the import line that brings in `CreateQuoteDialog` — remove `CreateQuoteDialog` from the `@/features/quotes` import destructure (keep `useGetQuotesQuery`). Add a new import: `import { CreateDocumentDialog } from "@/components/CreateDocumentDialog";`

2. Find the button that currently reads "Create Quote" (search for `Create Quote` in the file — there should be one or two references near the schedule/quote actions). Rename the button label to `"Create Document"`. Keep the icon (likely `<Plus />`).

3. Rename the corresponding state variable from `createQuoteOpen` → `createDocumentOpen` (and setter) for clarity. Optional but preferred — fall back to leaving the name alone if it causes wide-reaching churn.

4. Replace the `<CreateQuoteDialog ... />` JSX with:
   ```jsx
   <CreateDocumentDialog
     open={createDocumentOpen}
     onOpenChange={setCreateDocumentOpen}
     businessId={businessId}
     prefilledJobId={jobId}
     prefilledJobTitle={job?.title}
   />
   ```
   Note the deliberate absence of `defaultType` — the toggle is visible on JobDetailPage per D-DLG-03.

**Step 4 — `QuotesPage.tsx`**:

1. Remove `CreateQuoteDialog` from the `@/features/quotes` import destructure (keep `useGetQuotesQuery`, `useDeleteQuoteMutation`, `QuotesDataView`).

2. Add import: `import { CreateDocumentDialog } from "@/components/CreateDocumentDialog";`

3. Replace the `<CreateQuoteDialog ... />` render with:
   ```jsx
   {businessId && (
     <CreateDocumentDialog
       open={createOpen}
       onOpenChange={setCreateOpen}
       businessId={businessId}
       defaultType="quote"
     />
   )}
   ```

**Step 5 — delete `CreateQuoteDialog.tsx` and prune its barrel export**:

1. DELETE file `trade-flow-ui/src/features/quotes/components/CreateQuoteDialog.tsx`
2. EDIT `trade-flow-ui/src/features/quotes/components/index.ts` — remove the line `export { CreateQuoteDialog } from "./CreateQuoteDialog";`

Verify no other file still imports `CreateQuoteDialog`:

```bash
grep -r "CreateQuoteDialog" trade-flow-ui/src/
```

Must return zero matches (except maybe in `CreateQuoteForm.tsx` or a comment if any — clean those up).

Run `cd trade-flow-ui && npm run ci` — this is the final phase-level CI gate. It must exit 0.
  </action>
  <verify>
    <automated>cd trade-flow-ui && npm run ci</automated>
  </verify>
  <acceptance_criteria>
    - `grep -c "EstimatesPage\|EstimateDetailPage" trade-flow-ui/src/App.tsx` returns at least 4 (2 imports + 2 route elements)
    - `grep -c "path=\"/estimates\"" trade-flow-ui/src/App.tsx` returns at least 1
    - `grep -c "path=\"/estimates/:estimateId\"" trade-flow-ui/src/App.tsx` returns at least 1
    - `grep -c "title: \"Estimates\"" trade-flow-ui/src/config/navigation.ts` returns at least 1
    - `grep -c "href: \"/estimates\"" trade-flow-ui/src/config/navigation.ts` returns at least 1
    - `grep -c "ClipboardList" trade-flow-ui/src/config/navigation.ts` returns at least 2 (import + usage)
    - `grep -c "CreateDocumentDialog" trade-flow-ui/src/pages/JobDetailPage.tsx` returns at least 2 (import + JSX)
    - `grep -c "Create Document" trade-flow-ui/src/pages/JobDetailPage.tsx` returns at least 1
    - `grep -c "CreateQuoteDialog" trade-flow-ui/src/pages/JobDetailPage.tsx` returns 0
    - `grep -c "CreateDocumentDialog" trade-flow-ui/src/pages/QuotesPage.tsx` returns at least 2
    - `grep -c "defaultType=\"quote\"" trade-flow-ui/src/pages/QuotesPage.tsx` returns at least 1
    - `grep -c "CreateQuoteDialog" trade-flow-ui/src/pages/QuotesPage.tsx` returns 0
    - File `trade-flow-ui/src/features/quotes/components/CreateQuoteDialog.tsx` does NOT exist (`test ! -e trade-flow-ui/src/features/quotes/components/CreateQuoteDialog.tsx`)
    - `grep -c "CreateQuoteDialog" trade-flow-ui/src/features/quotes/components/index.ts` returns 0
    - Full repo grep `grep -rc "CreateQuoteDialog" trade-flow-ui/src/` returns 0 (or only matches in generated/coverage directories if present)
    - `grep -rc "@ts-ignore\|@ts-expect-error\|@ts-nocheck\|eslint-disable" trade-flow-ui/src/App.tsx trade-flow-ui/src/config/navigation.ts trade-flow-ui/src/pages/JobDetailPage.tsx trade-flow-ui/src/pages/QuotesPage.tsx` returns 0
    - `cd trade-flow-ui && npm run ci` exits 0
  </acceptance_criteria>
  <done>Routes wired, sidebar link added, JobDetailPage button renamed and migrated, QuotesPage migrated, CreateQuoteDialog file deleted and references pruned, full CI gate green.</done>
</task>

</tasks>

<threat_model>
## Trust Boundaries

| Boundary | Description |
|----------|-------------|
| URL→client | `/estimates/:estimateId` accepts arbitrary string; React Router validates shape only; ownership enforced server-side |
| user→client | Freeform notes textareas, chip toggles, slider, display-mode toggle — all flow to PATCH /v1/estimates/:id |
| client→API | Every inline edit is a PATCH; delete is DELETE; all require JWT |

## STRIDE Threat Register

| Threat ID | Category | Component | Disposition | Mitigation Plan |
|-----------|----------|-----------|-------------|-----------------|
| T-43-06-01 | Spoofing | `/estimates/:estimateId` IDOR — user types another business's id | mitigate | Phase 41 EstimatePolicy enforces ownership at repository + service layer; API returns 403/404; UI shows the already-implemented "Estimate not found" state |
| T-43-06-02 | Tampering | PATCH of non-editable field on non-Draft estimate | mitigate | Phase 41 EstimateUpdater rejects any non-Draft update; UI disables controls when `!isEditable` for UX but API is authoritative |
| T-43-06-03 | XSS | `uncertaintyNotes` / `notes` rendered back on detail page as strings | mitigate | React escapes by default; no `dangerouslySetInnerHTML` anywhere in this plan; Textarea component is a native `<textarea>` |
| T-43-06-04 | Repudiation | User claims they never changed contingency after a PATCH | accept | Phase 41 stores updatedAt; Phase 45 response-summary audit trail outside scope; trader is the sole operator |
| T-43-06-05 | Tampering | Old CreateQuoteDialog file survives the migration and is still importable | mitigate | Task 2 deletes the file AND prunes the barrel AND asserts via grep that no references remain anywhere in trade-flow-ui/src/ |
| T-43-06-06 | Denial of service | Rapid slider drag fires many PATCHes and rate-limits | accept | Radix slider fires onValueChange on every step; contingency has 6 discrete values (0,5,10,15,20,25,30); worst case is 6 PATCHes per drag; acceptable for a solo-trader app. Optional debounce is planner's discretion (D-EDIT-01 says pessimistic default) |
| T-43-06-07 | Information disclosure | EstimateDetailPage's loading state flashes a blank frame that leaks route shell before auth check | accept | Same pattern as QuoteDetailPage; AuthenticatedLayout already gates the route; no net new exposure |
</threat_model>

<verification>
`cd trade-flow-ui && npm run ci` must exit 0. This is the FINAL CI gate of the phase — test + lint + format + typecheck all green.

Phase-level smoke check (manual, non-blocking): start the dev server, navigate to `/estimates`, confirm the sidebar link renders, confirm the list renders (empty state acceptable if no estimates exist yet), open the Create Estimate dialog from QuotesPage (toggle hidden), from EstimatesPage (toggle hidden), and from JobDetailPage (toggle visible). Confirm that an estimate created via the dialog lands on `/estimates/:id` and inline-edits PATCH successfully.
</verification>

<success_criteria>
- EstimateDetailPage renders with inline-edit Draft controls and disabled non-Draft controls
- `/estimates` and `/estimates/:estimateId` routes mounted in App.tsx under the same guards as quotes
- Estimates sidebar link present in navigation.ts
- JobDetailPage button renamed to "Create Document" and uses CreateDocumentDialog (no defaultType)
- QuotesPage uses CreateDocumentDialog defaultType="quote"
- CreateQuoteDialog.tsx file deleted and barrel export pruned
- Zero references to CreateQuoteDialog remain in trade-flow-ui/src/
- `npm run ci` passes in trade-flow-ui
- No suppressions
</success_criteria>

<output>
After completion, create `.planning/phases/43-estimate-frontend-crud/43-06-SUMMARY.md`
</output>
