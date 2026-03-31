# Phase 34: Use Luxon DateTime in DTOs, Extend IBaseResourceDto, Move createdAt/updatedAt to Entity Only - Research

**Researched:** 2026-03-30
**Domain:** Date/time standardization, DTO refactoring (NestJS backend + React frontend)
**Confidence:** HIGH

## Summary

This phase is a standardization refactor across both `trade-flow-api` and `trade-flow-ui`. The pattern for Luxon DateTime in DTOs is already established in the codebase (Quick Task 1 converted schedule, quote, and migration modules). The remaining work is: (1) align `ISubscriptionDto` to extend `IBaseResourceDto` and convert its date fields to Luxon DateTime, (2) strip `createdAt`/`updatedAt` from DTOs that do not need them, (3) ensure all repositories perform `Date`-to-`DateTime` conversion at the entity-to-DTO boundary, (4) add Luxon to the frontend and standardize date handling there, and (5) document standards in both CLAUDE.md files.

The key technical insight is that Luxon's `DateTime.toJSON()` calls `toISO()` internally, which means `JSON.stringify()` in NestJS response serialization automatically produces ISO 8601 strings from DateTime fields -- no custom serializer needed. This was already proven in Quick Task 1 where the migration module adopted this approach.

**Primary recommendation:** Follow the established Quick Task 1 pattern (DateTime in DTOs, raw types in entities, repository handles conversion) and extend it to `ISubscriptionDto` and any remaining outliers. Create a shared `toDateTime` utility in `src/core/utilities/` to avoid duplicating `DateTime.fromJSDate()` calls across repositories. Add Luxon + `@types/luxon` to `trade-flow-ui` and create a `date-helpers.ts` utility module for frontend date formatting.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- **D-01:** `IBaseResourceDto` stays as `{ id: string }` -- no timestamps added to the base. It remains minimal.
- **D-02:** Individual DTOs add `createdAt`/`updatedAt` as Luxon `DateTime` only when the feature explicitly needs them. Most DTOs won't have timestamps.
- **D-03:** `ISubscriptionDto` must extend `IBaseResourceDto` -- aligning it with all other DTOs.
- **D-04:** Strip `createdAt` and `updatedAt` from `ISubscriptionDto` -- they don't belong there. Entity keeps them for DB purposes.
- **D-05:** Convert all remaining date fields on `ISubscriptionDto` (e.g., `currentPeriodEnd`, `trialEnd`) from native `Date` to Luxon `DateTime`.
- **D-06:** Keep Stripe-specific field naming (e.g., `stripeSubscriptionId`, `stripeCustomerId`) -- no renaming to project conventions. Only structural changes.
- **D-07:** No DTO may use native JS `Date`. All date fields in DTOs must be Luxon `DateTime`. This is a hard rule enforced project-wide.
- **D-08:** `Date`-to-`DateTime` conversion happens in the **repository layer**, at the entity-to-DTO mapping boundary. Same place where `ObjectId`-to-`string` conversion already occurs. Services work purely with DTOs/DateTime.
- **D-09:** Dates in API responses serialize as ISO 8601 strings in UTC format (e.g., `"2026-03-30T10:00:00.000Z"`). Luxon's `DateTime.toISO()` or equivalent.
- **D-10:** Dates stored in MongoDB also use ISO 8601 UTC format for consistency.
- **D-11:** Add Luxon to `trade-flow-ui` as a dependency.
- **D-12:** Standardize all frontend date parsing and display to use Luxon `DateTime`. Replace any raw `new Date()` or string-based date handling.
- **D-13:** Update `trade-flow-api/CLAUDE.md` to document: Luxon DateTime as the DTO date standard, ISO 8601 UTC for storage and API responses, conversion happens in repository layer.
- **D-14:** Update `trade-flow-ui/CLAUDE.md` to document: Luxon DateTime for date parsing/display, ISO 8601 UTC format expected from API.

### Claude's Discretion
- Whether to create a shared `toDateTime(date: Date): DateTime` utility or inline the conversion in each repository
- How to handle edge cases where existing date fields are already strings vs Date objects
- Frontend Luxon integration approach (global helpers, per-component imports, or a date utility module)
- Ordering of changes (API first then UI, or parallel)

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope.
</user_constraints>

