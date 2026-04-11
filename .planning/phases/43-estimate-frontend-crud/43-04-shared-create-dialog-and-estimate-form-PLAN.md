---
phase: 43-estimate-frontend-crud
plan: 04
type: execute
wave: 2
depends_on:
  - 43-01
  - 43-02
files_modified:
  - trade-flow-ui/src/features/quotes/components/CreateQuoteForm.tsx
  - trade-flow-ui/src/features/quotes/components/index.ts
  - trade-flow-ui/src/components/CreateDocumentDialog.tsx
  - trade-flow-ui/src/features/estimates/components/CreateEstimateForm.tsx
  - trade-flow-ui/src/features/estimates/components/ContingencySlider.tsx
  - trade-flow-ui/src/features/estimates/components/UncertaintyChipGroup.tsx
  - trade-flow-ui/src/features/estimates/lib/uncertainty-chips.ts
  - trade-flow-ui/src/features/estimates/components/index.ts
  - trade-flow-ui/src/features/estimates/index.ts
autonomous: true
requirements:
  - CONT-03
  - CONT-04

must_haves:
  truths:
    - "CreateQuoteForm is a standalone component extracted from CreateQuoteDialog's inner form, under features/quotes/components/"
    - "CreateDocumentDialog is a shared shell under src/components/ that accepts defaultType?: 'quote' | 'estimate' and renders either CreateQuoteForm or CreateEstimateForm"
    - "ContingencySlider wraps the new shadcn Slider with min=0, max=30, step=5, default=10"
    - "UncertaintyChipGroup renders the five trade-agnostic chips as toggleable pill buttons"
    - "CreateEstimateForm uses ContingencySlider, UncertaintyChipGroup, a display-mode pill toggle, a uncertaintyNotes Textarea, and submits via useCreateEstimateMutation"
    - "UNCERTAINTY_CHIP_LABELS map exists under features/estimates/lib/ with exactly five entries"
  artifacts:
    - path: "trade-flow-ui/src/components/CreateDocumentDialog.tsx"
      provides: "Shared dialog shell with Quote/Estimate toggle"
      contains: "CreateDocumentDialog"
    - path: "trade-flow-ui/src/features/quotes/components/CreateQuoteForm.tsx"
      provides: "Quote form body extracted from old dialog"
      contains: "CreateQuoteForm"
    - path: "trade-flow-ui/src/features/estimates/components/CreateEstimateForm.tsx"
      provides: "Estimate form body"
      contains: "CreateEstimateForm"
    - path: "trade-flow-ui/src/features/estimates/components/ContingencySlider.tsx"
      provides: "Contingency slider wrapping shadcn Slider"
      contains: "ContingencySlider"
    - path: "trade-flow-ui/src/features/estimates/lib/uncertainty-chips.ts"
      provides: "Union type + UI label map for five uncertainty chips"
      contains: "UNCERTAINTY_CHIP_LABELS"
  key_links:
    - from: "trade-flow-ui/src/components/CreateDocumentDialog.tsx"
      to: "CreateQuoteForm and CreateEstimateForm"
      via: "conditional render based on selected type"
      pattern: "CreateQuoteForm|CreateEstimateForm"
    - from: "trade-flow-ui/src/features/estimates/components/CreateEstimateForm.tsx"
      to: "useCreateEstimateMutation"
      via: "RTK Query hook from estimateApi"
      pattern: "useCreateEstimateMutation"
    - from: "trade-flow-ui/src/features/estimates/components/ContingencySlider.tsx"
      to: "trade-flow-ui/src/components/ui/slider.tsx"
      via: "wraps base Slider"
      pattern: "from \"@/components/ui/slider\""
---

<objective>
Refactor the existing `CreateQuoteDialog` inner form into a standalone `CreateQuoteForm` component, build the shared `CreateDocumentDialog` shell that owns the Quote/Estimate toggle, and ship the new `CreateEstimateForm` + `ContingencySlider` + `UncertaintyChipGroup` + chip labels module so users can create estimates via the new dialog.

