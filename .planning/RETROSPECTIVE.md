# Project Retrospective

*A living document updated after each milestone. Lessons feed forward into future planning.*

## Milestone: v1.0 -- Scheduling

**Shipped:** 2026-03-07
**Phases:** 8 | **Plans:** 16 | **Execution time:** 1.5 hours

### What Was Built
- Visit type system: CRUD backend with default generation per trade, full management UI with color picker and status filtering
- Schedule module: data model, create/list/edit/cancel API with status state machine and structured filter parser
- Schedule UI: creation form with calendar, responsive list/detail views, edit mode, status transitions with confirmations
- Job detail integration: schedule-derived header hints and per-status badge breakdown replacing all mock data

### What Worked
- Consistent module pattern (visit-type -> schedule) made later phases faster -- avg plan time dropped from 10min (Phase 1) to 2min (Phase 8)
- Planning docs (CONTEXT.md, RESEARCH.md) gave each phase clear boundaries -- no scope creep during execution
- Two-repo architecture worked smoothly with coordinated feature work
- Smart defaults (1hr duration, auto-assignee) reduced required fields and simplified the creation form
- Deferring arrival window mode and conflict detection to v2 kept v1 focused and shippable

### What Was Inefficient
- ROADMAP.md plan checkboxes got out of sync with actual completion (some phases show unchecked plans that were completed) -- needs automated sync or skip manual tracking
- Some early phases had more verbose summaries than needed -- later phases were more concise

### Patterns Established
- Structured filter format `filter:<field>:<op>=value` as project-wide API standard
- STATUS_CONFIG record pattern for consistent badge rendering across views
- Job-scoped cache tags for RTK Query invalidation
- Luxon DateTime enforced in all DTOs for consistent date handling
- Split-file pattern for shadcn components to satisfy react-refresh lint rule

### Key Lessons
1. Following existing module patterns exactly (copy-adapt, don't reinvent) dramatically accelerates later phases
2. Separate transition service from updater service gives clean lifecycle management
3. ISO8601 single datetime field is cleaner than separate date + time fields
4. Cross-module validation in creator services catches errors early with specific error codes

### Cost Observations
- Model mix: primarily opus for planning and execution
- Total execution: 1.5 hours across 16 plans
- Notable: later phases completed 5x faster than early phases due to pattern reuse

---

## Milestone: v1.1 -- Item Tax Rate Linkage

**Shipped:** 2026-03-08
**Phases:** 2 | **Plans:** 6 | **Execution time:** ~1 hour

### What Was Built
- Item data layer migrated from numeric defaultTaxRate to taxRateId ObjectId reference (entity, DTO, repository, requests, response, mappers)
- Tax rate existence validation in both item creator and updater services with unit tests
- Onboarding flow reordered to create tax rates before items with ID threading
- Quote factories updated to resolve tax rate percentage from taxRateId via TaxRateRepository
- Tax rate dropdown on Material, Labour, and Fee item forms with "Name (rate%)" format and default pre-selection

### What Worked
- Phase 9 gap closure cycle worked well -- verifier caught missing updater validation and quote factory tests, gap plan addressed the critical one
- Cross-phase integration was clean -- Phase 10 UI consumed Phase 9 API contracts without surprises
- Client-side resolution pattern (RTK Query cache) avoided unnecessary server joins
- Following existing form patterns (FormSelect, useItemForm hook) made UI work straightforward
- Human verification checkpoint on item forms caught potential issues before completion

### What Was Inefficient
- Phase 9 had 4 plans for what was essentially 3 logical work items -- the gap closure plan (09-04) could have been avoided with better initial planning of the updater service validation
- SUMMARY.md frontmatter `requirements-completed` was inconsistent across plans -- some plans listed requirements, others didn't, making automated extraction unreliable
- ROADMAP.md plan checkboxes still not synced with completion (same issue as v1.0)

### Patterns Established
- ObjectId<->string conversion pattern for cross-entity references (taxRateId)
- Entity reference dropdown pattern: fetch via shared hook, filter active, map to {value, label}, render FormSelect
- Empty reference state: disabled control + inline warning with navigation link
- Tax rate validation pattern: both creator and updater validate existence before persisting

### Key Lessons
1. Cross-module validation (TaxRateRepository in ItemModule) is a clean pattern for referential integrity
2. Gap closure cycles add overhead -- investing in thorough initial plans (especially for update/delete operations) saves a planning+execution round
3. "Do not change" requirements (ITMUI-03) are valid constraints that can be verified by confirming zero references

### Cost Observations
- Model mix: primarily opus for execution, sonnet for verification
- Total execution: ~1 hour across 6 plans
- Notable: gap closure added ~30% overhead to Phase 9; without it, milestone would have been ~45 min

---

## Cross-Milestone Trends

### Process Evolution

| Milestone | Execution Time | Phases | Key Change |
|-----------|---------------|--------|------------|
| v1.0 | 1.5 hours | 8 | Established module-copy pattern, deferred v2 features cleanly |
| v1.1 | ~1 hour | 2 | Cross-module references, gap closure cycle, entity dropdown pattern |

### Top Lessons (Verified Across Milestones)

1. Copy-adapt existing module patterns for new domain entities (visit-type -> schedule -> item-tax-rate)
2. Defer non-essential modes/features to keep milestones shippable
3. Smart defaults reduce form complexity and improve UX for solo operators
4. Cross-module validation with specific error codes catches errors early (confirmed in both v1.0 schedules and v1.1 tax rates)
5. Plan update/delete operations explicitly from the start to avoid gap closure overhead
