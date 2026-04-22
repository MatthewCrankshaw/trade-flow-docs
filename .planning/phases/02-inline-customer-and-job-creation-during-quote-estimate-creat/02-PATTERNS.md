# Phase 2: Inline Customer and Job Creation During Quote/Estimate Creation - Pattern Map

**Mapped:** 2026-04-22
**Files analyzed:** 7 (new/modified)
**Analogs found:** 7 / 7

## File Classification

| New/Modified File | Role | Data Flow | Closest Analog | Match Quality |
|-------------------|------|-----------|----------------|---------------|
| `trade-flow-ui/src/features/customers/components/InlineCustomerForm.tsx` | component | request-response | `CustomerFormDialog.tsx` | exact |
| `trade-flow-ui/src/features/jobs/components/CreateJobDialog.tsx` (MODIFY) | component | request-response | Self (already has inline customer creation via search-to-create) | self |
| `trade-flow-ui/src/features/quotes/components/CreateQuoteForm.tsx` (MODIFY) | component | request-response | Self (already has inline job creation) | self |
| `trade-flow-ui/src/features/estimates/components/CreateEstimateForm.tsx` (MODIFY) | component | request-response | `CreateQuoteForm.tsx` (identical job-first pattern) | exact |
| `trade-flow-api/src/job/services/job-creator.service.ts` (MODIFY) | service | CRUD | Self (add title auto-generation) | self |
| `trade-flow-api/src/job/repositories/job.repository.ts` (MODIFY) | repository | CRUD | Self (add countByCustomerId) | self |
| `trade-flow-ui/src/features/customers/components/__tests__/InlineCustomerForm.test.tsx` | test | -- | `SendEstimateForm.test.tsx` | role-match |

## Critical Discovery: Existing State vs CONTEXT.md Assumptions

**CreateEstimateForm.tsx already has job-first pattern with inline job creation.** The CONTEXT.md assumption (D-05) that estimate creation needs conversion to job-first is WRONG -- it was already converted. Both CreateQuoteForm and CreateEstimateForm are nearly identical in structure.

**CreateJobDialog.tsx already has inline customer creation via search-to-create pattern** (type a name, select "Create as Individual/Business"). The CONTEXT.md decision (D-09) to add "+ New customer" action needs to be reconciled with this existing pattern. The current pattern creates customers inline without opening a separate dialog.

**Key question for planner:** The existing CreateJobDialog creates customers inline by typing a name in the search box and clicking "Create as Individual/Business" -- this defers actual API creation to the submit handler. Decision D-01/D-02 wants a separate InlineCustomerForm dialog. The planner must decide: replace the existing inline-create pattern, or augment it.

## Pattern Assignments

### `trade-flow-ui/src/features/customers/components/InlineCustomerForm.tsx` (NEW component, request-response)

**Analog:** `trade-flow-ui/src/features/customers/components/CustomerFormDialog.tsx`

**Imports pattern** (lines 1-13):
```typescript
import { useState } from "react";
import { valibotResolver } from "@hookform/resolvers/valibot";
import { FormProvider, useForm } from "react-hook-form";

import { Button } from "@/components/ui/button";
import { Dialog, DialogContent, DialogDescription, DialogFooter, DialogHeader, DialogTitle } from "@/components/ui/dialog";
import { FormField } from "@/components/ui/form-field";
import { toast } from "@/lib/toast";
import { useCreateCustomerMutation } from "@/services";
```

**Props interface pattern** (lines 24-30):
```typescript
interface CustomerFormDialogProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  businessId: string;
  customer?: Customer | null;
  mode: "create" | "edit";
}
```

InlineCustomerForm should be simpler -- no mode, no customer prop, add `onCustomerCreated` callback:
```typescript
interface InlineCustomerFormProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  businessId: string;
  onCustomerCreated: (customer: { id: string; name: string }) => void;
}
```

**Form setup pattern** (lines 83-91):
```typescript
const [createCustomer, { isLoading: isCreating }] = useCreateCustomerMutation();

const form = useForm<CustomerFormValues>({
  resolver: valibotResolver(customerFormSchema),
  defaultValues: getInitialFormState(mode, customer),
});
```

**Submit + mutation pattern** (lines 113-158):
```typescript
const handleSubmit = form.handleSubmit(async (values) => {
  try {
    if (mode === "create") {
      const createData: CreateCustomerRequest = {
        name: values.name.trim(),
        customerType: values.customerType,
      };
      // ... optional fields ...
      await createCustomer({ businessId, data: createData }).unwrap();
    }
    onClose();
  } catch (error) {
    console.error("Failed to save customer:", error);
    toast.error("Failed to create customer", {
      description: "Please check your input and try again.",
    });
  }
});
```