Purpose: This is the user-visible create flow for estimates and the pivot from "quotes only" to "shared document create dialog" that Plan 06 wires into QuotesPage, EstimatesPage, and JobDetailPage.

Output: one refactored quote form, one new shared dialog, three new estimate-specific components, one new labels module, updated feature barrels.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/phases/43-estimate-frontend-crud/43-CONTEXT.md
@trade-flow-ui/src/features/quotes/components/CreateQuoteDialog.tsx
@trade-flow-ui/src/components/ui/dialog.tsx
@trade-flow-ui/src/components/ui/slider.tsx
@trade-flow-ui/src/components/ui/button.tsx
@trade-flow-ui/src/components/ui/textarea.tsx
@trade-flow-ui/src/components/ui/label.tsx
@trade-flow-ui/src/features/estimates/api/estimateApi.ts
@trade-flow-ui/src/types/estimate.ts
@trade-flow-ui/src/features/jobs

<interfaces>
<!-- Existing CreateQuoteDialog shape (split into shell + form in this plan) -->

From trade-flow-ui/src/features/quotes/components/CreateQuoteDialog.tsx (summarised — read the full file before editing):
- `CreateQuoteDialogProps`: `{ open, onOpenChange, businessId, prefilledJobId?, prefilledJobTitle? }`
- Outer `Dialog` renders `<CreateQuoteForm onOpenChange businessId prefilledJobId prefilledJobTitle />` when `open` is true
- Inner `CreateQuoteForm` is already a standalone function in the same file — extract it into its own file unchanged, then delete the outer `CreateQuoteDialog` shell in Plan 06 once all consumers have migrated

The inner form owns: `selectedJobId`, `selectedJobTitle`, `quoteDate`, `notes`, `jobPopoverOpen`, `jobSearch`, `createJobOpen` state, `useGetJobsQuery`, `useCreateQuoteMutation`, job selection popover, customer display, Create button. It is ~225 lines of well-factored code — extract verbatim, do not refactor the internals.

Phase 43 locked decisions:
- D-DLG-01/02/06/07: shared shell, separate bodies, `defaultType?` prop, segmented-control toggle using two Button pills (variant="default" selected / variant="outline" unselected)
- D-DLG-06: CreateDocumentDialog home is `src/components/` (NOT under features/quotes or features/estimates)
- D-SLD-02/03: ContingencySlider wraps shadcn Slider with min=0 max=30 step=5 defaultValue=[10], echoes "X% uncertainty buffer" label beneath, fires onValueChange(percent: number)
- D-CHIP-01/02/03: five chips as pill buttons in a wrapped row, multi-select, stored as `UncertaintyReason[]`, labels in `UNCERTAINTY_CHIP_LABELS` map
- D-CHIP-04: single Textarea below chips labelled "Other notes (optional)"
- D-MOD-01: displayMode toggle as two pill buttons (same pattern as Quote/Estimate toggle)
- D-EDIT-02: CreateEstimateForm must include the contingency slider, chips, notes, displayMode, but NOT line items — line items are added on EstimateDetailPage after create (same as quotes)
- Estimate create payload (from Plan 01 CreateEstimateRequest): `{ customerId, jobId, estimateDate, notes?, contingencyPercent, displayMode, uncertaintyReasons?, uncertaintyNotes? }`

From trade-flow-ui/src/features/estimates/api/estimateApi.ts (created in Plan 03):
```typescript
useCreateEstimateMutation() returns [createEstimate, { isLoading }]
await createEstimate({ businessId, data: CreateEstimateRequest }).unwrap() → Estimate
```
</interfaces>
</context>

<tasks>

