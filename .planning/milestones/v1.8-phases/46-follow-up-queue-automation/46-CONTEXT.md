# Phase 46: Follow-up Queue & Automation - Context

**Gathered:** 2026-04-12
**Status:** Ready for planning

<domain>
## Phase Boundary

Sending an estimate automatically schedules 3/10/21-day follow-up emails and a 30-day expiry job via BullMQ delayed jobs on a new `ESTIMATE_FOLLOWUPS` queue. Follow-ups fire reliably across worker restarts (Redis AOF persistence enforced at startup), cancel cleanly on any exit transition (customer response, revision, conversion, deletion, mark-as-lost), and auto-expire estimates 30 days after send.

Ships in this phase:
- New `ESTIMATE_FOLLOWUPS` queue constant in `src/queue/queue.constant.ts`.
- `EstimateFollowupScheduler` service: enqueues 3 delayed follow-up jobs (72h, 240h, 504h) + 1 expiry job (720h) with deterministic jobIds. Called from Phase 44's `EstimateEmailSender` as a new step after send completes.
- `EstimateFollowupProcessor` (BullMQ worker processor): defence-in-depth status re-read before sending each follow-up, silent no-op if estimate is no longer in a follow-up-worthy status. Appends rows to `estimate_email_sends` collection (Phase 44 created this) with types `FOLLOWUP_3D`/`FOLLOWUP_10D`/`FOLLOWUP_21D`.
- `BullMQEstimateFollowupCanceller` service: implements `IEstimateFollowupCanceller` (Phase 42 wired the interface + `NoopEstimateFollowupCanceller` default binding). Cancels all pending follow-ups + expiry job for a given estimate/revision via deterministic jobId removal.
- `EstimateFollowupsModule` that rebinds `ESTIMATE_FOLLOWUP_CANCELLER` token from `NoopEstimateFollowupCanceller` to `BullMQEstimateFollowupCanceller`.
- `EstimateExpiryProcessor`: processes the 30-day expiry delayed job, transitions estimate to `Expired` status via `EstimateTransitionService`.
- New shared Maizzle template `estimate-followup.html` in `trade-flow-api/src/email/templates/` — brief reminder with estimate number, job title, and View Estimate link plus all three CTAs.
- Amendment to Phase 44's `EstimateEmailSender`: inject `EstimateFollowupScheduler` and call `scheduleFollowups()` after the send lifecycle completes (new step 9 in D-AUDIT-08 ordering). On revised-send path, the existing `cancelAllFollowups` call (step 8) cancels old-revision follow-ups before scheduling new ones.
- Redis AOF runtime check: worker startup validates `CONFIG GET appendonly === "yes"` before registering the `ESTIMATE_FOLLOWUPS` processor. Fatal log + processor skip if not enabled.
- Docker Compose Redis configuration updated with `appendonly yes` / `appendfsync everysec` for local development.
- `WorkerModule` updated to register `EstimateFollowupProcessor` and `EstimateExpiryProcessor`.

**Out of scope (explicitly deferred):**
- Trader-facing UI for viewing or managing scheduled follow-ups (no visibility — invisible infrastructure).
- Configurable follow-up timing or delays (fixed 3/10/21-day schedule).
- Trader notification on auto-expiry (silent transition).
- Phase 45's public customer page response handling (Phase 45 scope).
- Phase 47's convert-to-quote / mark-as-lost (Phase 47 scope, but both call `cancelAllFollowups` on exit).

</domain>

<decisions>
## Implementation Decisions

### Follow-up Email Content (D-EMAIL)

- **D-EMAIL-01:** All three follow-up steps (3d/10d/21d) use the **same friendly, professional tone**. No escalating urgency, no "last chance" language. The tradesperson's brand stays warm and consistent across all touchpoints.
- **D-EMAIL-02:** Follow-up emails are **brief reminders with a link**, not full estimate summaries. Subject references the estimate number; body has a short message and a "View Estimate" button linking to the public page. Keeps emails short and drives clicks to the full estimate.
- **D-EMAIL-03:** **One shared Maizzle template** `estimate-followup.html` with a template variable for the step (e.g. `followupStep: "3d" | "10d" | "21d"`). The variable is available for minor copy variations (e.g. "Just checking in" vs "Following up") but the visual layout and tone are identical across steps.
- **D-EMAIL-04:** All three CTAs appear in every follow-up email, matching the initial send:
  - **Primary CTA:** "Go Ahead" or "Happy to Proceed" — natural language that UK homeowners actually use.
  - **Secondary CTA:** "Message [Tradesperson First Name]" — covers questions, revision requests, and site visit requests through one low-friction channel.
  - **Tertiary:** "Not right for me" as a text link — softer than "Decline."
  - **Disclaimer text below primary CTA:** "This is an estimate — the final cost may differ from this approximate figure."