## Project Constraints (from CLAUDE.md)

- **Two repos:** Changes span `trade-flow-api` and `trade-flow-ui` with coordinated feature branches
- **Backend conventions:** Strict Controller -> Service -> Repository layering; kebab-case files with type suffixes
- **Path aliases:** `@core/*` -> `src/core/`, `@subscription/*` -> `src/subscription/`, etc.
- **Code style:** Double quotes, semicolons, 125 char width, 2-space indent
- **Testing:** Jest with `ts-jest`; test files in `test/` subdirectory per module
- **Frontend:** Feature-based modules, RTK Query for API data, Valibot for validation, `cn()` for conditional classes
- **Import type:** Use `import type` for type-only imports on frontend

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| luxon | 3.7.2 | Date/time manipulation (API) | Already in use in 42+ API files; immutable, timezone-aware |
| luxon | 3.7.2 | Date/time parsing/display (UI) | New dependency per D-11; same library both sides of stack |
| @types/luxon | 3.7.1 | TypeScript definitions for luxon | Required for TypeScript projects using luxon |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| luxon | date-fns | Locked decision -- luxon already adopted in API, must match |
| luxon | dayjs | Locked decision -- luxon is the project standard |

**Installation (trade-flow-ui only -- API already has luxon):**
```bash
npm install luxon
npm install --save-dev @types/luxon
```

**Version verification:**
- `luxon`: 3.7.2 (verified via `npm view luxon version`)
- `@types/luxon`: 3.7.1 (verified via `npm view @types/luxon version`)

## Architecture Patterns

### Established Date Conversion Pattern (from Quick Task 1)

The project already has a proven pattern for Luxon DateTime in DTOs. This phase extends it to remaining modules.

```
Entity (MongoDB)          Repository            DTO (Business Logic)       Response (HTTP)
─────────────────    ───────────────────    ────────────────────────    ─────────────────
Date (native JS)  →  DateTime.fromJSDate()  →  DateTime (Luxon)      →  ISO 8601 string
                     in toDto() method          used by services          via JSON.stringify
                                                                          (toJSON -> toISO)
```

### Pattern 1: Repository Date-to-DateTime Conversion
**What:** Convert native JS `Date` from MongoDB entities to Luxon `DateTime` in the repository `toDto()` method.
**When to use:** Every repository that maps entities containing date fields to DTOs.
**Example:**
```typescript
// Source: Established pattern in schedule.repository.ts and quote.repository.ts
import { DateTime } from "luxon";

private toDto(entity: ISubscriptionEntity): ISubscriptionDto {
  return {
    id: entity._id.toHexString(),
    userId: entity.userId,
    stripeSubscriptionId: entity.stripeSubscriptionId,
    stripeCustomerId: entity.stripeCustomerId,
    status: entity.status,
    currentPeriodEnd: entity.currentPeriodEnd
      ? DateTime.fromJSDate(entity.currentPeriodEnd, { zone: "utc" })
      : undefined,
    trialEnd: entity.trialEnd
      ? DateTime.fromJSDate(entity.trialEnd, { zone: "utc" })
      : undefined,
    canceledAt: entity.canceledAt
      ? DateTime.fromJSDate(entity.canceledAt, { zone: "utc" })
      : undefined,
    cancelAtPeriodEnd: entity.cancelAtPeriodEnd,
  };
}
```

### Pattern 2: Shared Conversion Utility (Recommended)
**What:** A reusable utility function to avoid duplicating `DateTime.fromJSDate()` with UTC zone in every repository.
**When to use:** When multiple repositories need the same Date-to-DateTime conversion.
**Example:**
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

### Pattern 3: Frontend Date Helpers Module
**What:** A centralized utility module for parsing API date strings and formatting for display.
**When to use:** All frontend components that display dates from API responses.
**Example:**
```typescript
// src/lib/date-helpers.ts
import { DateTime } from "luxon";

/** Parse an ISO 8601 string from the API into a Luxon DateTime */
export function parseApiDate(isoString: string): DateTime {
  return DateTime.fromISO(isoString, { zone: "utc" });
}

/** Format a date for display (e.g., "30 Mar 2026") */
export function formatDate(isoString: string): string {
  return DateTime.fromISO(isoString).toLocaleString(DateTime.DATE_MED);
}

/** Format a date with time (e.g., "30 Mar 2026, 10:00") */
export function formatDateTime(isoString: string): string {
  return DateTime.fromISO(isoString).toLocaleString(DateTime.DATETIME_MED);
}

/** Format relative time (e.g., "in 5 days", "3 hours ago") */
export function formatRelative(isoString: string): string | null {
  return DateTime.fromISO(isoString).toRelative();
}
```