<task type="auto" tdd="false">
  <name>Task 1: Extract CreateQuoteForm + ship uncertainty chip lib + ContingencySlider + UncertaintyChipGroup</name>
  <files>trade-flow-ui/src/features/quotes/components/CreateQuoteForm.tsx, trade-flow-ui/src/features/quotes/components/CreateQuoteDialog.tsx, trade-flow-ui/src/features/quotes/components/index.ts, trade-flow-ui/src/features/estimates/lib/uncertainty-chips.ts, trade-flow-ui/src/features/estimates/components/ContingencySlider.tsx, trade-flow-ui/src/features/estimates/components/UncertaintyChipGroup.tsx</files>
  <read_first>
    - trade-flow-ui/src/features/quotes/components/CreateQuoteDialog.tsx (the FULL file — you are extracting the inner CreateQuoteForm)
    - trade-flow-ui/src/features/quotes/components/index.ts (current barrel to see what is exported)
    - trade-flow-ui/src/components/ui/slider.tsx (created in plan 02)
    - trade-flow-ui/src/components/ui/button.tsx (for Button variant props)
    - trade-flow-ui/src/types/estimate.ts (for UncertaintyReason type)
    - .planning/phases/43-estimate-frontend-crud/43-CONTEXT.md decisions D-CHIP-01 through D-CHIP-06 and D-SLD-01 through D-SLD-05
  </read_first>
  <action>
**Step 1 — extract CreateQuoteForm into its own file.**

Create NEW file `trade-flow-ui/src/features/quotes/components/CreateQuoteForm.tsx`. Copy the entire existing `CreateQuoteForm` function (currently living inside `CreateQuoteDialog.tsx`, ~lines 59-295) verbatim into the new file. Add appropriate imports (exactly the imports the inner form needs — check the read_first file). Export the component as `export function CreateQuoteForm(props: CreateQuoteFormProps)`. The `CreateQuoteFormProps` interface also moves to this file. Do NOT change internal behaviour.

Then EDIT `CreateQuoteDialog.tsx` to import `CreateQuoteForm` from the new file instead of defining it inline. The outer Dialog shell stays in place for now (Plan 06 will delete it after migrating consumers). The file should become a thin wrapper that imports and uses `CreateQuoteForm`.

EDIT `trade-flow-ui/src/features/quotes/components/index.ts` to add `export * from "./CreateQuoteForm"` alongside the existing exports.

**Step 2 — create the uncertainty chips library module.**

Create NEW directory `trade-flow-ui/src/features/estimates/lib/` and NEW file `uncertainty-chips.ts`:

```typescript
import type { UncertaintyReason } from "@/types";

export const UNCERTAINTY_CHIP_LABELS: Record<UncertaintyReason, string> = {
  site_inspection: "Site inspection needed",
  hidden_conditions: "Hidden conditions",
  materials_supply: "Materials & supply",
  access_working_space: "Access & working space",
  scope_unclear: "Scope unclear until investigation",
};

export const UNCERTAINTY_CHIP_ORDER: readonly UncertaintyReason[] = [
  "site_inspection",
  "hidden_conditions",
  "materials_supply",
  "access_working_space",
  "scope_unclear",
] as const;
```

Do NOT import from features/quotes. Do NOT define a new enum. Re-export the `UncertaintyReason` union from `@/types`.

**Step 3 — create ContingencySlider.**

Create NEW file `trade-flow-ui/src/features/estimates/components/ContingencySlider.tsx`:

```typescript
import { Label } from "@/components/ui/label";
import { Slider } from "@/components/ui/slider";
import { cn } from "@/lib/utils";

interface ContingencySliderProps {
  value: number;
  onValueChange: (percent: number) => void;
  disabled?: boolean;
  className?: string;
  id?: string;
}

export function ContingencySlider({ value, onValueChange, disabled, className, id }: ContingencySliderProps) {
  const sliderId = id ?? "contingency-slider";
  return (
    <div className={cn("space-y-3", className)}>
      <div className="flex items-center justify-between">
        <Label htmlFor={sliderId}>Contingency (uncertainty buffer)</Label>
        <span className="text-sm font-medium tabular-nums" aria-live="polite">
          {value}%
        </span>
      </div>
      <Slider
        id={sliderId}
        min={0}
        max={30}
        step={5}
        value={[value]}
        onValueChange={(values) => {
          const next = values[0];
          if (typeof next === "number") {
            onValueChange(next);
          }
        }}
        disabled={disabled}
        aria-label="Contingency percentage"
        aria-valuemin={0}
        aria-valuemax={30}
        aria-valuenow={value}
      />
      <p className="text-xs text-muted-foreground">
        {value}% uncertainty buffer. The API computes the price range from your line items plus this percentage.
      </p>
    </div>
  );
}
```

