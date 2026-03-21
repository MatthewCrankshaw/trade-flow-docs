---
phase: quick
plan: 260321-thu
type: execute
wave: 1
depends_on: []
files_modified:
  # Backend (trade-flow-api)
  - trade-flow-api/src/job/responses/job.response.ts
  - trade-flow-api/src/job/controllers/mappers/map-job-to-response.utility.ts
  - trade-flow-api/src/job/controllers/job.controller.ts
  - trade-flow-api/src/job/services/job-retriever.service.ts
  - trade-flow-api/openapi.yaml
  # Frontend (trade-flow-ui)
  - trade-flow-ui/src/types/api.types.ts
  - trade-flow-ui/src/features/quotes/components/CreateQuoteDialog.tsx
  - trade-flow-ui/src/pages/JobDetailPage.tsx
autonomous: true
requirements: []
must_haves:
  truths:
    - "User sees a single job selector when creating a quote (no customer selector)"
    - "Selecting a job auto-resolves and displays the customer name"
    - "Prefilled flow from JobDetailPage still works (shows read-only job + customer)"
    - "Quote is created with the correct customerId derived from the selected job"
  artifacts:
    - path: "trade-flow-api/src/job/responses/job.response.ts"
      provides: "customerId and customerName fields on IJobResponse"
      contains: "customerId"
    - path: "trade-flow-ui/src/types/api.types.ts"
      provides: "Updated Job interface with customerId and customerName"
      contains: "customerName"
    - path: "trade-flow-ui/src/features/quotes/components/CreateQuoteDialog.tsx"
      provides: "Simplified job-first quote creation dialog"
  key_links:
    - from: "trade-flow-api/src/job/controllers/mappers/map-job-to-response.utility.ts"
      to: "IJobResponse"
      via: "maps customerId and customerName from IJobDto + customer lookup"
      pattern: "customerId.*customerName"
    - from: "trade-flow-ui/src/features/quotes/components/CreateQuoteDialog.tsx"
      to: "useGetJobsQuery"
      via: "job selection auto-resolves customer from job.customerId"
      pattern: "job\\.customerName"
---

<objective>
Simplify quote creation by replacing the two-step customer-then-job selection with a single job selector that auto-resolves the customer.

Purpose: Every job already has a customerId. The current flow forces the user to select a customer first, then a job -- an unnecessary step. Selecting the job first eliminates one selection and makes quote creation faster.

Output: Updated backend Job API response (includes customerId + customerName), simplified frontend CreateQuoteDialog with job-first flow.
</objective>

<execution_context>
@/Users/mcrankshaw/.claude/get-shit-done/workflows/execute-plan.md
@/Users/mcrankshaw/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@trade-flow-ui/CLAUDE.md
@trade-flow-api/CLAUDE.md

<interfaces>
<!-- Backend: Current Job response (missing customer info) -->
From trade-flow-api/src/job/responses/job.response.ts:
```typescript
export interface IJobResponse {
  id: string;
  title: string;
  description: string | null;
  status: JobStatus;
  address: { ... } | null;
}
```

From trade-flow-api/src/job/data-transfer-objects/job.dto.ts:
```typescript
export interface IJobDto extends IBaseResourceDto {
  id: string;
  businessId: string;
  customerId: string;  // <-- Already available in DTO
  title: string;
  jobTypeId: string;
  description: string | null;
  status: JobStatus;
  address: { ... } | null;
}
```

From trade-flow-api/src/job/services/job-retriever.service.ts:
```typescript
// JobModule already imports CustomerModule (which exports CustomerRetriever)
export class JobRetrieverService {
  findByIdOrFail(authUser: IUserDto, jobId: string): Promise<IJobDto>;
  findAllByBusinessId(authUser: IUserDto, businessId: string, queryOptions: IBaseQueryOptionsDto): Promise<DtoCollection<IJobDto>>;
}
```

From trade-flow-api/src/customer/services/customer-retriever.service.ts:
```typescript
export class CustomerRetriever {
  findByIdOrFail(authUser: IUserDto, id: string): Promise<ICustomerDto>;
  findAllByBusinessId(authUser: IUserDto, businessId: string, ...): Promise<DtoCollection<ICustomerDto>>;
}
```

<!-- Frontend: Current types -->
From trade-flow-ui/src/types/api.types.ts:
```typescript
export interface Job {
  id: string;
  title: string;
  description: string | null;
  status: JobStatus;
}
```