### Pattern 4: ISubscriptionDto Alignment
**What:** Make `ISubscriptionDto` extend `IBaseResourceDto` and convert date fields.
**Example:**
```typescript
// src/subscription/data-transfer-objects/subscription.dto.ts
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
}
```

### Anti-Patterns to Avoid
- **Native Date in DTOs:** Never use `Date` in any DTO interface. The type system enforces Luxon DateTime.
- **Conversion in services:** Never convert Date-to-DateTime in services. The repository is the conversion boundary.
- **Conversion in controllers:** Never convert DateTime-to-string manually in controllers. Let `JSON.stringify()` call `toJSON()` which calls `toISO()` automatically.
- **Missing UTC zone:** Always pass `{ zone: "utc" }` to `DateTime.fromJSDate()`. Without it, Luxon uses the system local timezone, producing inconsistent results across environments.
- **Frontend `new Date()`:** Never use `new Date(isoString)` on the frontend. Always use `DateTime.fromISO()`.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Date-to-DateTime conversion | Inline `DateTime.fromJSDate(date, { zone: "utc" })` in every repo | Shared `toDateTime()` / `toOptionalDateTime()` utility | Eliminates duplication, ensures UTC zone is never forgotten |
| ISO 8601 serialization | Custom `@Transform()` decorators or manual `.toISO()` in controllers | Luxon's built-in `toJSON()` method (auto-called by `JSON.stringify()`) | DateTime.toJSON() already returns ISO 8601; NestJS uses JSON.stringify for responses |
| Frontend date formatting | Custom format functions with string manipulation | Luxon's `toLocaleString()`, `toRelative()`, `toFormat()` | Handles locales, timezones, edge cases automatically |
| Timezone handling | Manual UTC offset arithmetic | Luxon's `{ zone: "utc" }` option and `.toUTC()` method | Luxon handles DST transitions, leap seconds, etc. |

**Key insight:** Luxon's `DateTime.toJSON()` calling `toISO()` means the entire NestJS response pipeline works without any custom serialization. This was already proven in Quick Task 1.

## Common Pitfalls

### Pitfall 1: Forgetting `{ zone: "utc" }` on fromJSDate
**What goes wrong:** DateTime is created in the server's local timezone instead of UTC, causing inconsistent date values.
**Why it happens:** `DateTime.fromJSDate()` defaults to the system timezone.
**How to avoid:** Always pass `{ zone: "utc" }` as the second argument. The shared utility enforces this.
**Warning signs:** Dates appear shifted by hours in API responses; different results on local dev vs production.

### Pitfall 2: Breaking Consumers When Stripping createdAt/updatedAt
**What goes wrong:** Code that reads `dto.createdAt` from `ISubscriptionDto` gets a TypeScript error or runtime `undefined`.
**Why it happens:** D-04 strips these fields from the DTO, but consumers may still reference them.
**How to avoid:** Search all consumers of `ISubscriptionDto` before removing fields. Update subscription response mappers, controllers, webhook handlers, and frontend types.
**Warning signs:** TypeScript compilation errors after DTO change; `undefined` in API responses where dates were expected.

### Pitfall 3: Frontend Expects Date Objects but Gets ISO Strings
**What goes wrong:** Frontend code that used `new Date(response.someDate)` breaks or produces incorrect results when the API now returns ISO strings directly from Luxon DateTime.
**Why it happens:** Previously, if controllers returned `Date` objects, NestJS serialized them as ISO strings anyway -- but frontend code might have been doing `new Date()` parsing.
**How to avoid:** Replace all `new Date()` calls with `DateTime.fromISO()` on the frontend. The API has always returned ISO strings (JSON.stringify converts Date to ISO string), so the wire format does not actually change.
**Warning signs:** "Invalid Date" errors in the browser console; dates displaying as NaN.