The exact labels are load-bearing per D-SLD-03 — do not rename them.

**Step 4 — create UncertaintyChipGroup.**

Create NEW file `trade-flow-ui/src/features/estimates/components/UncertaintyChipGroup.tsx`:

```typescript
import { Check } from "lucide-react";

import { Button } from "@/components/ui/button";
import { Label } from "@/components/ui/label";
import { cn } from "@/lib/utils";
import { UNCERTAINTY_CHIP_LABELS, UNCERTAINTY_CHIP_ORDER } from "@/features/estimates/lib/uncertainty-chips";

import type { UncertaintyReason } from "@/types";

interface UncertaintyChipGroupProps {
  value: UncertaintyReason[];
  onValueChange: (next: UncertaintyReason[]) => void;
  disabled?: boolean;
  className?: string;
}

export function UncertaintyChipGroup({ value, onValueChange, disabled, className }: UncertaintyChipGroupProps) {
  const selected = new Set(value);

  const toggle = (reason: UncertaintyReason) => {
    const next = new Set(selected);
    if (next.has(reason)) {
      next.delete(reason);
    } else {
      next.add(reason);
    }
    onValueChange(UNCERTAINTY_CHIP_ORDER.filter((r) => next.has(r)));
  };

  return (
    <div className={cn("space-y-2", className)}>
      <Label>Why is this an estimate? (optional, pick any that apply)</Label>
      <div className="flex flex-wrap gap-2" role="group" aria-label="Uncertainty reasons">
        {UNCERTAINTY_CHIP_ORDER.map((reason) => {
          const isSelected = selected.has(reason);
          return (
            <Button
              key={reason}
              type="button"
              size="sm"
              variant={isSelected ? "default" : "outline"}
              onClick={() => toggle(reason)}
              disabled={disabled}
              aria-pressed={isSelected}
            >
              {isSelected && <Check className="mr-1 h-3.5 w-3.5" />}
              {UNCERTAINTY_CHIP_LABELS[reason]}
            </Button>
          );
        })}
      </div>
    </div>
  );
}
```

