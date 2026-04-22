# Phase 2: Inline Customer and Job Creation During Quote/Estimate Creation - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-22
**Phase:** 02-inline-customer-and-job-creation-during-quote-estimate-creat
**Areas discussed:** Inline customer creation flow, Estimate creation parity, Creation chain UX, Job creation scope

---

## Inline Customer Creation Flow

| Option | Description | Selected |
|--------|-------------|----------|
| From job creation only | User creates customer from within inline job creation dialog. Since quotes/estimates are job-first, customer flows through job. | ✓ |
| From both job and quote/estimate dialogs | Customer creation available at both levels — more access points but more complex. | |
| Standalone + inline | Add "New customer" everywhere a customer selector appears. | |

**User's choice:** From job creation only
**Notes:** Keeps quote/estimate dialogs simple. Customer creation lives one level deep in the job dialog.

| Option | Description | Selected |
|--------|-------------|----------|
| Minimal — name only | Just customer name. Fill in details later. | ✓ |
| Essential — name + contact | Name, email, and phone number. | |
| Full form | Same fields as standalone customer creation. | |

**User's choice:** Minimal — name only
**Notes:** Fast inline flow. Details added later from customer page.

| Option | Description | Selected |
|--------|-------------|----------|
| Yes, auto-select | Newly created customer immediately selected in job form. | ✓ |
| No, show in dropdown | Customer appears in dropdown but user manually selects. | |

**User's choice:** Yes, auto-select
**Notes:** Same pattern as Quick task #9.

---

## Estimate Creation Parity

| Option | Description | Selected |
|--------|-------------|----------|
| Full parity | Same inline flow for both document types. | ✓ |
| Quotes only for now | Only add inline creation to quote flow. | |
| Shared dialog component | Extract job+customer selection into shared component. | |

**User's choice:** Full parity
**Notes:** Estimates and quotes get identical creation chain experience.

| Option | Description | Selected |
|--------|-------------|----------|
| Yes, job-first | Estimates use same job-first pattern as quotes. | ✓ |
| Keep current estimate flow | Estimate creation stays as-is. | |

**User's choice:** Yes, job-first
**Notes:** Consistent experience across both document types. Same Quick task #8 pattern.

---

## Creation Chain UX

| Option | Description | Selected |
|--------|-------------|----------|
| "New job" button in job selector | Dropdown shows existing jobs plus "+ New job" action. | ✓ |
| Prompt before dialog opens | Show "Create a job first" message. | |
| Wizard-style steps | Multi-step wizard: customer → job → quote/estimate. | |

**User's choice:** "New job" button in job selector
**Notes:** Layered approach within the same dialog context.

| Option | Description | Selected |
|--------|-------------|----------|
| Stacked dialogs | Each inline creation opens a new dialog on top. | ✓ |
| Inline expansion | Forms expand within the parent dialog. | |
| Slide-over panels | New forms slide in from the right. | |

**User's choice:** Stacked dialogs
**Notes:** Familiar pattern from mobile apps. Clear visual hierarchy.

| Option | Description | Selected |
|--------|-------------|----------|
| Return to parent dialog | Cancelling returns to parent. No data lost in parent forms. | ✓ |
| Close everything | Cancelling at any level closes all dialogs. | |

**User's choice:** Return to parent dialog
**Notes:** Preserves work in parent forms.

---

## Job Creation Scope

| Option | Description | Selected |
|--------|-------------|----------|
| No customer field — "+ New customer" only | Job form shows customer selector with existing customers AND "+ New customer" button. | ✓ |
| Always show customer selector | Full customer dropdown every time. | |
| Pre-fill and lock | Lock customer field if context exists. | |

**User's choice:** Customer selector with "+ New customer" (existing dialog fields)
**Notes:** Existing CreateJobDialog already has these fields.

| Option | Description | Selected |
|--------|-------------|----------|
| Minimal — title + customer | Just title and customer selection. | |
| Essential — title + customer + job type | Title, customer, and job type. | |
| Full form | Same as standalone job creation. | |

**User's choice:** Minimal with auto-generated title (freeform)
**Notes:** User specified job title should be auto-generated, keeping form to customer + job type.

| Option | Description | Selected |
|--------|-------------|----------|
| Customer name + sequential number | e.g., "John Smith #1" | |
| Generic sequential | e.g., "Job #47" | |
| Date-based | e.g., "Job 22-Apr-2026" | |

**User's choice:** Job Type - Customer Name - Sequential number (freeform)
**Notes:** e.g., "Plumbing - John Smith #1". Requires job type to be selected.

| Option | Description | Selected |
|--------|-------------|----------|
| Yes, include job type selector | Customer selector + job type dropdown. Title auto-generates. | ✓ |
| Default to first job type | Auto-pick first/default job type. | |
| Use business trade as default | Use business's trade type as default. | |

**User's choice:** Yes, include job type selector — dialog already exists
**Notes:** Existing CreateJobDialog already has these fields. Reuse as-is with inline customer addition.

---

## Claude's Discretion

- Sequential number strategy for auto-generated job titles
- Whether auto-generated title is editable in inline form
- RTK Query cache invalidation for inline entity creation chain
- Mobile responsiveness of stacked dialogs

## Deferred Ideas

None — discussion stayed within phase scope
