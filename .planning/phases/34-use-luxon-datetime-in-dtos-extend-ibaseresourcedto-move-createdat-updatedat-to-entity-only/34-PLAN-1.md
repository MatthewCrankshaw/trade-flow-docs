# Phase 34 — Plan 1: API Date Standardization — Shared Utility, ISubscriptionDto Alignment, and DTO Audit

**Goal:** Create a shared toDateTime utility, align ISubscriptionDto to extend IBaseResourceDto with Luxon DateTime date fields, strip createdAt/updatedAt from ISubscriptionDto, update all consumers and tests, audit remaining DTOs for native Date usage, and document standards in API CLAUDE.md.
**Repo:** trade-flow-api
**Depends on:** none

## Pre-conditions
- `trade-flow-api` repo is available and builds cleanly
- Luxon is already installed in the API (`luxon@3.5.1` in package.json)
- Quick Task 1 already converted schedule, quote, and migration modules to Luxon DateTime in DTOs

## Tasks

### Task 1: Create shared toDateTime utility
**File(s):**
- `src/core/utilities/to-date-time.utility.ts` (create)
- `src/core/test/utilities/to-date-time.utility.spec.ts` (create)

**Action:** create
**Details:**
Create a reusable Date-to-DateTime conversion utility in `src/core/utilities/` per D-08 (conversion at repository boundary). This eliminates duplicating `DateTime.fromJSDate(date, { zone: "utc" })` across repositories and ensures UTC zone is never forgotten.

```typescript
// src/core/utilities/to-date-time.utility.ts
import { DateTime } from "luxon";

export function toDateTime(date: Date): DateTime {
  return DateTime.fromJSDate(date, { zone: "utc" });
}

export function toOptionalDateTime(date: Date | undefined | null): DateTime | undefined {
  return date ? DateTime.fromJSDate(date, { zone: "utc" }) : undefined;
}
```

Add a unit test file at `src/core/test/utilities/to-date-time.utility.spec.ts`:
- Test `toDateTime` returns a UTC DateTime from a JS Date
- Test `toOptionalDateTime` returns undefined for null/undefined inputs
- Test `toOptionalDateTime` returns a UTC DateTime for valid Date input
- Test that the zone is always UTC regardless of system timezone

**Verification:**
- [ ] `npm test -- --testPathPattern=to-date-time.utility` passes
- [ ] `toDateTime` and `toOptionalDateTime` are exported from the utility file
- [ ] TypeScript compiles without errors: `npx tsc --noEmit`

### Task 2: Align ISubscriptionDto — extend IBaseResourceDto, strip timestamps, convert dates to Luxon DateTime
**File(s):**
- `src/subscription/data-transfer-objects/subscription.dto.ts` (modify)
- `src/subscription/entities/subscription.entity.ts` (verify — entity keeps createdAt/updatedAt)
- `src/subscription/repositories/subscription.repository.ts` (modify — add Date-to-DateTime conversion in toDto)
- `src/subscription/responses/subscription.response.ts` (modify if it references createdAt/updatedAt)

**Action:** modify
**Details:**

**Per D-03:** Make `ISubscriptionDto` extend `IBaseResourceDto`:
```typescript
import { DateTime } from "luxon";
import { IBaseResourceDto } from "@core/data-transfer-objects/base-resource.dto";

export interface ISubscriptionDto extends IBaseResourceDto {
  userId: string;
  stripeSubscriptionId: string;
  stripeCustomerId: string;
  status: string;
  currentPeriodEnd?: DateTime;
  trialEnd?: DateTime;
  canceledAt?: DateTime;
  cancelAtPeriodEnd: boolean;
  // NO createdAt or updatedAt — per D-04
}
```

**Per D-04:** Strip `createdAt` and `updatedAt` from `ISubscriptionDto`. The entity (`ISubscriptionEntity`) keeps them for MongoDB purposes.

**Per D-05:** Convert `currentPeriodEnd`, `trialEnd`, and `canceledAt` from native `Date` to Luxon `DateTime`.

**Per D-06:** Keep Stripe-specific field naming (`stripeSubscriptionId`, `stripeCustomerId`) unchanged.

**Per D-08:** Update the repository's `toDto()` method to use the new `toOptionalDateTime` utility for date conversion:
```typescript
import { toOptionalDateTime } from "@core/utilities/to-date-time.utility";

private toDto(entity: ISubscriptionEntity): ISubscriptionDto {
  return {
    id: entity._id.toHexString(),
    userId: entity.userId,
    stripeSubscriptionId: entity.stripeSubscriptionId,
    stripeCustomerId: entity.stripeCustomerId,
    status: entity.status,
    currentPeriodEnd: toOptionalDateTime(entity.currentPeriodEnd),
    trialEnd: toOptionalDateTime(entity.trialEnd),
    canceledAt: toOptionalDateTime(entity.canceledAt),
    cancelAtPeriodEnd: entity.cancelAtPeriodEnd,
  };
}
```

**Per D-09:** Luxon DateTime fields automatically serialize to ISO 8601 via `toJSON()` -> `toISO()` in JSON.stringify — no manual serialization needed.

Update the subscription response interface to match the new DTO shape (no createdAt/updatedAt). Search for any response mappers that reference these stripped fields and remove them.

**Verification:**
- [ ] `npx tsc --noEmit` compiles with zero errors — all consumers of ISubscriptionDto are type-safe
- [ ] `npm test -- --testPathPattern=subscription` passes
- [ ] `ISubscriptionDto` extends `IBaseResourceDto` (has `id: string`)
- [ ] No native `Date` type in `ISubscriptionDto` — only `DateTime` for date fields
- [ ] No `createdAt` or `updatedAt` on `ISubscriptionDto`