Run `cd trade-flow-ui && npm run typecheck && npm run lint && npm run format:check` — must all pass.
  </action>
  <verify>
    <automated>cd trade-flow-ui && npm run typecheck && npm run lint src/features/estimates src/features/quotes</automated>
  </verify>
  <acceptance_criteria>
    - File `trade-flow-ui/src/features/quotes/components/CreateQuoteForm.tsx` exists
    - `grep -c "export function CreateQuoteForm" trade-flow-ui/src/features/quotes/components/CreateQuoteForm.tsx` returns at least 1
    - `grep -c "CreateQuoteForm" trade-flow-ui/src/features/quotes/components/CreateQuoteDialog.tsx` returns at least 1 (now imports instead of defining inline)
    - `grep -c "./CreateQuoteForm" trade-flow-ui/src/features/quotes/components/index.ts` returns at least 1
    - File `trade-flow-ui/src/features/estimates/lib/uncertainty-chips.ts` exists
    - `grep -c "UNCERTAINTY_CHIP_LABELS" trade-flow-ui/src/features/estimates/lib/uncertainty-chips.ts` returns at least 1
    - `grep -c "site_inspection\|hidden_conditions\|materials_supply\|access_working_space\|scope_unclear" trade-flow-ui/src/features/estimates/lib/uncertainty-chips.ts` returns at least 5
    - `grep -c "Site inspection needed\|Hidden conditions\|Materials & supply\|Access & working space\|Scope unclear until investigation" trade-flow-ui/src/features/estimates/lib/uncertainty-chips.ts` returns at least 5
    - File `trade-flow-ui/src/features/estimates/components/ContingencySlider.tsx` exists
    - `grep -c "min={0}" trade-flow-ui/src/features/estimates/components/ContingencySlider.tsx` returns at least 1
    - `grep -c "max={30}" trade-flow-ui/src/features/estimates/components/ContingencySlider.tsx` returns at least 1
    - `grep -c "step={5}" trade-flow-ui/src/features/estimates/components/ContingencySlider.tsx` returns at least 1
    - `grep -c "uncertainty buffer" trade-flow-ui/src/features/estimates/components/ContingencySlider.tsx` returns at least 1
    - File `trade-flow-ui/src/features/estimates/components/UncertaintyChipGroup.tsx` exists
    - `grep -c "UNCERTAINTY_CHIP_ORDER.map" trade-flow-ui/src/features/estimates/components/UncertaintyChipGroup.tsx` returns at least 1
    - `grep -c "aria-pressed" trade-flow-ui/src/features/estimates/components/UncertaintyChipGroup.tsx` returns at least 1
    - `grep -rc "@ts-ignore\|@ts-expect-error\|@ts-nocheck\|eslint-disable" trade-flow-ui/src/features/estimates trade-flow-ui/src/features/quotes/components/CreateQuoteForm.tsx` returns 0
    - `cd trade-flow-ui && npm run typecheck` exits 0
    - `cd trade-flow-ui && npm run lint` exits 0 for touched files
  </acceptance_criteria>
  <done>CreateQuoteForm extracted to its own file, shim CreateQuoteDialog still works, uncertainty chip library + ContingencySlider + UncertaintyChipGroup all exist and typecheck clean.</done>
</task>

<task type="auto" tdd="false">
  <name>Task 2: Build CreateEstimateForm + CreateDocumentDialog shared shell</name>
  <files>trade-flow-ui/src/features/estimates/components/CreateEstimateForm.tsx, trade-flow-ui/src/features/estimates/components/index.ts, trade-flow-ui/src/features/estimates/index.ts, trade-flow-ui/src/components/CreateDocumentDialog.tsx</files>
  <read_first>
    - trade-flow-ui/src/features/quotes/components/CreateQuoteForm.tsx (created in Task 1 — CreateEstimateForm mirrors its job/customer selection UX exactly)
    - trade-flow-ui/src/features/estimates/api/estimateApi.ts (for useCreateEstimateMutation signature)
    - trade-flow-ui/src/features/estimates/components/ContingencySlider.tsx (created in Task 1)
    - trade-flow-ui/src/features/estimates/components/UncertaintyChipGroup.tsx (created in Task 1)
    - trade-flow-ui/src/components/ui/dialog.tsx (for Dialog primitives)
    - trade-flow-ui/src/components/ui/textarea.tsx
    - trade-flow-ui/src/types/estimate.ts (for CreateEstimateRequest, UncertaintyReason, EstimateDisplayMode)
    - .planning/phases/43-estimate-frontend-crud/43-CONTEXT.md decisions D-DLG-01 through D-DLG-07 and D-MOD-01
  </read_first>
  <behavior>
    - CreateEstimateForm owns local state: selectedJobId, selectedJobTitle, estimateDate, notes, contingencyPercent (default 10), displayMode (default "range"), uncertaintyReasons (default []), uncertaintyNotes (default "")
    - Submit calls useCreateEstimateMutation({ businessId, data: CreateEstimateRequest }).unwrap(), toasts success, navigates to `/estimates/${result.id}`
    - Submit disabled until job selected + estimateDate present
    - CreateDocumentDialog owns open/onOpenChange + a local selectedType state; renders Quote/Estimate segmented-control toggle as two equal pill Buttons (variant="default" selected / variant="outline" unselected), and renders the corresponding form body below; when `defaultType` is set, hides the toggle and renders the matching form immediately
    - Uses SAME job-picker popover pattern as CreateQuoteForm (D-DLG-03 implies reusing the same job/customer selection logic) — executor may factor this into a shared helper OR duplicate it inline. Duplication is acceptable per Separation-over-DRY.
  </behavior>
  <action>
