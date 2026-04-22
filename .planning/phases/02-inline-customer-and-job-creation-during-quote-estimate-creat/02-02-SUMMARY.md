---
phase: 02-inline-customer-and-job-creation-during-quote-estimate-creat
plan: 02
subsystem: ui-customer-inline-creation
tags: [inline-form, stacked-dialog, customer-creation, valibot]
dependency_graph:
  requires: []
  provides: [InlineCustomerForm, inlineCustomerSchema]
  affects: [CreateJobDialog]
tech_stack:
  added: []
  patterns: [stacked-dialog-sibling-pattern, name-only-inline-form]
key_files:
  created:
    - trade-flow-ui/src/features/customers/components/InlineCustomerForm.tsx
    - trade-flow-ui/src/features/customers/components/__tests__/InlineCustomerForm.test.tsx
    - trade-flow-ui/src/features/jobs/components/__tests__/CreateJobDialog.test.tsx
    - trade-flow-ui/src/lib/forms/schemas/inlineCustomer.schema.ts
  modified:
    - trade-flow-ui/src/features/jobs/components/CreateJobDialog.tsx
    - trade-flow-ui/src/lib/forms/schemas/index.ts
decisions:
  - Default customerType to "individual" for inline creation (most common case for tradespeople)
  - Kept existing search-to-create pattern alongside new "+ New customer" dialog action
metrics:
  duration: 503s
  completed: 2026-04-22T17:50:08Z
  tasks_completed: 2
  tasks_total: 2
  files_created: 4
  files_modified: 2
---

# Phase 02 Plan 02: InlineCustomerForm and CreateJobDialog Integration Summary

Name-only customer creation dialog (InlineCustomerForm) with stacked dialog integration in CreateJobDialog, using valibot validation and the existing useCreateCustomerMutation hook.

## Tasks Completed

| Task | Name | Commit | Key Files |
|------|------|--------|-----------|
| 1 | Create InlineCustomerForm component and valibot schema | `1316985` (trade-flow-ui) | InlineCustomerForm.tsx, inlineCustomer.schema.ts, InlineCustomerForm.test.tsx |
| 2 | Integrate InlineCustomerForm as stacked dialog in CreateJobDialog | `160b12e` (trade-flow-ui) | CreateJobDialog.tsx, CreateJobDialog.test.tsx |

## What Was Built

### InlineCustomerForm (new component)
- Minimal dialog with a single "Customer name" field
- Uses valibot schema (1-100 chars validation)
- Creates customer via existing `useCreateCustomerMutation` with `customerType: "individual"`
- Calls `onCustomerCreated` callback with `{ id, name }` on success
- Shows success/error toasts
- Conditionally renders content with `{open && ...}` for form reset on reopen

### CreateJobDialog (modified)
- Added always-visible "+ New customer" CommandItem in customer selector dropdown
- Opens InlineCustomerForm as a sibling dialog (stacked dialog pattern, same as CreateQuoteForm -> CreateJobDialog)
- `handleInlineCustomerCreated` sets selected customer with `type: "existing"` (customer is already persisted)
- Existing search-to-create pattern retained under "Quick Create" heading as alternative

### Valibot Schema
- `inlineCustomerSchema` with single `name` field (minLength 1, maxLength 100)
- Exported from schemas barrel index

## Test Coverage

- **InlineCustomerForm.test.tsx**: 6 tests (render, submit mutation call, onCustomerCreated callback, validation error, error toast, closed state)
- **CreateJobDialog.test.tsx**: 3 tests (renders "+ New customer" item, opens InlineCustomerForm on click, auto-selects created customer)
- All 154 tests in trade-flow-ui pass

## Deviations from Plan

None - plan executed exactly as written.

## Known Stubs

None - all data paths are wired to real API mutations.

## CI Gate Status

All our files pass lint, format, and typecheck. Full CI exits non-zero due to a pre-existing Prettier formatting issue in `src/features/jobs/components/JobTimeline.tsx` (dirty working tree from another plan's uncommitted changes). Our changes do not introduce any CI failures.

## Self-Check: PASSED

All created files exist and all commits are present in git history.
