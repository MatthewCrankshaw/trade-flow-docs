# Phase 34: Use Luxon DateTime in DTOs, Extend IBaseResourceDto, Move createdAt/updatedAt to Entity Only - Context

**Gathered:** 2026-03-30
**Status:** Ready for planning

<domain>
## Phase Boundary

Backend + frontend refactoring phase spanning both `trade-flow-api` and `trade-flow-ui`. Standardize date/time handling across the entire stack:

1. **Luxon DateTime as the DTO standard** — All date fields in DTOs use Luxon `DateTime`, never native JS `Date`. Repositories handle the `Date`→`DateTime` conversion at the entity→DTO boundary.

2. **IBaseResourceDto compliance** — All DTOs must extend `IBaseResourceDto` (`{ id: string }`). The outlier `ISubscriptionDto` gets aligned.

3. **createdAt/updatedAt cleanup** — These fields belong on entities (`IBaseEntity`) for database purposes. Strip them from DTOs that don't need them (e.g., `ISubscriptionDto`). Individual DTOs may add them as Luxon `DateTime` only when the feature requires exposing them.

4. **ISO 8601 UTC standard** — Dates stored in the database and sent in API responses use ISO 8601 strings in UTC format.

5. **Frontend Luxon adoption** — Add Luxon to `trade-flow-ui` and standardize all date parsing/display on the frontend.

6. **CLAUDE.md updates** — Document the Luxon DateTime and ISO 8601 UTC standards in both `trade-flow-api/CLAUDE.md` and `trade-flow-ui/CLAUDE.md`.

</domain>

<decisions>
## Implementation Decisions

### IBaseResourceDto Shape
- **D-01:** `IBaseResourceDto` stays as `{ id: string }` — no timestamps added to the base. It remains minimal.
- **D-02:** Individual DTOs add `createdAt`/`updatedAt` as Luxon `DateTime` only when the feature explicitly needs them. Most DTOs won't have timestamps.

### ISubscriptionDto Alignment
- **D-03:** `ISubscriptionDto` must extend `IBaseResourceDto` — aligning it with all other DTOs.
- **D-04:** Strip `createdAt` and `updatedAt` from `ISubscriptionDto` — they don't belong there. Entity keeps them for DB purposes.
- **D-05:** Convert all remaining date fields on `ISubscriptionDto` (e.g., `currentPeriodEnd`, `trialEnd`) from native `Date` to Luxon `DateTime`.
- **D-06:** Keep Stripe-specific field naming (e.g., `stripeSubscriptionId`, `stripeCustomerId`) — no renaming to project conventions. Only structural changes.

### No Native Date in DTOs
- **D-07:** No DTO may use native JS `Date`. All date fields in DTOs must be Luxon `DateTime`. This is a hard rule enforced project-wide.

### Conversion Boundary
- **D-08:** `Date`→`DateTime` conversion happens in the **repository layer**, at the entity→DTO mapping boundary. Same place where `ObjectId`→`string` conversion already occurs. Services work purely with DTOs/DateTime.

### API Response Serialization
- **D-09:** Dates in API responses serialize as ISO 8601 strings in UTC format (e.g., `"2026-03-30T10:00:00.000Z"`). Luxon's `DateTime.toISO()` or equivalent.
- **D-10:** Dates stored in MongoDB also use ISO 8601 UTC format for consistency.

### Frontend Luxon Adoption
- **D-11:** Add Luxon to `trade-flow-ui` as a dependency.
- **D-12:** Standardize all frontend date parsing and display to use Luxon `DateTime`. Replace any raw `new Date()` or string-based date handling.

### CLAUDE.md Documentation
- **D-13:** Update `trade-flow-api/CLAUDE.md` to document: Luxon DateTime as the DTO date standard, ISO 8601 UTC for storage and API responses, conversion happens in repository layer.
- **D-14:** Update `trade-flow-ui/CLAUDE.md` to document: Luxon DateTime for date parsing/display, ISO 8601 UTC format expected from API.

### Claude's Discretion
- Whether to create a shared `toDateTime(date: Date): DateTime` utility or inline the conversion in each repository
- How to handle edge cases where existing date fields are already strings vs Date objects
- Frontend Luxon integration approach (global helpers, per-component imports, or a date utility module)
- Ordering of changes (API first then UI, or parallel)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Core Interfaces
- `trade-flow-api/src/core/data-transfer-objects/base-resource.dto.ts` — `IBaseResourceDto` definition (currently `{ id: string }`)
- `trade-flow-api/src/core/entities/base.entity.ts` — `IBaseEntity` with `createdAt: Date`, `updatedAt: Date`

### Outlier DTO
- `trade-flow-api/src/subscription/data-transfer-objects/subscription.dto.ts` — `ISubscriptionDto` that needs alignment (extends nothing, uses native Date)

### Existing Luxon Patterns
- `trade-flow-api/src/schedule/data-transfer-objects/schedule.dto.ts` — Example of DTO using Luxon `DateTime`
- `trade-flow-api/src/quote/data-transfer-objects/quote.dto.ts` — Example with `updatedAt?: DateTime`
- `trade-flow-api/src/schedule/repositories/schedule.repository.ts` — Example of Date→DateTime conversion in repository layer

### Documentation Targets
- `trade-flow-api/CLAUDE.md` — Backend conventions to update
- `trade-flow-ui/CLAUDE.md` — Frontend conventions to update

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- Luxon `DateTime` is already imported in 42+ files across schedule, quote, quote-token, and migration modules
- `IBaseEntity` in `src/core/entities/base.entity.ts` already provides the entity-level timestamp pattern
- Repository layer already handles entity→DTO mapping (ObjectId→string conversion exists as a pattern)

### Established Patterns
- **DTO date fields:** Luxon `DateTime` (schedule, quote modules set the precedent)
- **Entity→DTO mapping:** Happens in repository layer, one-to-one field mapping
- **Import pattern:** `import { DateTime } from "luxon";`
- **All DTOs extend `IBaseResourceDto`** — except `ISubscriptionDto` which is the outlier

### Integration Points
- Every repository that maps entities to DTOs may need Date→DateTime conversion added
- `ISubscriptionDto` consumers (subscription service, controller, webhook handler) will see type changes
- Frontend components that display dates need Luxon adoption
- API response serialization pipeline — ensure DateTime fields serialize correctly

</code_context>

<specifics>
## Specific Ideas

- The user wants ISO 8601 UTC as the universal date format — both in the database and in API responses
- CLAUDE.md in both repos must be updated to document these standards so all future development follows them
- This is a standardization/consistency phase — not adding new features, but ensuring the entire codebase follows one date handling approach

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 34-use-luxon-datetime-in-dtos-extend-ibaseresourcedto-move-createdat-updatedat-to-entity-only*
*Context gathered: 2026-03-30*