### Task 3: Update all ISubscriptionDto consumers — services, controllers, webhook processor, and tests
**File(s):**
- `src/subscription/services/subscription-creator.service.ts` (modify if it constructs DTOs with Date fields)
- `src/subscription/services/subscription-updater.service.ts` (modify if it references createdAt/updatedAt or uses Date)
- `src/subscription/services/subscription-retriever.service.ts` (verify)
- `src/subscription/controllers/subscription.controller.ts` (verify)
- `src/subscription/controllers/webhook.controller.ts` (verify)
- `src/worker/processors/stripe-webhook.processor.ts` (modify — Stripe timestamps need DateTime conversion)
- `src/subscription/test/**/*.spec.ts` (modify — update test mocks to use DateTime instead of Date)

**Action:** modify
**Details:**

**Webhook processor update:** The BullMQ processor that handles Stripe webhook events constructs or updates subscription data. Stripe provides Unix timestamps (seconds). Update to use `DateTime.fromSeconds(stripeTimestamp, { zone: "utc" })` for Stripe timestamp fields. Per D-08, if the processor writes through the repository, the repository handles conversion. If the processor constructs DTOs directly, it must use DateTime.

**Test mock updates:** Per Pitfall 5 from research, update all test mocks/fixtures that create `ISubscriptionDto` objects:
- Replace `new Date()` with `DateTime.now()` or `DateTime.fromISO("2026-01-01T00:00:00.000Z")`
- Remove any mock references to `createdAt`/`updatedAt` on subscription DTOs
- Update mock type assertions to match the new interface

**Service layer verification:** Services should already work with DTOs (per D-08, repositories handle conversion). Verify no service manually converts dates. If any service references `createdAt`/`updatedAt` from a subscription DTO, remove the reference.

**Controller layer verification:** Controllers should not need changes since they pass DTOs through. Verify the response shape matches the updated DTO (no createdAt/updatedAt in responses).

Search comprehensively for consumers:
```bash
grep -rn "ISubscriptionDto\|createdAt\|updatedAt" src/subscription/ src/worker/
```

**Verification:**
- [ ] `npm test` passes (full test suite — subscription tests plus any dependent tests)
- [ ] `npx tsc --noEmit` compiles cleanly
- [ ] `grep -rn "createdAt\|updatedAt" src/subscription/data-transfer-objects/` returns no matches
- [ ] `grep -rn "new Date" src/subscription/test/` returns no matches (all mocks use DateTime)

### Task 4: Audit all DTOs for native Date usage and update API CLAUDE.md
**File(s):**
- All `src/*/data-transfer-objects/*.dto.ts` files (audit — modify only if native Date found)
- `CLAUDE.md` (modify)

**Action:** modify
**Details:**

**DTO audit (per D-07 — no native Date in any DTO):**
Run a comprehensive search across all DTO files:
```bash
grep -rn ": Date\|: Date " src/*/data-transfer-objects/
```

For any DTO still using native `Date`:
1. Change the type to `DateTime` from luxon
2. Update the corresponding repository's `toDto()` method to use `toDateTime()`/`toOptionalDateTime()` from the shared utility
3. Update any test mocks for that DTO

Note: Quick Task 1 already converted schedule, quote, and migration modules. The subscription module is converted in Tasks 2-3. This audit catches any remaining outliers (e.g., customer, job, business, item, tax-rate modules — check if they have any date fields in DTOs).

**CLAUDE.md update (per D-13):**
Add a new section to `trade-flow-api/CLAUDE.md` documenting:
- Luxon `DateTime` is the DTO standard for all date/time fields — no native JS `Date` in DTOs
- ISO 8601 UTC format for API responses (serialization is automatic via `DateTime.toJSON()`)
- ISO 8601 UTC format for MongoDB storage (per D-10)
- Conversion boundary: `Date`-to-`DateTime` conversion happens in the repository layer at entity-to-DTO mapping
- Shared utility: `toDateTime()` and `toOptionalDateTime()` from `@core/utilities/to-date-time.utility`
- Import pattern: `import { DateTime } from "luxon";`

Place this section under a heading like `## Date/Time Standards` near the existing conventions sections.

**Verification:**
- [ ] `grep -rn ": Date" src/*/data-transfer-objects/*.dto.ts` returns zero matches (excluding `DateTime`)
- [ ] `npx tsc --noEmit` compiles cleanly
- [ ] `npm test` passes
- [ ] CLAUDE.md contains the new Date/Time Standards section

## Success Criteria
- [ ] `ISubscriptionDto` extends `IBaseResourceDto` with `id: string` (D-01, D-03)
- [ ] `ISubscriptionDto` has no `createdAt` or `updatedAt` fields (D-04)
- [ ] All date fields on `ISubscriptionDto` are Luxon `DateTime` (D-05)
- [ ] Stripe field names unchanged (D-06)
- [ ] No DTO across the entire API uses native JS `Date` for date fields (D-07)
- [ ] All Date-to-DateTime conversion happens in repository layer via shared utility (D-08)
- [ ] `toDateTime` and `toOptionalDateTime` utility exists in `@core/utilities/` with tests
- [ ] Full test suite passes: `npm test`
- [ ] TypeScript compiles: `npx tsc --noEmit`
- [ ] API CLAUDE.md documents Luxon DateTime and ISO 8601 UTC standards (D-13)
