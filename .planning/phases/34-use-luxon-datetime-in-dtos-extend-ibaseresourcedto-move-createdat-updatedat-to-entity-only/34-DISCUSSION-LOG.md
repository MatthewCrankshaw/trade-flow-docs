# Phase 34: Use Luxon DateTime in DTOs, Extend IBaseResourceDto, Move createdAt/updatedAt to Entity Only - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-03-30
**Phase:** 34-use-luxon-datetime-in-dtos-extend-ibaseresourcedto-move-createdat-updatedat-to-entity-only
**Areas discussed:** IBaseResourceDto shape, SubscriptionDto alignment, Conversion boundary, API response serialization

---

## IBaseResourceDto Shape

### Q1: Should IBaseResourceDto include both createdAt and updatedAt?

| Option | Description | Selected |
|--------|-------------|----------|
| Both createdAt + updatedAt | All DTOs inherit both timestamps as Luxon DateTime | |
| Only createdAt | updatedAt stays per-DTO where needed | |
| Both, but updatedAt optional | createdAt required, updatedAt optional | |

**User's choice:** Other — createdAt and updatedAt are NOT needed on IBaseResourceDto. Most DTOs don't need those fields. If they do, they can manually add them to the child DTO interface.

### Q2: What should be added to IBaseResourceDto if not timestamps?

| Option | Description | Selected |
|--------|-------------|----------|
| Nothing — keep as-is | IBaseResourceDto stays as { id: string } | |
| Add businessId | Most DTOs already have businessId: string | |
| Add businessId + createdAt | businessId for scoping, createdAt as read-only timestamp | |

**User's choice:** Other — "extend IBaseResourceDto" means making ISubscriptionDto (and any other outliers) properly extend it, not adding new fields to the base.

### Q3: Should createdAt/updatedAt be removed or converted on ISubscriptionDto?

| Option | Description | Selected |
|--------|-------------|----------|
| Remove from SubscriptionDto | Strip createdAt/updatedAt. Entity keeps them for DB. | ✓ |
| Convert to Luxon DateTime | Keep them but switch from Date to DateTime | |
| You decide | Claude evaluates | |

**User's choice:** Remove from SubscriptionDto. Convert any remaining date fields on the DTO to Luxon DateTime instead of JS Date.

---

## SubscriptionDto Alignment

### Q4: Field naming when SubscriptionDto extends IBaseResourceDto?

| Option | Description | Selected |
|--------|-------------|----------|
| Keep Stripe naming | stripeSubscriptionId etc. stay as-is. Only structural changes. | ✓ |
| Rename to project conventions | e.g., externalSubscriptionId. More consistent but more churn. | |
| You decide | Claude evaluates | |

**User's choice:** Keep Stripe naming (Recommended)

---

## Conversion Boundary

### Q5: Where should Date→DateTime conversion happen?

| Option | Description | Selected |
|--------|-------------|----------|
| Repository layer | Repos already map entities to DTOs. Same place as ObjectId→string. | ✓ |
| Shared utility function | Reusable toDateTime(date) helper callable anywhere | |
| You decide | Claude picks based on codebase patterns | |

**User's choice:** Repository layer (Recommended)

---

## API Response Serialization

### Q6: How should Luxon DateTime fields serialize in JSON responses?

| Option | Description | Selected |
|--------|-------------|----------|
| ISO 8601 strings | DateTime.toISO() → industry standard | |
| Unix timestamps | DateTime.toSeconds() → compact, timezone-neutral | |
| You decide | Claude picks based on frontend expectations | |

**User's choice:** Other — Dates when saved to the database and sent in API responses should be ISO 8601 strings in UTC format.

### Q7: Does the frontend need changes?

| Option | Description | Selected |
|--------|-------------|----------|
| Backend only — no frontend changes | API responses already send ISO strings | |
| Frontend should also use Luxon | Add Luxon to trade-flow-ui, standardize date handling | ✓ |
| You decide | Claude checks current frontend date handling | |

**User's choice:** Frontend should also use Luxon

### Additional: CLAUDE.md updates

**User's note:** Both trade-flow-api/CLAUDE.md and trade-flow-ui/CLAUDE.md need to be updated to reflect these date/time standards.

---

## Claude's Discretion

- Shared toDateTime utility vs inline conversion
- Edge case handling for existing date formats
- Frontend Luxon integration approach
- Change ordering (API first vs parallel)

## Deferred Ideas

None — discussion stayed within phase scope