- **D-EMAIL-05:** Follow-up emails use the **same document-token link** as the initial send (token references `rootEstimateId`, not per-revision id — per Phase 44 D-TKN-01). No new token generation for follow-ups.

### Expiry Behaviour (D-EXP)

- **D-EXP-01:** Auto-expiry at 30 days is a **silent transition**. No email notification to the trader. The estimate status flips to `Expired` in the database; the trader notices next time they view the estimate list or detail page.
- **D-EXP-02:** Customer sees a **friendly expired message** on the public estimate page after expiry: "This estimate has expired. Please contact [Business Name] if you're still interested." No response buttons. Consistent with RESP-07 terminal-state behaviour. (Phase 45 implements the customer page rendering; Phase 46 only transitions the status.)
- **D-EXP-03:** Expiry uses a **per-estimate delayed BullMQ job** at 720h (30 days), not a periodic sweep. Deterministic jobId pattern: `estimate-expiry:{estimateId}:{revisionNumber}`. Cancelled alongside follow-ups on any exit transition via `cancelAllFollowups`.
- **D-EXP-04:** The `IEstimateFollowupCanceller.cancelAllFollowups(estimateId, revisionNumber)` implementation must cancel **both** follow-up jobs (3 jobIds) **and** the expiry job (1 jobId) — 4 removals total per cancellation call. The `cancelAllFollowupsForChain(rootEstimateId)` variant cancels across all revisions in the chain.

### Scheduling Trigger (D-SCHED)

