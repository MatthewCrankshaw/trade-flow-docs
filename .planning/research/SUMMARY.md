# Research Summary: Job Scheduling for Trade Flow

**Research Date:** 2026-02-21

## Key Findings

### Stack
No new libraries needed. The scheduling feature is straightforward CRUD that fits the existing NestJS + Mongoose + RTK Query patterns perfectly. Consider `date-fns` for date manipulation. Avoid @nestjs/schedule (that's for cron jobs), calendar libraries (no calendar view this milestone), and embedded MongoDB subdocuments (use a separate collection).

### Table Stakes Features
- Create/edit/cancel schedule entries on jobs
- Date + time + duration
- View all visits for a job
- Visit status tracking (scheduled → confirmed → completed/canceled/no-show)
- Conflict visibility (warn, don't block)

### Differentiators
- **Two scheduling modes** (exact start + arrival window) — matches how trades actually communicate with customers. Most competitors only support exact times.
- **Visit types per trade** — generated on business creation, mirrors existing JobType pattern. Low cost, high value.
- **Per-visit notes** — capture what happened on each visit, not just the job overall.

### Architecture
- Separate Schedule collection in MongoDB (not embedded in Job)
- Schedule module follows existing Controller → Service → Repository pattern
- Visit Type module mirrors JobType exactly
- ConflictChecker service for advisory overlap warnings
- Frontend: new `features/schedules/` directory with RTK Query endpoints

### Build Order
1. Visit Type backend (mirrors JobType)
2. Schedule backend (CRUD + status management)
3. Conflict detection (advisory warnings)
4. Default visit types (add to business creation)
5. Schedule UI (list + create dialog with mode selector)
6. Schedule UI (edit + status transitions)
7. Conflict warning UI
8. Integration (replace MOCK_SCHEDULES in JobDetailTabs)

### Watch Out For
1. **Over-engineering** — Keep it flat CRUD, no availability systems or slot-based logic
2. **Slow creation UX** — Must be under 60 seconds. Smart defaults, minimal required fields
3. **Timezone bugs** — Store UTC, convert in UI only, test near midnight
4. **Conflict blocking** — Must warn, never block. Trades intentionally overlap.
5. **Two-mode UI confusion** — Clear mode toggle, show only relevant fields per mode
6. **Mock data cleanup** — Explicitly remove MOCK_SCHEDULES from JobDetailTabs

## Recommendations for Roadmap

- **Phase structure should follow the build order** — backend first, UI second, integration last
- **Visit types are a prerequisite** — build them first since schedules reference them
- **Conflict detection is a separate concern** — can be added after basic CRUD works
- **UI should be iterated** — start with functional, optimize for speed after

---
*Summary compiled: 2026-02-21*
