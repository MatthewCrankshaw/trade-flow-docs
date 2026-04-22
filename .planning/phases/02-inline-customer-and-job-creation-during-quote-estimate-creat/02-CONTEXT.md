# Phase 2: Inline Customer and Job Creation During Quote/Estimate Creation - Context

**Gathered:** 2026-04-22
**Status:** Ready for planning

<domain>
## Phase Boundary

Enable users to create new customers and jobs inline while creating a quote or estimate — without leaving the creation dialog. Currently, users must pre-create customers and jobs before starting a quote/estimate. After this phase, a user can go from zero (no customer, no job) to a fully created quote/estimate in a single flow using stacked dialogs.

</domain>

<decisions>
## Implementation Decisions

### Inline Customer Creation
- **D-01:** Inline customer creation is available from the job creation dialog only — not directly from quote/estimate dialogs. Since both document types are job-first, customer creation flows through the job.
- **D-02:** The inline customer form captures name only (minimal). Users fill in email, phone, address later from the customer page.
- **D-03:** After creating a customer inline, it auto-selects in the job form's customer field. Same pattern as Quick task #9 (auto-select newly created job).

### Estimate Creation Parity
- **D-04:** Estimate creation gets full parity with quotes — same inline job creation (with inline customer creation inside it). Both document types share the same creation chain UX.
- **D-05:** Estimate creation dialog adopts the job-first pattern from Quick task #8. User selects a job, customer auto-resolves from `job.customerId`. Consistent experience across both document types.

### Creation Chain UX
- **D-06:** Job selector in quote/estimate dialogs shows existing jobs plus a "+ New job" action at the bottom of the dropdown. Clicking it opens the inline job creation dialog.
- **D-07:** Inline forms use stacked dialogs — each inline creation opens a new dialog on top. Creating a customer opens over the job dialog, which is over the quote/estimate dialog. Closing returns to the parent.
- **D-08:** Cancelling a nested dialog returns to the parent dialog. No data is lost in parent forms. Only closing the top-level quote/estimate dialog discards everything.

### Job Creation Scope
- **D-09:** The inline job creation reuses the existing `CreateJobDialog` as-is, with the addition of a "+ New customer" action in the customer selector.
- **D-10:** Inline job form includes: customer selector (with "+ New customer") and job type dropdown. These fields already exist in `CreateJobDialog`.
- **D-11:** Job title is auto-generated using the pattern: `{Job Type} - {Customer Name} #{sequential number}`. Example: "Plumbing - John Smith #1". User can rename from the job page later.
- **D-12:** After creating a job inline, it auto-selects in the quote/estimate dialog's job selector (existing Quick task #9 pattern).

### Claude's Discretion
- Sequential number strategy for auto-generated job titles (per-customer counter, global counter, or derived from existing jobs)
- Whether the auto-generated title is editable in the inline form or only from the job page
- RTK Query cache invalidation strategy when entities are created inline (customer → job → quote/estimate)
- Mobile responsiveness of stacked dialogs (may need fullscreen dialogs on small screens — existing pattern from Quick task 260417-bwd)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Quote Creation (current implementation)
- `trade-flow-ui/src/features/quotes/components/CreateQuoteDialog.tsx` — Current quote creation dialog with job-first selection
- `trade-flow-api/src/quote/controllers/quote.controller.ts` — Quote creation endpoint

### Estimate Creation (current implementation)
- `trade-flow-ui/src/features/estimates/components/CreateEstimateDialog.tsx` — Current estimate creation dialog (needs job-first conversion)
- `trade-flow-api/src/estimate/controllers/estimate.controller.ts` — Estimate creation endpoint

### Job Creation (inline pattern exists)
- `trade-flow-ui/src/features/jobs/components/CreateJobDialog.tsx` — Existing job creation dialog to be reused/enhanced
- `trade-flow-api/src/job/controllers/job.controller.ts` — Job creation endpoint
- `trade-flow-api/src/job/responses/job.response.ts` — Job response includes customerId and customerName (added in Quick task #8)

### Customer Creation
- `trade-flow-ui/src/features/customers/components/CustomerFormDialog.tsx` — Existing customer form dialog (reuse for inline creation)
- `trade-flow-api/src/customer/controllers/customer.controller.ts` — Customer creation endpoint

### Prior Quick Tasks (foundational work)
- `.planning/quick/260321-thu-simplify-quote-creation-select-job-first/` — Quick task #8: job-first quote creation
- `.planning/quick/260321-tua-auto-select-newly-created-job-in-quote-c/` — Quick task #9: auto-select newly created job

### Codebase Patterns
- `.planning/codebase/CONVENTIONS.md` — Naming and architecture patterns
- `.planning/codebase/ARCHITECTURE.md` — Layer architecture reference

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `CreateJobDialog` — Existing inline job creation dialog with callback pattern (`onJobCreated`). Already used from quote creation. Needs enhancement with inline customer creation.
- `CustomerFormDialog` — Existing customer form dialog. Can be adapted for inline use with an `onCustomerCreated` callback.
- RTK Query hooks: `useGetJobsQuery`, `useGetCustomersQuery`, `useGetJobTypesQuery` — all available for selector dropdowns.
- Job-first pattern from Quick task #8 — job response includes `customerId` and `customerName` for auto-resolve.

### Established Patterns
- Auto-select on inline create: Quick task #9 established the pattern of auto-selecting a newly created entity in the parent dialog's selector.
- RTK Query cache invalidation with tags for list refreshes after mutations.
- Stacked Radix UI dialogs — existing dialog component supports nested open state.
- Mobile fullscreen dialogs — Quick task 260417-bwd established fullscreen modal pattern on mobile.

### Integration Points
- Quote creation dialog needs "+ New job" in job selector
- Estimate creation dialog needs conversion to job-first AND "+ New job" in job selector
- Job creation dialog needs "+ New customer" in customer selector
- Customer creation dialog (or a minimal variant) needs `onCustomerCreated` callback
- Job creation needs auto-title generation logic (`{JobType} - {CustomerName} #N`)

</code_context>

<specifics>
## Specific Ideas

- Job title auto-generation format: "{Job Type} - {Customer Name} #{sequential number}" (e.g., "Plumbing - John Smith #1")
- The full creation chain should feel effortless: user clicks "New Quote" → "New Job" → "New Customer" → types a name → done → job auto-created with title → quote dialog ready
- Customer creation is intentionally minimal (name only) because tradespeople often get a name from a phone call and want to send a quote immediately

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 02-inline-customer-and-job-creation-during-quote-estimate-creat*
*Context gathered: 2026-04-22*