From trade-flow-ui/src/types/quote.ts:
```typescript
export interface CreateQuoteRequest {
  customerId: string;
  jobId: string;
  quoteDate: string;
  title?: string;
  notes?: string;
}
```

<!-- Frontend: Current callers of CreateQuoteDialog -->
QuotesPage.tsx (line 126): <CreateQuoteDialog open={createOpen} onOpenChange={setCreateOpen} businessId={businessId} />
JobDetailPage.tsx (line 239): <CreateQuoteDialog open={...} ... prefilledJobId={jobId} prefilledCustomerId={...} prefilledJobTitle={...} prefilledCustomerName={...} />
</interfaces>
</context>

<tasks>

<task type="auto">
  <name>Task 1: Add customerId and customerName to Job API response</name>
  <files>
    trade-flow-api/src/job/responses/job.response.ts,
    trade-flow-api/src/job/controllers/mappers/map-job-to-response.utility.ts,
    trade-flow-api/src/job/controllers/job.controller.ts,
    trade-flow-api/src/job/services/job-retriever.service.ts,
    trade-flow-api/openapi.yaml
  </files>
  <action>
    The Job API currently strips customerId from the response. Add `customerId` and `customerName` to the response so the frontend can resolve the customer from a job selection.

    **Approach -- resolve customer names at the retriever service level:**

    Since the controller layer should not have business logic, and the mapper is a pure DTO-to-response function, the cleanest approach is to enrich the job DTOs in `JobRetrieverService` before they reach the controller. However, since the retriever should only work with DTOs and the customer name is not on the job DTO, instead handle it in the controller mapper by passing an additional `customerName` parameter.

    Simpler approach: Since the controller already has access to the job DTOs (which contain `customerId`), and we need to look up customer names, the cleanest pattern is:

    1. **IJobResponse** (`job.response.ts`): Add `customerId: string` and `customerName: string` fields.

    2. **mapJobToResponse** (`map-job-to-response.utility.ts`): Change signature to accept `(job: IJobDto, customerName: string): IJobResponse`. Add `customerId: job.customerId` and `customerName` to the returned object.

    3. **JobController** (`job.controller.ts`):
       - Inject `CustomerRetriever` (already available via CustomerModule import in JobModule).
       - In `findOne`: After getting the job DTO, look up the customer: `const customer = await this.customerRetriever.findByIdOrFail(request.user, job.customerId)`. Pass `customer.name` to `mapJobToResponse(job, customer.name)`.
       - In `findAll`: After getting all jobs, collect unique customerIds, batch-fetch customers using `customerRetriever.findByIdOrFail` for each (or if a batch method exists, use it), build a `Map<string, string>` of customerId -> name. Map each job with its customer name. Note: Check if CustomerRetriever has a batch/findMany method first. If not, use Promise.all with individual lookups (the list is bounded by business size so this is fine).
       - In `create`: After creating the job, look up the customer name for the response.
       - In `update`: After updating the job, look up the customer name for the response.

    4. **openapi.yaml**: Add `customerId` (string, required) and `customerName` (string, required) to the `JobResponse` schema, after `id` and before `title`.

    5. **Update existing job controller tests** if they exist -- check `trade-flow-api/src/job/test/` for controller specs and update the mapper calls to include the new customerName parameter. If tests mock `mapJobToResponse`, update those mocks.

    Run `cd trade-flow-api && npm run validate` to confirm no type or lint errors.
  </action>
  <verify>
    <automated>cd /Users/mcrankshaw/PersonalProjects/trade-flow-api && npm run validate</automated>
  </verify>
  <done>
    Job API responses include customerId and customerName fields. All types compile. Existing tests pass.
  </done>
</task>