### Pitfall 4: Optional DateTime Fields and Null Handling
**What goes wrong:** `DateTime.fromJSDate(null)` throws an error.
**Why it happens:** MongoDB documents may have `null` or `undefined` for optional date fields (e.g., `trialEnd`, `canceledAt`).
**How to avoid:** Always guard with a null/undefined check before conversion. The `toOptionalDateTime()` utility handles this.
**Warning signs:** Runtime crash in repository `toDto()` method; "Invalid argument" errors from Luxon.

### Pitfall 5: Test Mocks Need DateTime Values
**What goes wrong:** Existing test mocks that create DTOs with `new Date()` for date fields fail TypeScript compilation.
**Why it happens:** DTO interfaces now require `DateTime` not `Date`.
**How to avoid:** Update all mock generators and test fixtures to use `DateTime.now()` or `DateTime.fromISO("2026-01-01T00:00:00.000Z")`.
**Warning signs:** TypeScript errors in `test/mocks/` files; test suite won't compile.

### Pitfall 6: Subscription Webhook Handler Creates DTOs Directly
**What goes wrong:** The webhook event handler (BullMQ processor) may construct `ISubscriptionDto` objects directly with native `Date` values from Stripe event data.
**Why it happens:** Stripe webhook events provide Unix timestamps (numbers) that get converted to `Date` in existing code.
**How to avoid:** Update webhook handler to use `DateTime.fromSeconds(stripeTimestamp)` or route through the repository layer. Per D-08, the repository is the conversion boundary -- if the webhook handler writes directly through the repository, conversion happens there automatically.
**Warning signs:** Type errors in the webhook processor file; incorrect date values from Stripe timestamps.

## Code Examples

### Date-to-DateTime Utility
```typescript
// Source: Recommended new file based on established patterns
// src/core/utilities/to-date-time.utility.ts
import { DateTime } from "luxon";

export function toDateTime(date: Date): DateTime {
  return DateTime.fromJSDate(date, { zone: "utc" });
}

export function toOptionalDateTime(date: Date | undefined | null): DateTime | undefined {
  return date ? DateTime.fromJSDate(date, { zone: "utc" }) : undefined;
}
```

### Stripe Timestamp Conversion
```typescript
// For Stripe webhook data where dates come as Unix timestamps
import { DateTime } from "luxon";

// Stripe provides seconds-based Unix timestamps
const currentPeriodEnd = DateTime.fromSeconds(stripeSubscription.current_period_end, { zone: "utc" });
const trialEnd = stripeSubscription.trial_end
  ? DateTime.fromSeconds(stripeSubscription.trial_end, { zone: "utc" })
  : undefined;
```

### Frontend Date Display
```typescript
// Source: Recommended pattern for trade-flow-ui components
import { DateTime } from "luxon";

// Parse API response ISO string
const trialEnd = DateTime.fromISO(subscription.trialEnd);

// Display formatted
const display = trialEnd.toLocaleString(DateTime.DATE_MED); // "30 Mar 2026"

// Calculate days remaining
const daysLeft = trialEnd.diff(DateTime.now(), "days").days; // e.g., 15.3
const daysRemaining = Math.ceil(daysLeft); // 16
```

