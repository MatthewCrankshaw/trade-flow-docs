# Phase 46: Follow-up Queue & Automation - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-12
**Phase:** 46-follow-up-queue-automation
**Areas discussed:** Follow-up email content, Expiry behaviour, Scheduling trigger, Redis AOF infra gate

---

## Follow-up Email Content

| Option | Description | Selected |
|--------|-------------|----------|
| Same friendly tone | All three use the same warm, professional template — just a gentle nudge each time | ✓ |
| Escalating urgency | 3d: friendly check-in. 10d: reminder. 21d: final notice with expiry warning | |
| You decide | Claude picks | |

**User's choice:** Same friendly tone
**Notes:** Tradesperson brand stays consistent across all touchpoints.

| Option | Description | Selected |
|--------|-------------|----------|
| Brief reminder + link | Short message and View Estimate button linking to the public page | ✓ |
| Full summary included | Re-includes estimate scope, price range, and response buttons inline | |
| You decide | Claude picks | |

**User's choice:** Brief reminder + link
**Notes:** Keeps emails short and drives clicks to the full estimate.

| Option | Description | Selected |
|--------|-------------|----------|
| One shared template | Single estimate-followup.html with step variable | ✓ |
| Three separate templates | Separate files per step (3d/10d/21d) | |

**User's choice:** One shared template

**CTA Structure Change (user-initiated):**
User provided updated CTA design replacing the original 4-button structured flow:
- Primary: "Go Ahead" or "Happy to Proceed"
- Secondary: "Message [Tradesperson First Name]"
- Tertiary: "Not right for me" (text link)
- Disclaimer: "This is an estimate — the final cost may differ from this approximate figure."

| Option | Description | Selected |
|--------|-------------|----------|
| All three CTAs | Follow-up includes all three CTAs matching initial send | ✓ |
| Primary CTA + View link | Just "Go Ahead" button and a "View your estimate" link | |
| You decide | Claude picks | |

**User's choice:** All three CTAs in every follow-up

---

## Expiry Behaviour

| Option | Description | Selected |
|--------|-------------|----------|
| Yes, email the trader | Brief notification email on auto-expiry | |
| Silent transition only | Status flips to Expired, trader notices when viewing estimate list | ✓ |
| You decide | Claude picks | |

**User's choice:** Silent transition only

| Option | Description | Selected |
|--------|-------------|----------|
| Friendly expired message | "This estimate has expired. Please contact [Business Name] if you're still interested." | ✓ |
| Last active state (read-only) | Estimate content visible but buttons disabled with expired banner | |
| You decide | Claude picks | |

**User's choice:** Friendly expired message

| Option | Description | Selected |
|--------|-------------|----------|
| Periodic sweep | Hourly recurring job queries and expires overdue estimates in batch | |
| Per-estimate delayed job | Each send enqueues a 30-day delayed job with deterministic jobId | ✓ |
| You decide | Claude picks | |

**User's choice:** Per-estimate delayed job

---

## Scheduling Trigger

| Option | Description | Selected |
|--------|-------------|----------|
| Immediately on send | Follow-ups enqueued the moment send completes | ✓ |
| Configurable delay | Trader sets a wait period in estimate-settings | |

**User's choice:** Immediately on send

| Option | Description | Selected |
|--------|-------------|----------|
| No visibility | Follow-ups are invisible infrastructure, zero UI | ✓ |
| Read-only indicator | Estimate detail shows "Next follow-up: 3 days" | |
| Full management | Trader can see, pause, or cancel individual follow-ups | |

**User's choice:** No visibility

| Option | Description | Selected |
|--------|-------------|----------|
| Inside EstimateEmailSender | Inject scheduler, call as step 9 in send lifecycle | ✓ |
| Controller orchestration | Controller calls sender then scheduler as separate calls | |
| You decide | Claude picks | |

**User's choice:** Inside EstimateEmailSender

---

## Redis AOF Infrastructure Gate

| Option | Description | Selected |
|--------|-------------|----------|
| Runtime startup check | Worker checks CONFIG GET appendonly, refuses to register processor if not enabled | ✓ |
| Deploy-time manual check | Pre-deploy checklist item, no runtime enforcement | |
| You decide | Claude picks | |

**User's choice:** Runtime startup check
**Notes:** User asked for explanation of what Redis AOF is before answering. Explained: AOF writes every Redis operation to disk so scheduled jobs survive restarts. Without it, follow-ups are silently lost on restart.

| Option | Description | Selected |
|--------|-------------|----------|
| Manual smoke procedure | Documented steps: enqueue test job, restart Redis, assert job fires | ✓ |
| Automated integration test | Jest test that programmatically restarts Redis container | |
| You decide | Claude picks | |

**User's choice:** Manual smoke procedure

---

## Claude's Discretion

- Exact copy variations between 3d/10d/21d follow-up steps
- EstimateFollowupsModule internal structure and provider wiring
- Whether expiry is a separate processor or special case within follow-up processor
- Error handling / retry policy for failed follow-up sends
- Exact order of AOF startup check within worker bootstrap

## Deferred Ideas

- Phase 45 CONTEXT.md must be updated with new 3-CTA structure before planning
- Trader notification on expiry (currently silent)
- Follow-up visibility UI on estimate detail page
- Configurable follow-up timing
- Follow-up analytics dashboard