- **D-SCHED-01:** Follow-ups are scheduled **immediately on send** — the moment `EstimateEmailSender` completes. No configurable delay, no opt-out, no settings. Matches SC #1: "Sending an estimate automatically schedules follow-ups."
- **D-SCHED-02:** The scheduling call lives **inside `EstimateEmailSender`** (Phase 44's service). `EstimateFollowupScheduler` is injected and `scheduleFollowups()` is called as step 9 in the D-AUDIT-08 send lifecycle (after audit row, transition, token extend). This matches the existing pattern where `cancelAllFollowups` is also called from within `EstimateEmailSender`.
- **D-SCHED-03:** **No trader-facing UI** for follow-ups. Follow-ups are invisible infrastructure. No "next follow-up in 3 days" indicator on the estimate detail page. No management dashboard. Zero UI in Phase 46.
- **D-SCHED-04:** On the **pure re-send path** (same revision, status Sent/Viewed/Responded/SiteVisitRequested → Sent), follow-ups are NOT re-scheduled. The existing follow-up schedule from the original send continues on its original timeline. This matches Phase 44 D-TKN-06: "no follow-up cancellation happens" on pure re-send.
- **D-SCHED-05:** On the **revised-send path** (new revision, Draft → Sent), Phase 44's `EstimateEmailSender` first calls `cancelAllFollowups(oldRevisionId, oldRevisionNumber)` (step 8 — already wired), then calls `scheduleFollowups(newRevisionId, newRevisionNumber)` (step 9 — Phase 46 adds this). Fresh 3/10/21/30-day schedule starts from the new send time.
- **D-SCHED-06:** `scheduleFollowups()` uses deterministic jobIds (`estimate-followup:{estimateId}:{revisionNumber}:{step}`) per SC #1. A second call for the same estimate/revision is a silent no-op (BullMQ deduplicates by jobId).

### Redis AOF Infrastructure Gate (D-AOF)

- **D-AOF-01:** **Runtime startup check**: Worker service checks `CONFIG GET appendonly` on Redis connection at startup. If the value is not `"yes"`, log at `AppLogger.error` (FATAL level) and **refuse to register** the `ESTIMATE_FOLLOWUPS` processor. The worker still starts (other processors like `STRIPE_WEBHOOKS` remain functional), but follow-up scheduling will fail with a clear error when `EstimateFollowupScheduler` attempts to enqueue.
- **D-AOF-02:** Docker Compose Redis configuration is updated with `appendonly yes` and `appendfsync everysec` for local development parity with production.
- **D-AOF-03:** The SC #4 smoke test ("schedule a 60-second delayed job, restart Redis, confirm it fires") is a **manual smoke procedure** documented in the plan's verification section. Steps: enqueue a short-delay test job via the existing `POST /v1/queue/test-echo` pattern, `docker restart redis`, wait for delay to elapse, assert job processed in worker logs. Not an automated CI test — Redis restart is infrastructure-level.
- **D-AOF-04:** Railway production Redis must have AOF enabled before Phase 46 deploys. This is verified by the runtime startup check (D-AOF-01) — if misconfigured, the worker logs a fatal error and follow-ups don't process, making the issue immediately visible in deployment logs.

### Claude's Discretion

- Exact copy variations between 3d/10d/21d follow-up steps (within the same friendly tone)
- `EstimateFollowupsModule` internal structure and provider wiring
- Whether `EstimateExpiryProcessor` is a separate processor class or handled as a special case within `EstimateFollowupProcessor`
- Error handling strategy for failed follow-up sends (retry policy, dead letter queue)
- Exact order of operations for the AOF startup check within worker bootstrap

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Phase Dependencies
- `.planning/phases/42-revisions/42-CONTEXT.md` §D-HOOK — `IEstimateFollowupCanceller` interface, `NoopEstimateFollowupCanceller`, DI token `ESTIMATE_FOLLOWUP_CANCELLER`, wire diagram for module rebinding
- `.planning/phases/44-email-send-flow/44-CONTEXT.md` §D-AUDIT — `estimate_email_sends` collection schema, `EstimateEmailSendType` enum (defines `FOLLOWUP_3D`/`FOLLOWUP_10D`/`FOLLOWUP_21D`), write path ordering in `EstimateEmailSender.send()` (D-AUDIT-08)
- `.planning/phases/44-email-send-flow/44-CONTEXT.md` §D-TKN — Token lookup/reuse/revision handling, pure re-send vs revised-send path detection (D-TKN-06, D-TKN-07, D-TKN-08)
- `.planning/phases/41-estimate-module-crud-backend/41-CONTEXT.md` §D-TXN — `EstimateTransitionService`, `ALLOWED_TRANSITIONS` map, `EstimateStatus` enum values

### Requirements
- `.planning/REQUIREMENTS.md` — FUP-01 through FUP-08 (follow-up automation requirements)
- `.planning/ROADMAP.md` §Phase 46 — Goal, success criteria, infra gate

### Infrastructure Precedent
- `.planning/RETROSPECTIVE.md` §v1.4 — BullMQ worker pattern, `QUEUE_NAMES` constant, `src/worker.ts` entry point, `WorkerModule`, processor pattern, IORedis type workaround
- `.planning/RETROSPECTIVE.md` §v1.6 — STRIPE_WEBHOOKS queue + processor precedent, deterministic jobId for deduplication

### Codebase
- `.planning/codebase/ARCHITECTURE.md` — Backend layered architecture, module organization
- `.planning/codebase/CONVENTIONS.md` — File naming, class naming, error handling patterns

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- **BullMQ worker infrastructure** (v1.4): `src/worker.ts`, `WorkerModule`, `QUEUE_NAMES` constant, processor pattern — Phase 46 follows the same pattern for `ESTIMATE_FOLLOWUPS` queue
- **STRIPE_WEBHOOKS processor** (v1.6): precedent for BullMQ processor with deterministic jobId deduplication and event-type dispatch — Phase 46 mirrors for follow-up step dispatch
- **`EstimateEmailSendType` enum** (Phase 44): already defines `FOLLOWUP_3D`, `FOLLOWUP_10D`, `FOLLOWUP_21D` values — Phase 46 only consumes them
- **`estimate_email_sends` collection** (Phase 44): already exists with indexes — Phase 46 appends follow-up rows via `EstimateEmailSendCreator`
- **`IEstimateFollowupCanceller` interface** (Phase 42): already declared with `cancelAllFollowups` and `cancelAllFollowupsForChain` methods, DI token wired, `NoopEstimateFollowupCanceller` as default binding
- **Maizzle email templates** (Phase 18/44): existing template infrastructure in `src/email/templates/` — Phase 46 adds `estimate-followup.html`
- **`EstimateTransitionService`** (Phase 41): validates status transitions — Phase 46 uses for `→ Expired` transition
- **`EmailSenderService`** (existing): Resend SDK wrapper for sending emails — Phase 46 reuses for follow-up dispatch

### Established Patterns
- **Queue constant + processor**: `QUEUE_NAMES` in `src/queue/queue.constant.ts` is the single source of truth shared by producer and consumer
- **Deterministic jobId**: STRIPE_WEBHOOKS uses `event.id` as jobId for deduplication — Phase 46 uses `estimate-followup:{estimateId}:{revisionNumber}:{step}`
- **IORedis type workaround**: `as unknown as ConnectionOptions` is the accepted BullMQ integration pattern (v1.4 retrospective)
- **Service orchestration**: `EstimateEmailSender` orchestrates the full send lifecycle (Phase 44 D-AUDIT-08) — Phase 46 adds scheduling as a new step within this orchestration

### Integration Points
- **`EstimateEmailSender`** (Phase 44): inject `EstimateFollowupScheduler`, add `scheduleFollowups()` as step 9 in send lifecycle
- **`EstimateModule`** (Phase 41): Phase 46's `EstimateFollowupsModule` imports `EstimateModule` and rebinds `ESTIMATE_FOLLOWUP_CANCELLER` token
- **`WorkerModule`**: register `EstimateFollowupProcessor` and `EstimateExpiryProcessor`
- **`QUEUE_NAMES`**: add `ESTIMATE_FOLLOWUPS` constant
- **Docker Compose**: add `appendonly yes` / `appendfsync everysec` to Redis service config

</code_context>

<specifics>
## Specific Ideas

### CTA Structure (Cross-Phase Decision)
The response CTAs have been redesigned from the original 4-button structured flow (Phase 45 CUST/RESP requirements) to 3 conversational CTAs that reflect how UK homeowners actually respond to tradespeople:

- **Primary:** "Go Ahead" or "Happy to Proceed" — natural language, avoids contractual weight of "Accept"
- **Secondary:** "Message [Tradesperson First Name]" — single low-friction channel for questions, revisions, site visits
- **Tertiary:** "Not right for me" as text link — softer than "Decline"
- **Disclaimer:** "This is an estimate — the final cost may differ from this approximate figure."

This applies to both the initial send email (Phase 44), the public customer page (Phase 45), and all follow-up emails (Phase 46). Phase 45 CONTEXT.md must be updated before Phase 45 is planned to reflect this change.

### Deterministic JobId Patterns
```
Follow-ups:  estimate-followup:{estimateId}:{revisionNumber}:{step}
             where step = 1 | 2 | 3 (mapping to 3d/10d/21d)
Expiry:      estimate-expiry:{estimateId}:{revisionNumber}
```

### Cancellation Surface
Every exit transition must cancel 4 jobs: 3 follow-ups + 1 expiry.
Exit transitions that trigger cancellation:
- Customer response (Phase 45) → via `cancelAllFollowups`
- Revision send (Phase 44) → already wired in D-AUDIT-08 step 8
- Conversion (Phase 47) → calls `cancelAllFollowupsForChain`
- Mark as lost (Phase 47) → calls `cancelAllFollowups`
- Deletion (Phase 41 EstimateDeleter) → calls `cancelAllFollowups`

</specifics>

<deferred>
## Deferred Ideas

- **Phase 45 CTA update**: Phase 45 CONTEXT.md must be updated to reflect the new 3-CTA conversational structure before Phase 45 is planned. The original CUST/RESP requirements reference a 4-button structured flow (Book site visit / Send me a quote / I have a question / Not right now) that has been superseded.
- **Trader notification on expiry**: Currently silent. Could add an optional email or in-app notification in a future phase if traders report missing expired estimates.
- **Follow-up visibility UI**: Trader-facing indicator showing scheduled follow-ups on the estimate detail page. Deferred — follow-ups are invisible infrastructure in v1.8.
- **Configurable follow-up timing**: Let traders adjust the 3/10/21-day schedule in estimate-settings. Deferred — fixed schedule for v1.8.
- **Follow-up analytics**: Dashboard showing follow-up effectiveness (open rates, response rates per step). Future phase.

None — discussion stayed within phase scope (aside from the CTA cross-phase update noted above).

</deferred>

---

*Phase: 46-follow-up-queue-automation*
*Context gathered: 2026-04-12*
