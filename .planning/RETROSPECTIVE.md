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

## Cross-Milestone Trends

### Process Evolution

| Milestone | Execution Time | Phases | Key Change |
|-----------|---------------|--------|------------|
| v1.0 | 1.5 hours | 8 | Established module-copy pattern, deferred v2 features cleanly |

### Top Lessons (Verified Across Milestones)

1. Copy-adapt existing module patterns for new domain entities (visit-type -> schedule)
2. Defer non-essential modes/features to keep milestones shippable
3. Smart defaults reduce form complexity and improve UX for solo operators
