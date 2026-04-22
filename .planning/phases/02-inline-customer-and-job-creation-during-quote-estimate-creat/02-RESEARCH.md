# Phase 2: Inline Customer and Job Creation During Quote/Estimate Creation - Research

**Researched:** 2026-04-22
**Domain:** React UI dialog chaining, RTK Query cache invalidation, NestJS job title auto-generation
**Confidence:** HIGH

## Summary

This phase adds inline entity creation within stacked dialogs so a tradesperson can go from zero (no customer, no job) to a complete quote or estimate in a single flow. The work is primarily frontend-focused with a small backend addition (auto-generated job titles). All core UI components already exist -- CreateQuoteDialog (with job-first pattern from Quick task #8), CreateJobDialog (with onJobCreated callback from Quick task #9), and CustomerFormDialog. The main engineering tasks are: (1) adding "+ New" action items to Select/Combobox dropdowns, (2) managing stacked Radix Dialog instances with parent form state preservation, (3) converting CreateEstimateDialog to the job-first pattern, and (4) adding job title auto-generation logic.

The backend requires only a small change: the job creation endpoint needs to support auto-generated titles when no explicit title is provided. The sequential number calculation (counting existing jobs per customer) can be done in the JobCreator service using the existing repository. No new API endpoints, collections, or modules are needed.

**Primary recommendation:** Build incrementally -- first enhance CreateJobDialog with inline customer creation, then update both quote and estimate dialogs to use the enhanced job creation chain.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- **D-01:** Inline customer creation is available from the job creation dialog only -- not directly from quote/estimate dialogs. Since both document types are job-first, customer creation flows through the job.
- **D-02:** The inline customer form captures name only (minimal). Users fill in email, phone, address later from the customer page.
- **D-03:** After creating a customer inline, it auto-selects in the job form's customer field. Same pattern as Quick task #9 (auto-select newly created job).
- **D-04:** Estimate creation gets full parity with quotes -- same inline job creation (with inline customer creation inside it). Both document types share the same creation chain UX.
- **D-05:** Estimate creation dialog adopts the job-first pattern from Quick task #8. User selects a job, customer auto-resolves from `job.customerId`. Consistent experience across both document types.
- **D-06:** Job selector in quote/estimate dialogs shows existing jobs plus a "+ New job" action at the bottom of the dropdown. Clicking it opens the inline job creation dialog.
- **D-07:** Inline forms use stacked dialogs -- each inline creation opens a new dialog on top. Creating a customer opens over the job dialog, which is over the quote/estimate dialog. Closing returns to the parent.
- **D-08:** Cancelling a nested dialog returns to the parent dialog. No data is lost in parent forms. Only closing the top-level quote/estimate dialog discards everything.
- **D-09:** The inline job creation reuses the existing `CreateJobDialog` as-is, with the addition of a "+ New customer" action in the customer selector.
- **D-10:** Inline job form includes: customer selector (with "+ New customer") and job type dropdown. These fields already exist in `CreateJobDialog`.
- **D-11:** Job title is auto-generated using the pattern: `{Job Type} - {Customer Name} #{sequential number}`. Example: "Plumbing - John Smith #1". User can rename from the job page later.
- **D-12:** After creating a job inline, it auto-selects in the quote/estimate dialog's job selector (existing Quick task #9 pattern).

### Claude's Discretion
- Sequential number strategy for auto-generated job titles (per-customer counter, global counter, or derived from existing jobs)
- Whether the auto-generated title is editable in the inline form or only from the job page
- RTK Query cache invalidation strategy when entities are created inline (customer -> job -> quote/estimate)
- Mobile responsiveness of stacked dialogs (may need fullscreen dialogs on small screens -- existing pattern from Quick task 260417-bwd)

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope
</user_constraints>

## Architectural Responsibility Map

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| Stacked dialog chain UX | Browser / Client | -- | Pure UI state management -- open/close states, form preservation |
| "+ New" action in selectors | Browser / Client | -- | Dropdown UI enhancement, opens child dialog |
| Inline customer creation | Browser / Client | API / Backend | Minimal form submits to existing POST /v1/customer endpoint |
| Inline job creation | Browser / Client | API / Backend | Existing CreateJobDialog submits to existing POST /v1/job endpoint |
| Job title auto-generation | API / Backend | -- | Sequential number requires counting existing jobs per customer in DB |
| Estimate job-first conversion | Browser / Client | -- | Rewriting estimate dialog to match quote dialog pattern |
| RTK Query cache invalidation | Browser / Client | -- | Tag-based cache invalidation on create mutations |
| Auto-select after create | Browser / Client | -- | Callback pattern from parent to child dialog |

## Standard Stack

### Core (already installed -- no new dependencies)

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| React | 19.2.0 | UI framework | Project standard [VERIFIED: CLAUDE.md] |
| Radix UI Dialog | 1.1-2.2 range | Stacked dialog primitives | Project standard, supports multiple open instances [VERIFIED: CLAUDE.md] |
| react-hook-form | 7.71.1 | Form state management | Project standard [VERIFIED: CLAUDE.md] |
| valibot | 1.2.0 | Schema validation | Project standard for frontend validation [VERIFIED: CLAUDE.md] |
| @reduxjs/toolkit | 2.11.2 | RTK Query cache management | Project standard for API state [VERIFIED: CLAUDE.md] |
| lucide-react | 0.563.0 | Icons (Plus icon for "+ New" actions) | Project standard [VERIFIED: CLAUDE.md] |
| sonner | 2.0.7 | Toast notifications | Project standard [VERIFIED: CLAUDE.md] |

**Installation:** No new packages needed. All dependencies are already installed.

## Architecture Patterns

### System Architecture Diagram

```
User clicks "New Quote/Estimate"
        |
        v
+---------------------------+
| CreateQuoteDialog /       |  Layer 1
| CreateEstimateDialog      |
|  - Job selector           |
|  - "+ New job" action     |
|  - Auto-resolved customer |
+---------------------------+
        |  user clicks "+ New job"
        v
+---------------------------+
| CreateJobDialog           |  Layer 2
|  - Customer selector      |
|  - "+ New customer" action|
|  - Job type selector      |
|  - Title: auto-generated  |
+---------------------------+
        |  user clicks "+ New customer"
        v
+---------------------------+
| InlineCustomerForm        |  Layer 3
|  - Name field (required)  |
|  - "Create Customer" CTA  |
+---------------------------+
        |  submit
        v
POST /v1/customer  -->  onCustomerCreated(customer)
                              |
                              v  auto-select in Layer 2
POST /v1/job  -->  onJobCreated(job)
                        |
                        v  auto-select in Layer 1
POST /v1/quote or /v1/estimate  -->  navigate to detail page
```

### Recommended Project Structure (new/modified files)

```
trade-flow-ui/src/
  features/
    customers/
      components/
        InlineCustomerForm.tsx       # NEW: minimal name-only customer creation dialog
        CustomerFormDialog.tsx        # EXISTING: no changes needed (InlineCustomerForm is separate)
    jobs/
      components/
        CreateJobDialog.tsx           # MODIFY: add "+ New customer" action in customer selector
    quotes/
      components/
        CreateQuoteDialog.tsx         # MODIFY: add "+ New job" action in job selector (may already exist)
    estimates/
      components/
        CreateEstimateDialog.tsx      # MODIFY: convert to job-first, add "+ New job" action

trade-flow-api/src/
  job/
    services/
      job-creator.service.ts          # MODIFY: add auto-title generation logic
    repositories/
      job.repository.ts              # MODIFY: add countByCustomerId method (if not exists)
```

### Pattern 1: Stacked Radix Dialogs

**What:** Multiple Radix Dialog.Root instances, each independently controlled, where child dialogs open on top of parents.

**When to use:** When inline entity creation requires a new form without discarding the parent form's state.

**Example:**
```typescript
// Parent component manages its own open state
// Source: Radix UI Dialog docs + existing CreateQuoteDialog pattern [VERIFIED: Context7]
function CreateQuoteDialog({ open, onOpenChange }: Props) {
  const [createJobOpen, setCreateJobOpen] = useState(false);

  return (
    <Dialog.Root open={open} onOpenChange={onOpenChange}>
      <Dialog.Portal>
        <Dialog.Overlay />
        <Dialog.Content>
          {/* Job selector with "+ New job" action */}
          {/* Parent form state preserved while child dialog is open */}
        </Dialog.Content>
      </Dialog.Portal>

      {/* Child dialog -- separate Dialog.Root, stacks on top */}
      <CreateJobDialog
        open={createJobOpen}
        onOpenChange={setCreateJobOpen}
        businessId={businessId}
        onJobCreated={handleJobCreated}
      />
    </Dialog.Root>
  );
}
```

**Key behavior:** Each Radix Dialog.Root manages its own focus trap and ESC handling. When a child dialog opens, it traps focus within itself. ESC closes only the topmost dialog. Parent form state is preserved because the parent component remains mounted. [VERIFIED: Context7 - Radix Dialog docs confirm each Dialog.Root is independent]

### Pattern 2: Auto-Select After Inline Creation (Callback Pattern)

**What:** After creating an entity in a child dialog, the parent dialog auto-selects it via a callback.

**When to use:** Whenever an inline creation dialog needs to communicate the newly created entity back to the parent.

**Example:**
```typescript
// Source: Quick task #9 established this pattern [VERIFIED: 260321-tua-PLAN.md]
// CreateJobDialog already has onJobCreated callback
interface CreateJobDialogProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  businessId: string;
  onJobCreated?: (job: { id: string; title: string }) => void;
}

// In parent (CreateQuoteDialog):
const handleJobCreated = (job: { id: string; title: string }) => {
  setSelectedJobId(job.id);
  setSelectedJobTitle(job.title);
  // Customer auto-resolves from job data via useEffect/useMemo
};
```

### Pattern 3: "+ New" Action in Selector Dropdown

**What:** A visually distinct action item at the bottom of a Select/Combobox dropdown, separated by a divider.

**When to use:** When users need the option to create a new entity from within a selector.

**Example:**
```typescript
// Source: UI Spec pattern [VERIFIED: 02-UI-SPEC.md]
<CommandGroup>
  {items.map((item) => (
    <CommandItem key={item.id} onSelect={() => handleSelect(item)}>
      {item.name}
    </CommandItem>
  ))}
</CommandGroup>
<Separator />
<CommandGroup>
  <CommandItem
    onSelect={() => setCreateDialogOpen(true)}
    className="text-primary font-semibold"
  >
    <Plus className="h-4 w-4 mr-2" />
    + New job
  </CommandItem>
</CommandGroup>
```

### Pattern 4: Job Title Auto-Generation (Backend)

**What:** When a job is created inline (without an explicit title), the backend generates a title using `{Job Type Name} - {Customer Name} #{N}`.

**When to use:** During inline job creation from the quote/estimate dialog chain.

**Recommendation for sequential number strategy (Claude's Discretion):**

Use a per-customer count derived from existing jobs. Query: count jobs where `customerId` matches, then N = count + 1. This is simple, deterministic, and meaningful to the user ("this is my 3rd job for John Smith").

```typescript
// In JobCreator service
const existingJobCount = await this.jobRepository.countByCustomerId(
  dto.businessId,
  dto.customerId,
);
const sequentialNumber = existingJobCount + 1;
const title = `${jobTypeName} - ${customerName} #${sequentialNumber}`;
```

**Race condition note:** Two simultaneous job creations for the same customer could produce duplicate numbers. This is acceptable for auto-generated titles since they are cosmetic and editable. No uniqueness constraint is needed.

### Anti-Patterns to Avoid

- **Sharing Dialog.Root between parent and child:** Each stacked dialog must be its own Dialog.Root instance. Nesting Dialog.Content inside a single Dialog.Root will break focus management and ESC behavior. [ASSUMED]
- **Relying on cache invalidation for auto-select:** Do NOT wait for RTK Query refetch to select the new entity. Pass the created entity directly via callback. Cache invalidation ensures the list is eventually consistent, but the callback provides immediate feedback.
- **Making job title generation client-side:** The sequential number requires knowing how many jobs exist for a customer, which is server-side data. Generate the title on the backend to avoid stale counts.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Dialog stacking/focus trap | Custom modal stack manager | Radix Dialog (separate Root instances) | Focus trap, ESC handling, overlay management are complex a11y problems |
| Form state management | Manual useState for each field | react-hook-form | Validation, dirty tracking, error state all handled |
| Cache invalidation | Manual refetch calls | RTK Query tag-based invalidation | Already established pattern; `invalidatesTags: ["Customer"]` triggers automatic refetch |
| Toast notifications | Custom notification system | sonner | Already integrated |

## Common Pitfalls

### Pitfall 1: Parent Form State Lost When Child Dialog Opens
**What goes wrong:** Opening a child dialog unmounts the parent form, losing user input.
**Why it happens:** Conditional rendering (`{!childOpen && <ParentForm />}`) instead of keeping parent mounted.
**How to avoid:** Keep parent Dialog.Root always mounted. Child dialog is a sibling Dialog.Root that visually overlays. Parent form state lives in React state/react-hook-form and persists because the component stays mounted.
**Warning signs:** User reports losing data when clicking "+ New job" then cancelling.

### Pitfall 2: ESC Key Closes All Dialogs
**What goes wrong:** Pressing ESC closes the entire dialog stack instead of just the topmost.
**Why it happens:** Single Dialog.Root wrapping everything, or custom ESC handlers that propagate.
**How to avoid:** Each dialog layer is a separate Dialog.Root. Radix handles ESC per-root automatically. Do not add custom keyboard handlers for ESC. [VERIFIED: Context7 - Radix Dialog handles ESC per instance]
**Warning signs:** Testing stacked dialogs and ESC closes everything.

### Pitfall 3: Customer Auto-Select But Job Title Not Updated
**What goes wrong:** After inline customer creation, the customer is selected in CreateJobDialog but the auto-generated title does not reflect the customer name.
**Why it happens:** Title generation happens on the backend at job creation time, not at customer selection time. This is not a bug -- the title is generated server-side when "Create Job" is submitted.
**How to avoid:** Ensure the job creation request includes the customerId and that the backend generates the title. The frontend does not need to preview the auto-generated title.
**Warning signs:** N/A -- this is by design.

### Pitfall 4: RTK Query Stale Data in Selectors After Inline Creation
**What goes wrong:** After creating a customer inline, the customer selector in CreateJobDialog still shows the old list.
**Why it happens:** Cache invalidation takes a refetch cycle. The newly created entity might not appear immediately in the dropdown.
**How to avoid:** Use the callback pattern for immediate auto-selection. The cache invalidation ensures the list is correct if the user reopens the dropdown. The auto-selected entity does not need to be in the cached list -- it is set directly via state.
**Warning signs:** Newly created entity missing from dropdown but correctly selected.

### Pitfall 5: Estimate Dialog Missing Job Response Fields
**What goes wrong:** EstimateDialog converted to job-first pattern but `customerId`/`customerName` not available on job response.
**Why it happens:** Forgetting that Quick task #8 added these fields to the job response specifically for quotes.
**How to avoid:** These fields already exist on `IJobResponse` and the frontend `Job` type (added in Quick task #8). No additional backend work needed for the estimate dialog -- it consumes the same job data.
**Warning signs:** TypeScript errors accessing `job.customerName` in estimate dialog.

## Code Examples

### InlineCustomerForm Component

```typescript
// New component: trade-flow-ui/src/features/customers/components/InlineCustomerForm.tsx
// Source: Derived from existing CustomerFormDialog pattern + D-02 decision [VERIFIED: CONTEXT.md]

interface InlineCustomerFormProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  businessId: string;
  onCustomerCreated: (customer: { id: string; name: string }) => void;
}

// Minimal form: single "name" field
// Uses react-hook-form with valibot schema: { name: v.pipe(v.string(), v.minLength(1), v.maxLength(100)) }
// On submit: calls useCreateCustomerMutation with { name, businessId }
// On success: calls onCustomerCreated({ id, name }), shows toast, closes dialog
```

### Job Title Auto-Generation (Backend)

```typescript
// Modified: trade-flow-api/src/job/services/job-creator.service.ts
// Source: D-11 decision [VERIFIED: CONTEXT.md]

// When title is not provided (inline creation):
// 1. Look up job type name via jobTypeId
// 2. Look up customer name via customerId
// 3. Count existing jobs for this customer in this business
// 4. Generate: `${jobTypeName} - ${customerName} #${count + 1}`
```

### Estimate Dialog Job-First Conversion

```typescript
// Modified: trade-flow-ui/src/features/estimates/components/CreateEstimateDialog.tsx
// Source: D-05 decision, mirrors CreateQuoteDialog pattern [VERIFIED: Quick task #8 plan]

// Replace customer-first selection with job-first:
// 1. Job selector (with "+ New job" action) -- primary selection
// 2. Customer auto-resolved from job.customerId / job.customerName -- read-only display
// 3. CreateJobDialog child for inline creation (with onJobCreated callback)
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Customer-first quote creation | Job-first quote creation | Quick task #8 (2026-03-21) | Customer auto-resolves from job |
| Manual job selection after create | Auto-select via callback | Quick task #9 (2026-03-21) | Immediate feedback pattern |
| Full-height modals on mobile | Fullscreen modals on mobile | Quick task 260417-bwd (2026-04-17) | Better mobile UX for stacked dialogs |

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | Separate Radix Dialog.Root instances stack correctly without custom z-index management | Architecture Patterns | Stacked dialogs may overlap incorrectly; would need manual z-index |
| A2 | CreateEstimateDialog currently has a customer-first pattern (not yet job-first) | Architecture Patterns | If already job-first, less work needed |
| A3 | Job repository does not yet have a countByCustomerId method | Architecture Patterns | If it exists, less backend work |
| A4 | The inline customer creation only needs name (no other required fields on the API) | Code Examples | If API requires more fields, InlineCustomerForm needs more inputs |
| A5 | Existing CreateJobDialog uses a Command/Combobox pattern for customer selection (not a plain Select) | Architecture Patterns | The "+ New customer" action integration differs for Select vs Command |

## Open Questions

1. **Does the customer create API accept name-only?**
   - What we know: D-02 says name only. The `create-customer.request.ts` exists but the actual required fields are not visible (no repo access).
   - What's unclear: Whether email/phone are required by the API validation decorators.
   - Recommendation: Check `create-customer.request.ts` at execution time. If other fields are required, either make them optional on the API or pass empty strings. The context decision says name-only.

2. **Exact selector component type in CreateJobDialog**
   - What we know: Quick task #8 used Command/Combobox pattern in CreateQuoteDialog for job selection.
   - What's unclear: Whether CreateJobDialog uses the same Command pattern or a plain Select for customer selection.
   - Recommendation: Check at execution time. The "+ New" action works with both patterns (CommandItem vs SelectItem with a separator).

3. **Does CreateEstimateDialog exist yet or is it a new component?**
   - What we know: The CONTEXT.md references it as a canonical file. The estimates feature directory exists in STRUCTURE.md.
   - What's unclear: The exact current implementation (it may be a stub or fully implemented with a different pattern).
   - Recommendation: Executor must read the file and adapt accordingly.

## Environment Availability

Step 2.6: SKIPPED (no external dependencies identified). This phase modifies existing frontend components and one backend service. All tools and dependencies are already installed in both repositories.

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework (UI) | Vitest 4.1.3 with @testing-library/react |
| Framework (API) | Jest 30.2.0 with @nestjs/testing |
| Config file (UI) | vite.config.ts (mergeConfig) + vitest.setup.ts |
| Config file (API) | jest.config.js |
| Quick run command (UI) | `cd trade-flow-ui && npm run test` |
| Quick run command (API) | `cd trade-flow-api && npm run test` |
| Full suite command | `cd trade-flow-api && npm run ci && cd ../trade-flow-ui && npm run ci` |

### Phase Requirements -> Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| D-01 | InlineCustomerForm only accessible from CreateJobDialog | unit | `cd trade-flow-ui && npx vitest run src/features/customers/components/__tests__/InlineCustomerForm.test.tsx` | Wave 0 |
| D-02 | InlineCustomerForm has name-only field | unit | Same as above | Wave 0 |
| D-03 | Customer auto-selects after inline creation | unit | `cd trade-flow-ui && npx vitest run src/features/jobs/components/__tests__/CreateJobDialog.test.tsx` | Wave 0 |
| D-05 | Estimate dialog uses job-first pattern | unit | `cd trade-flow-ui && npx vitest run src/features/estimates/components/__tests__/CreateEstimateDialog.test.tsx` | Wave 0 |
| D-11 | Job title auto-generated | unit | `cd trade-flow-api && npx jest src/job/test/services/job-creator.service.spec.ts` | Check existing |
| D-12 | Job auto-selects in quote/estimate after inline creation | unit | `cd trade-flow-ui && npx vitest run src/features/quotes/components/__tests__/CreateQuoteDialog.test.tsx` | Check existing |

### Sampling Rate
- **Per task commit:** Quick run for affected repo (`npm run test` in modified repo)
- **Per wave merge:** Full `npm run ci` in both repos
- **Phase gate:** Full CI green before verification

### Wave 0 Gaps
- [ ] `trade-flow-ui/src/features/customers/components/__tests__/InlineCustomerForm.test.tsx` -- covers D-01, D-02
- [ ] `trade-flow-ui/src/features/jobs/components/__tests__/CreateJobDialog.test.tsx` -- covers D-03 (enhanced with inline customer)
- [ ] `trade-flow-ui/src/features/estimates/components/__tests__/CreateEstimateDialog.test.tsx` -- covers D-05

## Security Domain

### Applicable ASVS Categories

| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | No | Existing Firebase JWT -- no changes |
| V3 Session Management | No | No changes |
| V4 Access Control | Yes (existing) | Existing business ownership policies on customer/job/quote/estimate endpoints -- no changes needed |
| V5 Input Validation | Yes | Valibot (frontend), class-validator (backend) -- customer name validation |
| V6 Cryptography | No | No changes |

### Known Threat Patterns

| Pattern | STRIDE | Standard Mitigation |
|---------|--------|---------------------|
| Customer name XSS | Tampering | Backend class-validator sanitization + React auto-escaping of rendered text |
| IDOR on inline customer creation | Elevation of Privilege | Existing business ownership policy on POST /v1/customer validates businessId belongs to auth user |

No new security concerns. All inline creation uses existing authenticated endpoints with existing authorization policies.

## Sources

### Primary (HIGH confidence)
- Context7: Radix UI Dialog primitives documentation -- dialog stacking, focus trap, ESC behavior
- Context7: RTK Query automated refetching -- tag-based cache invalidation pattern
- Project codebase: `.planning/codebase/ARCHITECTURE.md`, `CONVENTIONS.md`, `STRUCTURE.md` -- existing patterns
- Project codebase: `.planning/quick/260321-thu-simplify-quote-creation-select-job-first/260321-thu-PLAN.md` -- job-first pattern
- Project codebase: `.planning/quick/260321-tua-auto-select-newly-created-job-in-quote-c/260321-tua-PLAN.md` -- auto-select callback pattern
- Project codebase: `02-CONTEXT.md` -- locked decisions
- Project codebase: `02-UI-SPEC.md` -- UI interaction contracts

### Secondary (MEDIUM confidence)
- None

### Tertiary (LOW confidence)
- None

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- all libraries already installed, versions confirmed from CLAUDE.md
- Architecture: HIGH -- builds on established patterns (Quick tasks #8, #9), Radix Dialog stacking confirmed
- Pitfalls: HIGH -- derived from concrete implementation experience in this codebase

**Research date:** 2026-04-22
**Valid until:** 2026-05-22 (stable -- no fast-moving dependencies)