<task type="auto">
  <name>Task 2: Simplify CreateQuoteDialog to job-first selection</name>
  <files>
    trade-flow-ui/src/types/api.types.ts,
    trade-flow-ui/src/features/quotes/components/CreateQuoteDialog.tsx,
    trade-flow-ui/src/pages/JobDetailPage.tsx
  </files>
  <action>
    **1. Update Job type** (`api.types.ts`): Add `customerId: string` and `customerName: string` to the `Job` interface.

    **2. Rewrite CreateQuoteDialog** (`CreateQuoteDialog.tsx`):

    Simplify props -- remove `prefilledCustomerId` and `prefilledCustomerName` since the customer is now always derived from the job:
    ```typescript
    interface CreateQuoteDialogProps {
      open: boolean;
      onOpenChange: (open: boolean) => void;
      businessId: string;
      prefilledJobId?: string;    // Keep: used from JobDetailPage
      prefilledJobTitle?: string; // Keep: used from JobDetailPage
    }
    ```

    Rewrite CreateQuoteForm:
    - Remove all customer selection state (`selectedCustomerId`, `selectedCustomerName`, `customerPopoverOpen`, `customerSearch`).
    - Remove `useGetCustomersQuery` -- no longer needed.
    - Remove `filteredCustomers`, `handleSelectCustomer`.
    - Keep job selection state and logic.
    - When a job is selected in `handleSelectJob`, auto-resolve customer from `job.customerId` and `job.customerName`:
      ```typescript
      const handleSelectJob = (job: Job) => {
        setSelectedJobId(job.id);
        setSelectedJobTitle(job.title);
        setResolvedCustomerId(job.customerId);
        setResolvedCustomerName(job.customerName);
        setJobPopoverOpen(false);
        setJobSearch("");
      };
      ```
    - Show the auto-resolved customer name as read-only text below the job selector (similar to the current prefilled display):
      ```tsx
      {resolvedCustomerName && (
        <div className="space-y-1">
          <Label className="text-muted-foreground">Customer</Label>
          <p className="text-sm font-medium flex items-center gap-2">
            <User className="h-4 w-4" />
            {resolvedCustomerName}
          </p>
        </div>
      )}
      ```
    - The job selector should show ALL jobs (not filtered by customer). Remove the `selectedCustomerId` guard that currently hides the job selector.
    - Update dialog description: "Select a job to create a quote." (was "Select a customer and job, then create a quote.")
    - `canSubmit` checks: `selectedJobId && resolvedCustomerId && quoteDate.trim() && !isLoading`
    - In `handleSubmit`, use `resolvedCustomerId` for the `customerId` field in `createQuote`.
    - Keep the "Add new job" option in the job command list. The inline `CreateJobDialog` should still work.
    - For the prefilled flow (`prefilledJobId` provided): Initialize `selectedJobId`, `selectedJobTitle` from props. Look up the job from the `jobs` query to get `customerId`/`customerName`, OR accept that when prefilled we need the job data. Since `useGetJobsQuery` fetches all jobs, find the matching job and extract customer info. Alternatively, add `prefilledCustomerName` back as optional for the prefilled case, but the cleaner approach is: always fetch jobs, and when `prefilledJobId` is set, find the job in the fetched list and resolve customer from it. Use a `useEffect` to set `resolvedCustomerId`/`resolvedCustomerName` when the jobs data loads and a prefilled job ID matches.
    - Remove unused imports (`Building2` from lucide, customer-related Command components if no longer needed, `useGetCustomersQuery`).

    **3. Update JobDetailPage.tsx** (line ~239):
    Remove `prefilledCustomerId` and `prefilledCustomerName` props from the `<CreateQuoteDialog>` call. Keep `prefilledJobId` and `prefilledJobTitle`:
    ```tsx
    <CreateQuoteDialog
      open={createQuoteOpen}
      onOpenChange={setCreateQuoteOpen}
      businessId={businessId}
      prefilledJobId={jobId}
      prefilledJobTitle={job?.title}
    />
    ```

    Run `cd trade-flow-ui && npm run lint && npm run typecheck` to confirm no errors.
  </action>
  <verify>
    <automated>cd /Users/mcrankshaw/PersonalProjects/trade-flow-ui && npm run lint && npm run typecheck</automated>
  </verify>
  <done>
    CreateQuoteDialog shows a single job selector. Selecting a job auto-displays the resolved customer name. No customer dropdown exists. Prefilled flow from JobDetailPage works. Lint and typecheck pass.
  </done>
</task>

</tasks>

<verification>
1. `cd trade-flow-api && npm run validate` -- backend compiles and lints clean
2. `cd trade-flow-ui && npm run lint && npm run typecheck` -- frontend compiles and lints clean
3. Manual: Open quote creation from QuotesPage -- see job selector only, select a job, customer name appears automatically, create quote succeeds
4. Manual: Open quote creation from JobDetailPage -- job and customer are prefilled, create quote succeeds
</verification>

<success_criteria>
- Backend Job API response includes customerId and customerName
- Frontend CreateQuoteDialog has a single job selector (no customer selector)
- Selecting a job auto-resolves and displays the customer
- Quote creation works from both QuotesPage and JobDetailPage
- All lint and typecheck pass in both projects
</success_criteria>

<output>
After completion, create `.planning/quick/260321-thu-simplify-quote-creation-select-job-first/260321-thu-SUMMARY.md`
</output>