**Dialog wrapper pattern** (lines 289-316):
```typescript
export function CustomerFormDialog({ open, onOpenChange, businessId, customer, mode }: CustomerFormDialogProps) {
  const formKey = `${mode}-${customer?.id || "new"}`;
  return (
    <Dialog open={open} onOpenChange={onOpenChange}>
      <DialogContent className="sm:max-w-lg" showCloseButton={false}>
        <DialogHeader>
          <DialogTitle>{title}</DialogTitle>
          <DialogDescription>{description}</DialogDescription>
        </DialogHeader>
        {open && (
          <CustomerFormContent key={formKey} businessId={businessId} onClose={() => onOpenChange(false)} />
        )}
      </DialogContent>
    </Dialog>
  );
}
```

**API required fields** (from `create-customer.request.ts` lines 33-48):
- `name: string` -- required, 1-64 chars
- `customerType: CustomerType` -- required enum ("individual" | "business")
- `phoneNumber`, `email`, `billingAddress` -- all optional

**Valibot schema pattern** (from `customer.schema.ts`):
```typescript
import * as v from "valibot";
export const customerFormSchema = v.object({
  customerType: v.picklist(["individual", "business"]),
  name: v.pipe(v.string(), v.minLength(1, "Name is required"), v.maxLength(100, "Name must be less than 100 characters")),
  // ... other optional fields
});
export type CustomerFormValues = v.InferOutput<typeof customerFormSchema>;
```

---

### `trade-flow-ui/src/features/jobs/components/CreateJobDialog.tsx` (MODIFY, request-response)

**Current file:** Already has inline customer creation via Command search-to-create pattern.

**Current inline customer creation pattern** (lines 141-163):
```typescript
const handleSelectExistingCustomer = (customer: Customer) => {
  setSelectedCustomer({
    type: "existing",
    id: customer.id,
    name: customer.name,
    customerType: customer.customerType,
  });
  setCustomerPopoverOpen(false);
};

const handleCreateNewCustomer = (customerType: CustomerType) => {
  const name = customerSearch.trim();
  if (!name) return;
  setSelectedCustomer({
    type: "new",
    name,
    customerType,
  });
  setCustomerPopoverOpen(false);
};
```

**"+ New" action in Command dropdown** (lines 323-339):
```typescript
{customerSearch.trim() && !customerSearchExistsInList && (
  <>
    <CommandSeparator />
    <CommandGroup heading="Create New">
      <CommandItem onSelect={() => handleCreateNewCustomer("individual")}>
        <Plus className="mr-1 h-4 w-4" />
        <User className="mr-1 h-4 w-4" />
        Create &ldquo;{customerSearch.trim()}&rdquo; as Individual
      </CommandItem>
      <CommandItem onSelect={() => handleCreateNewCustomer("business")}>
        <Plus className="mr-1 h-4 w-4" />
        <Building2 className="mr-1 h-4 w-4" />
        Create &ldquo;{customerSearch.trim()}&rdquo; as Business
      </CommandItem>
    </CommandGroup>
  </>
)}
```

**Submit with deferred customer creation** (lines 194-246):
```typescript
const handleSubmit = async () => {
  if (!canSubmit) return;
  try {
    let customerId = selectedCustomer.id;
    if (selectedCustomer.type === "new") {
      const newCustomer = await createCustomer({
        businessId,
        data: { name: selectedCustomer.name, customerType: selectedCustomer.customerType },
      }).unwrap();
      customerId = newCustomer.id;
    }
    // ... create job type if new ...
    const newJob = await createJob({ businessId, data: { title, customerId, jobTypeId, description } }).unwrap();
    onJobCreated?.({ id: newJob.id, title: newJob.title });
    onOpenChange(false);
  } catch (error) {
    console.error("Failed to create job:", error);
    toast.error("Failed to create job", { description: "Please check your input and try again." });
  }
};
```

**Modification needed:** Add state for InlineCustomerForm dialog, add "+ New customer" CommandItem that opens it (always visible, not just when searching), handle `onCustomerCreated` callback.

---

### `trade-flow-ui/src/features/quotes/components/CreateQuoteForm.tsx` (MODIFY, request-response)

**Current state:** Already has job-first pattern with inline job creation via CreateJobDialog.

**Inline job creation pattern** (lines 50, 82-85, 256-262):
```typescript
const [createJobOpen, setCreateJobOpen] = useState(false);

const handleJobCreated = (job: { id: string; title: string }) => {
  setSelectedJobId(job.id);
  setSelectedJobTitle(job.title);
};

// At end of JSX:
<CreateJobDialog
  open={createJobOpen}
  onOpenChange={setCreateJobOpen}
  businessId={businessId}
  onJobCreated={handleJobCreated}
/>
```

