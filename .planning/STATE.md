---
gsd_state_version: 1.0
milestone: v1.2
milestone_name: Bundles & Quotes
status: completed
stopped_at: Completed 14-04-PLAN.md -- UI polish and bundle pricing strategy
last_updated: "2026-03-14T20:39:02.862Z"
last_activity: 2026-03-14 -- Completed 14-04 UI polish and bundle pricing strategy selector
progress:
  total_phases: 4
  completed_phases: 4
  total_plans: 10
  completed_plans: 10
  percent: 100
---

# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2026-03-08)

**Core value:** A job is the centre of the business -- Trade Flow helps tradespeople run their entire business from first call to final payment
**Current focus:** v1.2 Bundles & Quotes -- Phase 14 in progress (4 of 4 phases)

## Current Position

Phase: 14 of 14 (Quote Detail and Line Items) -- 4 of 4 in v1.2
Plan: 4 of 4 in current phase
Status: Phase 14 complete -- all gap closure plans applied
Last activity: 2026-03-14 -- Completed 14-04 UI polish and bundle pricing strategy selector

Progress: [██████████] 100%

## Performance Metrics

**Velocity (cumulative):**
- Total plans completed: 30 (16 v1.0 + 6 v1.1 + 8 v1.2)
- Average duration: 6min
- Total execution time: ~2.5 hours

**By Milestone:**

| Milestone | Phases | Plans | Execution Time | Avg/Plan |
|-----------|--------|-------|---------------|----------|
| v1.0 Scheduling | 8 | 16 | 1.5 hours | 6min |
| v1.1 Item Tax Rate Linkage | 2 | 6 | ~1 hour | 4min |
| v1.2 Bundles & Quotes | 4 | 10 | 23min | 3min |
| Phase 12 P01 | 4min | 2 tasks | 6 files |
| Phase 12 P02 | 4min | 3 tasks | 3 files |
| Phase 13 P01 | 6min | 3 tasks | 13 files |
| Phase 13 P02 | 3min | 3 tasks | 11 files |
| Phase 13 P03 | 7min | 2 tasks | 7 files |
| Phase 14 P01 | 5min | 2 tasks | 9 files |
| Phase 14 P02 | 3min | 2 tasks | 5 files |
| Phase 14 P03 | 3min | 2 tasks | 6 files |
| Phase 14 P04 | 3min | 3 tasks | 8 files |

## Accumulated Context

### Decisions

Key decisions archived in PROJECT.md Key Decisions table.

- 11-01: Items pre-selected via SearchableItemPicker eliminates empty itemId intermediate state
- 11-01: SearchableItemPicker placed in src/components/ for cross-feature reuse
- [Phase 12]: Reused CreateBundleComponentRequest for update validation (same shape, same rules)
- [Phase 12]: Always validate bundle components on update, consistent with taxRateId always-validate pattern
- 12-02: Reused typeColors/typeLabels mappings across all three component views for consistency
- 12-02: No cost/price shown in component rows or header per user design decision
- 13-01: Atomic quote_counters collection with findOneAndUpdate/$inc for sequential Q-YYYY-NNN numbering
- 13-01: Denormalized customerName/jobTitle in API response to avoid N+1 on UI
- 13-01: updatedAt vs sentAt comparison for modified-since-sent indicator (no extra boolean)
- 13-02: Used date-fns (not Luxon) for quote date formatting -- consistent with project convention
- 13-02: formatDecimal for quote totals since API returns major units via toMajorUnits()
- [Phase 13-03]: Inner form component pattern for dialog state reset (avoids setState-in-effect lint error)
- [Phase 13-03]: AlertDialog confirmations only for irreversible transitions (Accept, Reject); Send is immediate
- 14-01: Totals not persisted in quote entity -- recalculated from line items on every read
- 14-01: Bundle component deletion cascades from parent -- cannot delete components individually
- 14-01: Line item update recalculates lineTotal and discountAmount server-side
- 14-02: Inline edit uses defaultValue with key={id+value} pattern for server-driven input reset
- 14-02: No delete confirmation on line items -- immediate delete, user can re-add
- 14-02: Bundle rows show dashes for unit price and tax columns (details on component rows)
- 14-03: Pure numeric percentageOf calculation avoids Dinero integer division precision loss
- 14-03: Soft delete via DELETED status enum rather than hard delete or deletedAt timestamp
- 14-03: Added updateMany to MongoDbWriter for batch soft-delete of bundle child line items
- 14-04: CSS grid for bundle component alignment instead of flex layout
- 14-04: Two-step add flow for bundles overrides locked immediate-add decision (per UAT gap diagnosis)
- 14-04: priceStrategy override uses spread to merge with bundleConfig

### Pending Todos

None.

### Blockers/Concerns

- Research flags Phase 12 (bundle component editing): BundleItemForm submit handler and mergeExistingItemWithChanges utility need careful tracing during planning
- Research flags Phase 14 (quote detail): parentLineItemId pipeline and Money serialization format need verification during planning

### Quick Tasks Completed

| # | Description | Date | Commit | Directory |
|---|-------------|------|--------|-----------|
| 1 | Enforce Luxon DateTime usage in DTOs for dates and durations in trade-flow-api | 2026-03-01 | 67894c5 | [1-enforce-luxon-datetime-usage-in-dtos-for](./quick/1-enforce-luxon-datetime-usage-in-dtos-for/) |
| 2 | Merge schedule date and startTime into startDateTime | 2026-03-01 | 0b2933f | [2-merge-schedule-date-and-starttime-into-a](./quick/2-merge-schedule-date-and-starttime-into-a/) |

## Session Continuity

Last session: 2026-03-14T20:33:39Z
Stopped at: Completed 14-04-PLAN.md -- UI polish and bundle pricing strategy
Resume file: None -- all gap closure plans complete