### Response Type Update (Frontend)
```typescript
// src/types/subscription.ts (or equivalent)
// API returns ISO strings -- frontend types reflect wire format
export interface SubscriptionResponse {
  id: string;
  userId: string;
  stripeSubscriptionId: string;
  stripeCustomerId: string;
  status: string;
  currentPeriodEnd?: string;  // ISO 8601 string from API
  trialEnd?: string;          // ISO 8601 string from API
  canceledAt?: string;        // ISO 8601 string from API
  cancelAtPeriodEnd: boolean;
}
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Native JS `Date` in DTOs | Luxon `DateTime` in DTOs | Quick Task 1 (2026-03-01) | 3 modules converted; subscription module is the remaining outlier |
| `createdAt`/`updatedAt` in some DTOs | Timestamps on entities only; DTOs add them individually as needed | This phase (D-01, D-02, D-04) | Cleaner separation of persistence vs business concerns |
| No Luxon on frontend | Luxon for all frontend date handling | This phase (D-11, D-12) | Consistent date library across full stack |

**Deprecated/outdated:**
- `new Date()` in DTOs: Replaced by `DateTime.now()` or `DateTime.fromJSDate()`
- `toJSDate()` in controllers for response serialization: No longer needed -- `JSON.stringify()` handles DateTime via `toJSON()`

## Discretion Recommendations

Based on research, here are recommendations for areas left to Claude's discretion:

### Shared Utility vs Inline Conversion
**Recommendation: Create shared utility.** A `toDateTime()` and `toOptionalDateTime()` utility in `src/core/utilities/` eliminates the risk of forgetting `{ zone: "utc" }` and reduces duplication across 5+ repositories. File naming: `to-date-time.utility.ts` following project conventions.

### Edge Cases: String vs Date Fields
**Recommendation: Handle both at the repository boundary.** Some MongoDB date fields may already be stored as ISO strings (D-10 says they should be). Use `DateTime.fromISO()` for string fields and `DateTime.fromJSDate()` for native Date fields. The utility can include a `toDateTimeFromAny(value: Date | string): DateTime` overload if needed, but start with explicit conversion and only add the polymorphic version if the codebase actually has mixed types.

### Frontend Integration Approach
**Recommendation: Date utility module.** Create `src/lib/date-helpers.ts` with focused helper functions (`parseApiDate`, `formatDate`, `formatDateTime`, `formatRelative`). Components import from this module. This avoids scattering `DateTime.fromISO()` calls across components and provides a single place to adjust formatting conventions.

### Ordering of Changes
**Recommendation: API first, then UI.** The API changes are structural (DTO interfaces, repository conversion) and must be correct before the frontend can consume the standardized responses. The wire format (ISO 8601 strings) does not actually change since `JSON.stringify(new Date())` and `JSON.stringify(DateTime)` both produce ISO strings -- but the API should be locked down first for confidence.

## Open Questions

1. **Which DTOs currently have createdAt/updatedAt that need review?**
   - What we know: `ISubscriptionDto` definitely has them and they must be stripped (D-04). Other DTOs may also have them.
   - What's unclear: Full inventory of all DTOs with timestamp fields across the API.
   - Recommendation: The planner should include a task to audit all DTOs for `createdAt`/`updatedAt` fields and determine which ones keep them (because the feature needs them) vs strip them.

2. **How many frontend components currently display dates?**
   - What we know: Subscription-related components (trial banner, billing tab) display dates. Schedule and quote features also show dates.
   - What's unclear: Exact count and locations of all date display/parsing in `trade-flow-ui`.
   - Recommendation: Implementer should grep for `new Date`, `toLocaleDateString`, `toLocaleString`, and similar patterns in the UI codebase.

3. **Do existing tests in the subscription module create DTOs with native Date?**
   - What we know: Quick Task 1 updated mocks in schedule, quote, and migration modules. Subscription module was added after (Phase 29-33).
   - What's unclear: Whether subscription test mocks use `Date` or already use appropriate types.
   - Recommendation: Check subscription test mocks during implementation and update as needed.

## Sources

### Primary (HIGH confidence)
- [Luxon API Documentation](https://moment.github.io/luxon/api-docs/index.html) - DateTime.fromJSDate(), toISO(), toJSON() methods
- [Luxon GitHub - toJSON behavior](https://github.com/moment/luxon/issues/249) - Confirmed toJSON() calls toISO()
- Quick Task 1 Summary (`.planning/quick/1-enforce-luxon-datetime-usage-in-dtos-for/1-SUMMARY.md`) - Established conversion patterns
- Phase 34 CONTEXT.md - User decisions D-01 through D-14
- Project CLAUDE.md - Conventions, architecture, stack details

### Secondary (MEDIUM confidence)
- [npm luxon](https://www.npmjs.com/package/luxon) - Version 3.7.2 confirmed
- [@types/luxon](https://www.npmjs.com/package/@types/luxon) - Version 3.7.1 confirmed
- [NestJS Serialization](https://docs.nestjs.com/techniques/serialization) - Response serialization approach

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH - Luxon already in use, version verified against npm registry
- Architecture: HIGH - Conversion pattern proven in Quick Task 1 across 3 modules
- Pitfalls: HIGH - Based on actual issues encountered in Quick Task 1 (see deviations section)

**Research date:** 2026-03-30
**Valid until:** 2026-04-30 (stable -- Luxon API is mature and rarely changes)
