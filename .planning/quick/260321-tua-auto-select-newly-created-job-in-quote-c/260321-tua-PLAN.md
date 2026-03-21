---
phase: quick
plan: 260321-tua
type: execute
wave: 1
depends_on: []
files_modified:
  - trade-flow-ui/src/features/jobs/components/CreateJobDialog.tsx
  - trade-flow-ui/src/features/quotes/components/CreateQuoteDialog.tsx
autonomous: true
requirements: [auto-select-new-job]
must_haves:
  truths:
    - "When user creates a new job from the quote creation dialog, the newly created job is automatically selected in the job selector"
    - "CreateJobDialog can optionally notify its parent of the newly created job"
    - "Existing CreateJobDialog usage (without callback) continues to work unchanged"
  artifacts:
    - path: "trade-flow-ui/src/features/jobs/components/CreateJobDialog.tsx"
      provides: "Optional onJobCreated callback prop"
    - path: "trade-flow-ui/src/features/quotes/components/CreateQuoteDialog.tsx"
      provides: "Auto-selection of newly created job via callback"
  key_links:
    - from: "CreateJobDialog"
      to: "CreateQuoteForm"
      via: "onJobCreated callback passing job id and title"
      pattern: "onJobCreated.*id.*title"
---

<objective>
Auto-select a newly created job in the CreateQuoteDialog's job selector.

Purpose: When creating a quote, if the user creates a new job via the "Add new job" option in the job combobox, the newly created job should be automatically selected upon creation -- eliminating the need for the user to manually find and select the job they just created.

Output: Updated CreateJobDialog with optional callback, updated CreateQuoteDialog wiring the callback to auto-select.
</objective>

<execution_context>
@/Users/mcrankshaw/.claude/get-shit-done/workflows/execute-plan.md
@/Users/mcrankshaw/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@trade-flow-ui/CLAUDE.md
@trade-flow-ui/src/features/jobs/components/CreateJobDialog.tsx
@trade-flow-ui/src/features/quotes/components/CreateQuoteDialog.tsx
</context>

<interfaces>
<!-- Key contracts the executor needs -->

From CreateJobDialog.tsx (current):
```typescript
interface CreateJobDialogProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  businessId: string;
}
// handleSubmit calls createJob().unwrap() which returns Job
// Job has: id, title, customerId, customerName, etc.
```

From CreateQuoteDialog.tsx (current):
```typescript
// Line 109: const [createJobOpen, setCreateJobOpen] = useState(false);
// Line 134: handleSelectJob sets selectedJobId and selectedJobTitle
// Line 341-345: <CreateJobDialog open={createJobOpen} onOpenChange={setCreateJobOpen} businessId={businessId} />
```
</interfaces>

<tasks>

<task type="auto">
  <name>Task 1: Add onJobCreated callback to CreateJobDialog</name>
  <files>trade-flow-ui/src/features/jobs/components/CreateJobDialog.tsx</files>
  <action>
    Add an optional `onJobCreated` callback to `CreateJobDialogProps`:
    ```typescript
    onJobCreated?: (job: { id: string; title: string }) => void;
    ```

    In `handleSubmit`, after the successful `createJob` call (line ~254, the `await createJob(...)` call), capture the result and invoke the callback before closing:
    ```typescript
    const newJob = await createJob({
      businessId,
      data: {
        title: jobTitle.trim(),
        customerId,
        jobTypeId,
        description: jobDescription.trim() || null,
      },
    }).unwrap();

    onJobCreated?.({ id: newJob.id, title: newJob.title });
    onOpenChange(false);
    ```

    This requires capturing the return value from `createJob().unwrap()` which currently is not stored (the result is discarded). The mutation already returns a `Job` object per the RTK Query definition (`builder.mutation<Job, ...>`).

    Do NOT change any other behavior. The callback is optional so all existing usages remain unaffected.
  </action>
  <verify>
    <automated>cd /Users/mcrankshaw/PersonalProjects/trade-flow-ui && npx tsc --noEmit 2>&1 | head -30</automated>
  </verify>
  <done>CreateJobDialog accepts optional onJobCreated callback and invokes it with {id, title} after successful job creation</done>
</task>

<task type="auto">
  <name>Task 2: Wire callback in CreateQuoteDialog to auto-select new job</name>
  <files>trade-flow-ui/src/features/quotes/components/CreateQuoteDialog.tsx</files>
  <action>
    In `CreateQuoteForm`, add a handler function:
    ```typescript
    const handleJobCreated = (job: { id: string; title: string }) => {
      setSelectedJobId(job.id);
      setSelectedJobTitle(job.title);
    };
    ```

    Update the `CreateJobDialog` usage (currently at line ~341-345) to pass this callback:
    ```tsx
    <CreateJobDialog
      open={createJobOpen}
      onOpenChange={setCreateJobOpen}
      businessId={businessId}
      onJobCreated={handleJobCreated}
    />
    ```

    This leverages the existing `setSelectedJobId` and `setSelectedJobTitle` state setters already used by `handleSelectJob`. The RTK Query `useGetJobsQuery` will automatically refetch (due to cache invalidation from the createJob mutation), so the job will appear in the dropdown list and be matched by `selectedJobId` in the `useMemo` for `selectedJob` (line ~120-123).

    Do NOT modify any other logic. The customer info will auto-resolve from the selected job via the existing `selectedJob` useMemo -> `resolvedCustomerId` / `resolvedCustomerName` derivation.
  </action>
  <verify>
    <automated>cd /Users/mcrankshaw/PersonalProjects/trade-flow-ui && npx tsc --noEmit 2>&1 | head -30 && npm run lint 2>&1 | tail -20</automated>
  </verify>
  <done>After creating a new job from the quote dialog, the job is automatically selected in the job combobox and customer info is auto-resolved</done>
</task>

</tasks>

<verification>
1. `npm run typecheck` passes with no errors
2. `npm run lint` passes with no errors
3. Manual verification: Open CreateQuoteDialog, click "Add new job" in job selector, create a job, confirm the dialog closes and the new job is auto-selected with customer info resolved
</verification>

<success_criteria>
- CreateJobDialog has optional onJobCreated prop, backward compatible
- CreateQuoteDialog passes onJobCreated to auto-select newly created job
- TypeScript and lint checks pass
- No regressions in existing CreateJobDialog usage elsewhere
</success_criteria>

<output>
After completion, create `.planning/quick/260321-tua-auto-select-newly-created-job-in-quote-c/260321-tua-SUMMARY.md`
</output>
