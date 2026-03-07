# Phase 1: Visit Type Backend - Context

**Gathered:** 2026-02-21
**Status:** Ready for planning

<domain>
## Phase Boundary

Visit types exist as a managed data type with CRUD API and sensible defaults generated per trade during onboarding. This phase delivers the backend data model, API endpoints, and default generation logic. UI for managing visit types is Phase 2.

</domain>

<decisions>
## Implementation Decisions

### Default visit types per trade
- 4 generic defaults for ALL trades (except "Other"): **Quote, Job, Follow-up, Inspection**
- "Other" trade gets no default visit types
- Defaults are not trade-specific — same 4 types regardless of trade
- Each default has a predefined, curated color + icon pairing (e.g., Quote = specific color + icon)

### Visit type data model
- Fields: name, description, business ID, color, icon
- No duration hint field — default schedule duration is hardcoded at 30 minutes for v1 (duration on visit type is a future enhancement)
- No isDefault/origin flag — defaults and custom types are treated identically once created
- No sort order field — no explicit ordering

### CRUD behavior rules
- **Soft delete** — mark as inactive/archived; existing schedules keep the reference, type hidden from new schedule creation
- **Name locked after creation** — description, color, and icon can be edited, but name is permanent
- **Unique names per business** — enforced; no duplicate visit type names within a business
- **No limit** on custom visit types per business
- **Must have at least one** — prevent deletion of the last active visit type for a business

### Generation trigger & lifecycle
- Defaults generated **during onboarding** (business creation) — they exist from the start
- No migration needed for existing businesses — not in production yet
- Pre-production: no backfill or lazy generation needed

### Claude's Discretion
- Exact color + icon assignments for the 4 defaults (should be visually distinct and intuitive)
- API endpoint structure and naming conventions (follow existing codebase patterns)
- Validation error message wording
- How soft-deleted types are stored/flagged in MongoDB

</decisions>

<specifics>
## Specific Ideas

- Visit types mirror the existing JobType pattern in the codebase
- Hardcoded 30-minute default duration lives at the schedule level, not the visit type level
- The fixed list of trades already exists in the system — "Other" is the only trade excluded from default generation

</specifics>

<deferred>
## Deferred Ideas

- Default duration per visit type (configurable on the type itself) — future enhancement
- Trade-specific visit type defaults (different types per trade) — future enhancement if needed

</deferred>

---

*Phase: 01-visit-type-backend*
*Context gathered: 2026-02-21*