**Step 1 — create CreateEstimateForm.**

Create NEW file `trade-flow-ui/src/features/estimates/components/CreateEstimateForm.tsx`. Start from `CreateQuoteForm.tsx` as the structural template for job selection and customer auto-resolution, then add the estimate-specific fields. The form must:

1. Accept props: `{ onOpenChange: (open: boolean) => void; businessId: string; prefilledJobId?: string; prefilledJobTitle?: string }`
2. Hold local state for: `selectedJobId`, `selectedJobTitle`, `estimateDate` (default `DateTime.now().toFormat("yyyy-MM-dd")`), `notes`, `contingencyPercent` (default `10`), `displayMode` (default `"range"`), `uncertaintyReasons` (default `[]`), `uncertaintyNotes` (default `""`)
3. Use the same `useGetJobsQuery` job popover + `CreateJobDialog` inline-create pattern as CreateQuoteForm
4. Render, in order:
   - Job picker (hidden if `prefilledJobId`, else popover combobox — same as CreateQuoteForm)
   - Resolved customer display (read-only)
   - Estimate Date input (type="date")
   - `<ContingencySlider value={contingencyPercent} onValueChange={setContingencyPercent} disabled={isLoading} />`
   - Display-mode toggle: two pill Buttons labelled "Range (£X - £Y)" and "From £X", using `variant="default"` for the selected one and `variant="outline"` for the unselected one, arranged side by side with a Label above ("Price display")
   - `<UncertaintyChipGroup value={uncertaintyReasons} onValueChange={setUncertaintyReasons} disabled={isLoading} />`
   - Textarea for `uncertaintyNotes` with Label "Other notes (optional)" and placeholder `"e.g. supplier has boiler on 2-week lead, prices may shift"`
   - Textarea for `notes` with Label "Internal notes (optional)" (same as quote form notes field)
5. Footer: Cancel button + submit button labelled "Create Estimate" (disabled until `selectedJobId && resolvedCustomerId && estimateDate.trim() && !isLoading`)
6. On submit, call:
   ```typescript
   const [createEstimate, { isLoading }] = useCreateEstimateMutation();
   const result = await createEstimate({
     businessId,
     data: {
       customerId: resolvedCustomerId,
       jobId: selectedJobId,
       estimateDate,
       notes: notes.trim() || undefined,
       contingencyPercent,
       displayMode,
       uncertaintyReasons: uncertaintyReasons.length > 0 ? uncertaintyReasons : undefined,
       uncertaintyNotes: uncertaintyNotes.trim() || undefined,
     },
   }).unwrap();
   toast.success("Estimate created successfully");
   onOpenChange(false);
   navigate(`/estimates/${result.id}`);
   ```
7. On error: `toast.error("Failed to create estimate", { description: "Please check your input and try again." })`

Wrap the whole form in a React fragment rendering `DialogHeader` + `DialogTitle` ("Create Estimate") + `DialogDescription` (same prefilled/not-prefilled copy as CreateQuoteForm) + body + `DialogFooter`. This matches the `CreateQuoteForm` pattern which returns its own `DialogHeader` etc. — the shell will provide the `<Dialog>` + `<DialogContent>`.

**Step 2 — create features/estimates/components/index.ts barrel.**

```typescript
export * from "./ContingencySlider";
export * from "./UncertaintyChipGroup";
export * from "./CreateEstimateForm";
```

**Step 3 — update features/estimates/index.ts to export components too.**

Append to the existing file:

```typescript
export * from "./components";
```

**Step 4 — create CreateDocumentDialog shared shell.**

Create NEW file `trade-flow-ui/src/components/CreateDocumentDialog.tsx`. This component owns the outer `<Dialog>` + `<DialogContent>` and the optional Quote/Estimate toggle, and delegates the body to either `CreateQuoteForm` or `CreateEstimateForm`:

```typescript
import { useState } from "react";

import { Button } from "@/components/ui/button";
import { Dialog, DialogContent } from "@/components/ui/dialog";
import { CreateQuoteForm } from "@/features/quotes/components/CreateQuoteForm";
import { CreateEstimateForm } from "@/features/estimates/components/CreateEstimateForm";

type DocumentType = "quote" | "estimate";

interface CreateDocumentDialogProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  businessId: string;
  defaultType?: DocumentType;
  prefilledJobId?: string;
  prefilledJobTitle?: string;
}

export function CreateDocumentDialog({
  open,
  onOpenChange,
  businessId,
  defaultType,
  prefilledJobId,
  prefilledJobTitle,
}: CreateDocumentDialogProps) {
  const [selectedType, setSelectedType] = useState<DocumentType>(defaultType ?? "quote");
  const toggleVisible = defaultType === undefined;

  return (
    <Dialog open={open} onOpenChange={onOpenChange}>
      <DialogContent className="sm:max-w-lg" showCloseButton={false}>
        {open && (
          <>
            {toggleVisible && (
              <div className="mb-4 grid grid-cols-2 gap-2" role="group" aria-label="Document type">
                <Button
                  type="button"
                  variant={selectedType === "quote" ? "default" : "outline"}
                  onClick={() => setSelectedType("quote")}
                  aria-pressed={selectedType === "quote"}
                >
                  Quote
                </Button>
                <Button
                  type="button"
                  variant={selectedType === "estimate" ? "default" : "outline"}
                  onClick={() => setSelectedType("estimate")}
                  aria-pressed={selectedType === "estimate"}
                >
                  Estimate
                </Button>
              </div>
            )}
            {selectedType === "quote" ? (
              <CreateQuoteForm
                onOpenChange={onOpenChange}
                businessId={businessId}
                prefilledJobId={prefilledJobId}
                prefilledJobTitle={prefilledJobTitle}
              />
            ) : (
              <CreateEstimateForm
                onOpenChange={onOpenChange}
                businessId={businessId}
                prefilledJobId={prefilledJobId}
                prefilledJobTitle={prefilledJobTitle}
              />
            )}
          </>
        )}
      </DialogContent>
    </Dialog>
  );
}
```

The `defaultType` check is a prop comparison, not stateful — if the caller provides `defaultType="quote"` the toggle is hidden and `selectedType` is initialised from the prop. If the caller omits `defaultType`, the toggle is visible and defaults to `"quote"` on first render.

Run `cd trade-flow-ui && npm run ci`. Fix any test/typecheck/lint issues inline — no suppressions.
  </action>
  <verify>
    <automated>cd trade-flow-ui && npm run ci</automated>
  </verify>
  <acceptance_criteria>
    - File `trade-flow-ui/src/features/estimates/components/CreateEstimateForm.tsx` exists
    - `grep -c "export function CreateEstimateForm" trade-flow-ui/src/features/estimates/components/CreateEstimateForm.tsx` returns at least 1
    - `grep -c "useCreateEstimateMutation" trade-flow-ui/src/features/estimates/components/CreateEstimateForm.tsx` returns at least 1
    - `grep -c "ContingencySlider" trade-flow-ui/src/features/estimates/components/CreateEstimateForm.tsx` returns at least 1
    - `grep -c "UncertaintyChipGroup" trade-flow-ui/src/features/estimates/components/CreateEstimateForm.tsx` returns at least 1
    - `grep -c "displayMode" trade-flow-ui/src/features/estimates/components/CreateEstimateForm.tsx` returns at least 2 (state + submit payload)
    - `grep -c "uncertaintyNotes" trade-flow-ui/src/features/estimates/components/CreateEstimateForm.tsx` returns at least 2
    - `grep -c "contingencyPercent" trade-flow-ui/src/features/estimates/components/CreateEstimateForm.tsx` returns at least 2
    - `grep -c "navigate(\`/estimates/" trade-flow-ui/src/features/estimates/components/CreateEstimateForm.tsx` returns at least 1
    - File `trade-flow-ui/src/components/CreateDocumentDialog.tsx` exists
    - `grep -c "defaultType" trade-flow-ui/src/components/CreateDocumentDialog.tsx` returns at least 2 (prop + condition)
    - `grep -c "CreateQuoteForm\|CreateEstimateForm" trade-flow-ui/src/components/CreateDocumentDialog.tsx` returns at least 2
    - `grep -c "aria-pressed" trade-flow-ui/src/components/CreateDocumentDialog.tsx` returns at least 2
    - `grep -c "./components" trade-flow-ui/src/features/estimates/index.ts` returns at least 1
    - `grep -c "CreateEstimateForm\|ContingencySlider\|UncertaintyChipGroup" trade-flow-ui/src/features/estimates/components/index.ts` returns at least 3
    - `grep -rc "@ts-ignore\|@ts-expect-error\|@ts-nocheck\|eslint-disable" trade-flow-ui/src/features/estimates trade-flow-ui/src/components/CreateDocumentDialog.tsx` returns 0
    - `cd trade-flow-ui && npm run ci` exits 0
  </acceptance_criteria>
  <done>CreateEstimateForm submits valid CreateEstimateRequest payloads via the RTK hook, CreateDocumentDialog routes between the two forms with the toggle hidden or shown depending on defaultType, npm run ci passes.</done>