**"+ New job" in Command dropdown** (lines 188-199):
```typescript
<CommandSeparator />
<CommandGroup>
  <CommandItem
    onSelect={() => {
      setJobPopoverOpen(false);
      setCreateJobOpen(true);
    }}
  >
    <Plus className="mr-1 h-4 w-4" />
    Add new job
  </CommandItem>
</CommandGroup>
```

**Modification scope:** Minimal or none. The "+ New job" action and CreateJobDialog integration already exist. The only change is that the underlying CreateJobDialog will now support inline customer creation (stacked dialog), which requires no changes to CreateQuoteForm itself.

---

### `trade-flow-ui/src/features/estimates/components/CreateEstimateForm.tsx` (MODIFY, request-response)

**Analog:** `CreateQuoteForm.tsx` -- nearly identical structure.

**Current state:** Already has job-first pattern with inline job creation. Lines 57, 85-88, 297-303 mirror CreateQuoteForm exactly.

**Modification scope:** Same as CreateQuoteForm -- minimal or none. The stacked dialog chain propagates automatically through CreateJobDialog.

---

### `trade-flow-api/src/job/services/job-creator.service.ts` (MODIFY, CRUD)

**Current pattern** (lines 1-55):
```typescript
import { ErrorCodes } from "@core/errors/error-codes.enum";
import { InvalidRequestError } from "@core/errors/invalid-request.error";
import { AuthorizedCreatorFactory } from "@core/factories/authorized-creator.factory";
import { ICreatorService } from "@core/interfaces/creator-service.interface";
import { AppLogger } from "@core/services/app-logger.service";
import { CustomerRetriever } from "@customer/services/customer-retriever.service";
import { IJobDto } from "@job/data-transfer-objects/job.dto";
import { JobPolicy } from "@job/policies/job.policy";
import { JobRepository } from "@job/repositories/job.repository";
import { JobTypeRetrieverService } from "@job/services/job-type-retriever.service";
import { Injectable } from "@nestjs/common";
import { IUserDto } from "@user/data-transfer-objects/user.dto";

@Injectable()
export class JobCreatorService implements ICreatorService {
  private readonly logger = new AppLogger(JobCreatorService.name);

  constructor(
    private readonly jobRepository: JobRepository,
    private readonly jobPolicy: JobPolicy,
    private readonly authorizedCreatorFactory: AuthorizedCreatorFactory,
    private readonly customerRetriever: CustomerRetriever,
    private readonly jobTypeRetriever: JobTypeRetrieverService,
    private readonly onboardingProgressUpdater: OnboardingProgressUpdater,
  ) {}

  public async create(authUser: IUserDto, job: IJobDto): Promise<IJobDto> {
    const customer = await this.customerRetriever.findByIdOrFail(authUser, job.customerId);
    // ... validation ...
    const jobType = await this.jobTypeRetriever.findById(authUser, job.jobTypeId);
    // ... validation ...
    const authorizedCreator = this.authorizedCreatorFactory.createFor(this.jobRepository, this.jobPolicy);
    const createdItem = await authorizedCreator.create(authUser, job);
    // ... onboarding steps ...
    return createdItem;
  }
}
```

**Modification needed:** After retrieving customer and jobType, check if `job.title` is empty/missing. If so, count existing jobs for this customer via repository and generate title: `${jobType.name} - ${customer.name} #${count + 1}`. Set `job.title` before passing to `authorizedCreator.create()`.

---

### `trade-flow-api/src/job/repositories/job.repository.ts` (MODIFY, CRUD)

**Current query pattern** (lines 40-49):
```typescript
public async findJobsByBusinessId(
  businessId: string,
  queryOptions: IBaseQueryOptionsDto,
): Promise<DtoCollection<IJobDto>> {
  const filter: Filter<IJobEntity> = {
    businessId: new ObjectId(businessId),
  };
  const results = await this.fetcher.findMany<IJobEntity>(JobRepository.COLLECTION, filter, queryOptions);
  return this.toDtoCollection(results);
}
```

**New method pattern** (modeled on existing query style):
```typescript
public async countByCustomerId(businessId: string, customerId: string): Promise<number> {
  const filter: Filter<IJobEntity> = {
    businessId: new ObjectId(businessId),
    customerId: new ObjectId(customerId),
  };
  return this.fetcher.count(JobRepository.COLLECTION, filter);
}
```

Note: Need to verify if `MongoDbFetcher` has a `count` method. If not, use `findMany` length or add `countDocuments` to the fetcher.

---

### `trade-flow-ui/src/features/customers/components/__tests__/InlineCustomerForm.test.tsx` (NEW test)

**Analog:** `trade-flow-ui/src/features/estimates/components/__tests__/SendEstimateForm.test.tsx`

