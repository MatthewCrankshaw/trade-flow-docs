# Architecture Research: Job Scheduling Module

**Research Date:** 2026-02-21
**Domain:** Scheduling/visit management module integration
**Existing Stack:** NestJS 11 (Controller → Service → Repository), React 19 (RTK Query), MongoDB

## Component Boundaries

### Backend Components

```
┌─────────────────────────────────────────────────┐
│ Schedule Module                                   │
│                                                   │
│  ┌──────────────┐  ┌──────────────────────────┐ │
│  │ Controller    │  │ Services                  │ │
│  │              │  │  ├─ ScheduleCreator       │ │
│  │ /v1/schedules│──│  ├─ ScheduleRetriever     │ │
│  │              │  │  ├─ ScheduleUpdater       │ │
│  └──────────────┘  │  └─ ConflictChecker       │ │
│                     └──────────────────────────┘ │
│                              │                    │
│                     ┌────────┴─────────┐         │
│                     │ Repository        │         │
│                     │ ScheduleRepo      │         │
│                     └──────────────────┘         │
│                                                   │
│  ┌──────────────┐  ┌──────────────────────────┐ │
│  │ Schema        │  │ DTOs                      │ │
│  │ schedule.     │  │  ├─ CreateScheduleDto    │ │
│  │ schema.ts     │  │  ├─ UpdateScheduleDto    │ │
│  └──────────────┘  │  └─ ScheduleResponseDto   │ │
│                     └──────────────────────────┘ │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│ Visit Type Module (mirrors JobType)               │
│  ├─ Controller                                    │
│  ├─ Services (Creator, Retriever)                │
│  ├─ Repository                                    │
│  └─ Schema                                        │
└─────────────────────────────────────────────────┘
```

### Frontend Components

```
┌─────────────────────────────────────────────────┐
│ features/schedules/                               │
│                                                   │
│  ├─ api/                                          │
│  │   └─ scheduleApi.ts (RTK Query endpoints)     │
│  │                                                │
│  ├─ components/                                   │
│  │   ├─ ScheduleList.tsx                          │
│  │   ├─ CreateScheduleDialog.tsx                  │
│  │   ├─ EditScheduleDialog.tsx                    │
│  │   ├─ ScheduleCard.tsx                          │
│  │   ├─ ScheduleModeSelector.tsx                  │
│  │   └─ ConflictWarning.tsx                       │
│  │                                                │
│  ├─ types/                                        │
│  │   └─ schedule.types.ts                         │
│  │                                                │
│  └─ hooks/ (if needed)                            │
│       └─ useScheduleConflicts.ts                  │
└─────────────────────────────────────────────────┘
```

## Data Model

### Schedule Entity

```typescript
{
  _id: ObjectId,
  businessId: ObjectId,       // Business this belongs to
  jobId: ObjectId,            // Required — every schedule links to a job
  userId: ObjectId,           // Who is assigned (defaults to logged-in user)

  // Scheduling mode
  schedulingMode: 'exact' | 'window',

  // Exact mode fields
  startTime: Date,            // When it starts (exact mode) or window start (window mode)
  duration: Number,           // Duration in minutes
  endTime: Date,              // Calculated: startTime + duration

  // Window mode additional fields
  windowEnd: Date | null,     // End of arrival window (window mode only)

  // Classification
  visitTypeId: ObjectId,      // Type of visit (links to VisitType)
  status: String,             // scheduled | confirmed | completed | canceled | no-show

  // Content
  notes: String,              // Free-text notes

  // Metadata
  createdAt: Date,
  updatedAt: Date,
  createdBy: ObjectId,
}
```

### Status State Machine

```
               ┌──────────┐
               │ scheduled │
               └─────┬────┘
                     │
              ┌──────┼───────┐
              ▼      ▼       ▼
         confirmed  canceled  no-show
              │
              ▼
          completed
```

Valid transitions:
- `scheduled → confirmed` (customer confirms)
- `scheduled → canceled` (user or customer cancels)
- `scheduled → no-show` (customer didn't show / wasn't home)
- `confirmed → completed` (work done)
- `confirmed → canceled` (cancellation after confirmation)

## Data Flow

### Create Schedule

```
UI: CreateScheduleDialog
  → RTK Query mutation (POST /v1/schedules)
    → ScheduleController.create()
      → ConflictChecker.check(userId, startTime, duration)
        → ScheduleRepository.findOverlapping()
        → Returns: { hasConflict: boolean, conflictingSchedules: [] }
      → ScheduleCreator.create(dto, conflicts)
        → ScheduleRepository.create()
      → Response: { data: [schedule], warnings?: [conflicts] }
    → UI shows ConflictWarning if warnings present
    → ScheduleList refreshes via cache invalidation
```

### Conflict Detection

```
ConflictChecker.check(userId, date, startTime, duration):
  1. Calculate time range: [startTime, startTime + duration]
  2. Query: Find all schedules for userId on date where:
     - For exact mode: existing.startTime < newEnd AND existing.endTime > newStart
     - For window mode: use window boundaries for overlap check
  3. Exclude canceled/no-show schedules from conflicts
  4. Return overlapping schedules (if any) as warnings
```

## Integration Points

### With Existing Job Module
- Schedule entries reference `jobId`
- Job detail page gains a "Schedule" tab (replaces MOCK_SCHEDULES)
- Job status could optionally reflect scheduling state (future consideration)

### With Existing Auth/Business Module
- Schedule entries scoped to `businessId` (authorization)
- Uses existing JwtAuthGuard and authorization patterns
- Follows existing AuthorizedCreatorFactory pattern

### With Visit Types (New)
- Mirrors JobType pattern exactly
- Default visit types generated per trade on business creation
- Stored in separate collection, referenced by Schedule

## Suggested Build Order

Based on dependencies and the existing codebase patterns:

1. **Visit Type backend** — Schema, repository, service, controller (mirrors JobType, low risk)
2. **Schedule backend** — Schema, repository, services (Creator/Retriever/Updater), controller
3. **Conflict detection** — ConflictChecker service, integrated into create/update flow
4. **Default visit types** — Add to business creation flow (mirrors default job types)
5. **Schedule UI — List + Create** — ScheduleList, CreateScheduleDialog, ScheduleModeSelector
6. **Schedule UI — Edit + Status** — EditScheduleDialog, status transitions
7. **Conflict warning UI** — ConflictWarning component
8. **Integration** — Replace MOCK_SCHEDULES in JobDetailTabs, wire up RTK Query

### Why This Order
- Backend first: establishes API contract for UI development
- Visit types first: simple mirror of existing pattern, builds confidence
- Conflict detection after basic CRUD: layered complexity
- UI follows API: can develop against real endpoints
- Integration last: everything tested independently first

---
*Research completed: 2026-02-21*