</task>

</tasks>

<threat_model>
## Trust Boundaries

| Boundary | Description |
|----------|-------------|
| user→client | Freeform `uncertaintyNotes` and `notes` textareas accept arbitrary user input; displayed back on detail page in later plans |
| client→API | Form submission produces a `CreateEstimateRequest` that is PATCHed to `/v1/estimates` (validated server-side by Phase 41 class-validator) |

## STRIDE Threat Register

| Threat ID | Category | Component | Disposition | Mitigation Plan |
|-----------|----------|-----------|-------------|-----------------|
| T-43-04-01 | Tampering | User supplies `contingencyPercent` outside 0-30 or not a multiple of 5 via DevTools | mitigate | Slider enforces min=0/max=30/step=5 client-side; Phase 41 class-validator re-checks server-side (authoritative). Client is soft gate only |
| T-43-04-02 | Tampering | User supplies a chip value not in the five-value union | mitigate | TypeScript union guards at compile time; UncertaintyChipGroup only emits values from UNCERTAINTY_CHIP_ORDER; Phase 41 validates server-side |
| T-43-04-03 | XSS | `uncertaintyNotes` / `notes` rendered back on detail page | mitigate | React auto-escapes text by default; detail page (Plan 06) uses `<Textarea value={...} />` and `{text}` — no dangerouslySetInnerHTML |
| T-43-04-04 | Information disclosure | Dialog remembers selectedType across remounts, leaking intent | accept | Local component state reset on unmount; no persistence |
| T-43-04-05 | Tampering | Mass-assignment via create mutation body | mitigate | Phase 41 DTOs use class-validator `forbidNonWhitelisted: true`; extra fields rejected at validator |
</threat_model>

<verification>
`cd trade-flow-ui && npm run ci` must exit 0 at the end of this plan.

Manual spot-check (non-blocking): the existing `CreateQuoteDialog` outer shell still works with its current consumers (`QuotesPage`, `JobDetailPage`) because Plan 06 has not yet migrated them. No existing call site should break during this plan's execution.
</verification>

<success_criteria>
- CreateQuoteForm is a standalone file, no inline definition in CreateQuoteDialog
- CreateDocumentDialog exists under src/components/ and accepts `defaultType?` prop
- CreateEstimateForm uses ContingencySlider, UncertaintyChipGroup, displayMode toggle, notes textarea
- UNCERTAINTY_CHIP_LABELS map has all five entries with exact stored values and exact UI labels
- `npm run ci` passes
- No suppressions
</success_criteria>

<output>
After completion, create `.planning/phases/43-estimate-frontend-crud/43-04-SUMMARY.md`
</output>
