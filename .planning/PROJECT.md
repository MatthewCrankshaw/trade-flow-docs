# Trade Flow

## What This Is

A business management application for sole tradespeople — plumbers, electricians, builders, and other independent contractors — that replaces scattered tools (paper notes, spreadsheets, generic invoicing apps, WhatsApp, calendar apps) with one streamlined system built around how trades actually work. Everything connects to the job: quotes, schedules, materials, labour, invoices, payments, and customer history.

Two independent codebases: `trade-flow-api` (NestJS/MongoDB) and `trade-flow-ui` (React/Vite), each with their own git repo, managed and deployed independently. Feature branches are created in both repos for coordinated feature work.

## Core Value

A job is the centre of the business — Trade Flow helps tradespeople run their entire business from first call to final payment in one simple, structured system.

## Requirements

### Validated

<!-- Shipped and confirmed valuable. Inferred from existing codebase. -->

- ✓ User can sign up and authenticate via Firebase — existing
- ✓ User can create and manage a business — existing
- ✓ User can create and manage customers — existing
- ✓ User can create and manage jobs (draft status, job types, descriptions) — existing
- ✓ User can manage inventory items — existing
- ✓ User can manage tax rates — existing
- ✓ User can create quotes on jobs — existing (basic display)
- ✓ Default job types and items generated per trade on business creation — existing
- ✓ Onboarding flow guides new users through setup — existing
- ✓ Email sending via SendGrid for verification and notifications — existing

### Active

<!-- Current scope: Job Scheduling milestone -->

- [ ] User can create schedule entries on a job with two modes: exact start time + duration, or arrival window (start/end) + duration
- [ ] User can view all schedule entries for a specific job
- [ ] User can edit existing schedule entries
- [ ] User can cancel/delete schedule entries
- [ ] User can set visit type on a schedule entry (generated per trade, similar to job types)
- [ ] Schedule entries have statuses: Scheduled, Confirmed, Completed, Canceled, No-show
- [ ] A job can have multiple schedule entries (assess, do work, follow-up)
- [ ] User sees a warning when creating a schedule that overlaps with an existing one
- [ ] User can add notes to a schedule entry
- [ ] Schedule assignee defaults to the logged-in user (solo operator)

### Out of Scope

<!-- Explicit boundaries for this milestone. -->

- Quote acceptance/rejection workflow — next milestone
- Invoice generation and management — next milestone
- Payment tracking — next milestone
- Standalone calendar or "Today" screen — future milestone (schedules show on job page only for now)
- Customer notifications for schedule entries — future
- Required items/materials on schedule entries — future
- Team member assignment on schedules — future (solo operator only for now)
- File/photo uploads on jobs — separate feature
- Job notes (non-schedule) — separate feature
- Data export/backup — future
- Audit logging — future

## Context

- **Brownfield project:** Existing codebase with established patterns (see `.planning/codebase/`)
- **Backend pattern:** Strict Controller → Service → Repository layering (NestJS)
- **Frontend pattern:** Feature-based modules with Redux RTK Query for server state
- **Auth:** Firebase JWT (RS256) with server-side public key validation
- **API contract:** Standardized response format: `{ data: T[], pagination?, errors? }`
- **Scheduling philosophy:** Tradespeople think in order of jobs, rough timing, buffer time, and customer expectations — NOT calendar precision or resource optimization. The system must support natural looseness (exact start times for morning jobs, "sometime after lunch" for afternoon ones)
- **Two scheduling modes:** Exact Start (start time + duration, auto-calc end) and Arrival Window (window start + window end + duration, optional "call before arrival" toggle for future)
- **Existing mock data:** `JobDetailTabs.tsx` contains MOCK_SCHEDULES — replace with real implementation
- **Visit types:** Similar concept to job types — generated per trade when a business is created

## Constraints

- **Tech stack:** Must follow existing NestJS (API) and React/Vite (UI) patterns — see `.planning/codebase/CONVENTIONS.md`
- **Two repos:** Changes span `trade-flow-api` and `trade-flow-ui` with coordinated feature branches
- **Solo operator:** Schedule assignee is always the logged-in user for now, but model the data to support team assignment later
- **No standalone calendar:** Schedules only appear within a job's detail page for this milestone

## Key Decisions

<!-- Decisions that constrain future work. -->

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Start Time + Duration (not Start + End) | Matches how tradespeople think — "I'll be there at 9, it'll take 2 hours" | — Pending |
| Two scheduling modes (Exact + Window) | Residential work uses arrival windows; commercial/repeat work uses exact times | — Pending |
| Warn on conflicts, don't block | Tradespeople sometimes intentionally double-book or stack jobs | — Pending |
| Schedules on job page only (no calendar) | Keep scope tight; "Today" screen is a future milestone | — Pending |
| Visit types generated per trade | Consistent with existing job types pattern; reduces setup friction | — Pending |
| Data model supports team assignment | Even though solo-only now, avoid costly migration later | — Pending |

---
*Last updated: 2026-02-21 after initialization*