**Test setup pattern** (lines 1-28):
```typescript
import { render, screen, waitFor } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { vi } from "vitest";

const mockCreateCustomer = vi.fn();

vi.mock("@/services", () => ({
  useCreateCustomerMutation: vi.fn(),
}));

vi.mock("@/lib/toast", () => ({
  toast: {
    success: vi.fn(),
    error: vi.fn(),
  },
}));

import { InlineCustomerForm } from "../InlineCustomerForm";
import { useCreateCustomerMutation } from "@/services";

const mockUseCreateCustomerMutation = useCreateCustomerMutation as ReturnType<typeof vi.fn>;
```

**Test structure pattern:**
```typescript
const defaultProps = {
  open: true,
  onOpenChange: vi.fn(),
  businessId: "biz-1",
  onCustomerCreated: vi.fn(),
};

describe("InlineCustomerForm", () => {
  beforeEach(() => {
    mockUseCreateCustomerMutation.mockReturnValue([mockCreateCustomer, { isLoading: false }]);
  });
  afterEach(() => {
    vi.clearAllMocks();
  });
  // ... tests ...
});
```

## Shared Patterns

### Dialog Component Pattern
**Source:** `CustomerFormDialog.tsx` lines 289-316
**Apply to:** InlineCustomerForm

All dialog components follow this structure:
- Outer function receives `open`, `onOpenChange`, `businessId` props
- Wraps content in `<Dialog open={open} onOpenChange={onOpenChange}>`
- Uses `<DialogContent className="sm:max-w-lg" showCloseButton={false}>`
- Conditionally renders form content with `{open && <Content />}` for form reset
- DialogHeader with DialogTitle + DialogDescription

### Stacked Dialog Pattern
**Source:** `CreateQuoteForm.tsx` lines 256-262
**Apply to:** CreateJobDialog (for InlineCustomerForm child)

Child dialogs are rendered as sibling JSX elements, not nested inside parent Dialog.Content:
```typescript
// End of parent form JSX
<CreateJobDialog
  open={createJobOpen}
  onOpenChange={setCreateJobOpen}
  businessId={businessId}
  onJobCreated={handleJobCreated}
/>
```

### Auto-Select After Inline Creation (Callback Pattern)
**Source:** `CreateQuoteForm.tsx` lines 82-85
**Apply to:** CreateJobDialog (for customer auto-select), CreateQuoteForm/CreateEstimateForm (for job auto-select)

```typescript
const handleJobCreated = (job: { id: string; title: string }) => {
  setSelectedJobId(job.id);
  setSelectedJobTitle(job.title);
};
```

### "+ New" Action in Command Dropdown
**Source:** `CreateQuoteForm.tsx` lines 188-199 (always visible)
**Also see:** `CreateJobDialog.tsx` lines 323-339 (visible only when search has no match)

Two variants exist:
1. **Always visible** (quote/estimate job selector): `<CommandSeparator />` + `<CommandGroup>` with `<CommandItem>` containing `<Plus />` icon
2. **Conditional on search** (job dialog customer selector): Only shown when `customerSearch.trim() && !customerSearchExistsInList`

### RTK Query Mutation + Toast Pattern
**Source:** `CustomerFormDialog.tsx` lines 113-158
**Apply to:** InlineCustomerForm

```typescript
try {
  const result = await createCustomer({ businessId, data }).unwrap();
  // callback with result
  onClose();
} catch (error) {
  console.error("Failed to create customer:", error);
  toast.error("Failed to create customer", {
    description: "Please check your input and try again.",
  });
}
```

### Backend Error Handling Pattern
**Source:** `job-creator.service.ts` lines 29-46
**Apply to:** Modified job-creator with title auto-generation

```typescript
const customer = await this.customerRetriever.findByIdOrFail(authUser, job.customerId);
if (!customer) {
  this.logger.warn("Customer not found for job creation", { customerId: job.customerId });
  throw new InvalidRequestError(
    ErrorCodes.JOB_CUSTOMER_NOT_FOUND_FOR_JOB_CREATION,
    `Customer with id ${job.customerId} not found for job creation`,
  );
}
```

### UI Test Setup Pattern
**Source:** `SendEstimateForm.test.tsx` lines 1-28
**Apply to:** All new test files

- Mock RTK Query hooks with `vi.mock()` before imports
- Mock `@/lib/toast` with `vi.fn()` stubs
- Import component after mocks
- Cast mocked hooks with `as ReturnType<typeof vi.fn>`
- Use `defaultProps` object, `beforeEach` to configure mock returns, `afterEach` with `vi.clearAllMocks()`

## No Analog Found

| File | Role | Data Flow | Reason |
|------|------|-----------|--------|
| -- | -- | -- | All files have strong analogs in the existing codebase |

## Metadata

**Analog search scope:** `trade-flow-ui/src/features/`, `trade-flow-api/src/job/`, `trade-flow-api/src/customer/`
**Files scanned:** 12 source files read, 4 glob searches, 3 grep searches
**Pattern extraction date:** 2026-04-22
