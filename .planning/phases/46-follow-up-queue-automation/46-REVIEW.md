---
phase: 46-follow-up-queue-automation
reviewed: 2026-04-14T12:00:00Z
depth: standard
files_reviewed: 23
files_reviewed_list:
  - trade-flow-api/docker-compose.yaml
  - trade-flow-api/src/app.module.ts
  - trade-flow-api/src/email/email.module.ts
  - trade-flow-api/src/email/services/estimate-followup-email-renderer.service.ts
  - trade-flow-api/src/email/templates/estimate-followup.html
  - trade-flow-api/src/estimate-followups/data-transfer-objects/followup-job.dto.ts
  - trade-flow-api/src/estimate-followups/estimate-followups.module.ts
  - trade-flow-api/src/estimate-followups/services/bullmq-estimate-followup-canceller.service.ts
  - trade-flow-api/src/estimate-followups/services/estimate-followup-scheduler.service.ts
  - trade-flow-api/src/estimate-followups/services/followup-delays.constant.ts
  - trade-flow-api/src/estimate-followups/test/services/bullmq-estimate-followup-canceller.service.spec.ts
  - trade-flow-api/src/estimate-followups/test/services/estimate-followup-email-renderer.service.spec.ts
  - trade-flow-api/src/estimate-followups/test/services/estimate-followup-scheduler.service.spec.ts
  - trade-flow-api/src/estimate/estimate.module.ts
  - trade-flow-api/src/estimate/services/estimate-email-sender.service.ts
  - trade-flow-api/src/estimate/test/services/estimate-email-sender.service.spec.ts
  - trade-flow-api/src/queue/queue.constant.ts
  - trade-flow-api/src/worker.ts
  - trade-flow-api/src/worker/processors/estimate-followup.processor.ts
  - trade-flow-api/src/worker/test/processors/estimate-expiry.processor.spec.ts
  - trade-flow-api/src/worker/test/processors/estimate-followup.processor.spec.ts
  - trade-flow-api/src/worker/worker.module.ts
  - trade-flow-api/tsconfig.json
findings:
  critical: 1
  warning: 3
  info: 2
  total: 6
status: issues_found
---

# Phase 46: Code Review Report

**Reviewed:** 2026-04-14T12:00:00Z
**Depth:** standard
**Files Reviewed:** 23
**Status:** issues_found

## Summary

Phase 46 adds BullMQ-based follow-up automation for estimates: a scheduler enqueues delayed follow-up emails at 3, 10, and 21 days plus a 30-day expiry job, a processor handles the jobs in a separate worker process, and a canceller removes pending jobs when estimates are revised or responded to. The architecture is well-structured with proper separation of concerns, deterministic job IDs, graceful error handling, and comprehensive test coverage. However, there is one critical bug where the processor uses the wrong email renderer, ignoring the purpose-built follow-up template entirely. Several warnings cover a missing stale-revision guard, duplicated `resolveFromEmail` logic, and an unescaped URL in the follow-up email template.

## Critical Issues

### CR-01: Processor uses wrong email renderer -- follow-up template is never rendered

**File:** `trade-flow-api/src/worker/processors/estimate-followup.processor.ts:107-115`
**Issue:** The `EstimateFollowupProcessor.handleFollowup()` method injects and calls `EstimateEmailRenderer` (the original estimate email renderer) with a hardcoded generic message (`"Just following up on your estimate..."`). Meanwhile, a purpose-built `EstimateFollowupEmailRenderer` was created with its own HTML template (`estimate-followup.html`) containing step-specific copy for 3d/10d/21d follow-ups, "Go Ahead" / "Message" / "Not right for me" CTAs, and the estimate disclaimer. The `followupStep` variable computed from `STEP_TO_LABEL[step]` is only used in the log message, never passed to any renderer.

The `EstimateFollowupEmailRenderer` is registered in `EmailModule`, exported, tested with 14 test cases, and has a dedicated HTML template -- but it is never imported or used by the processor. Customers will receive the generic estimate email (with its own layout, message field, and CTA structure) instead of the purpose-built follow-up email.

**Fix:** Replace `EstimateEmailRenderer` with `EstimateFollowupEmailRenderer` in the processor constructor and update the `handleFollowup` method to call it with the correct data shape:

```typescript
import { EstimateFollowupEmailRenderer, FollowupStep } from "@email/services/estimate-followup-email-renderer.service";

// In constructor:
private readonly estimateFollowupEmailRenderer: EstimateFollowupEmailRenderer,

// In handleFollowup:
const followupStep = STEP_TO_LABEL[step] as FollowupStep;

const html = await this.estimateFollowupEmailRenderer.render({
  businessName: business.name,
  followupStep,
  estimateNumber: estimate.number,
  viewUrl,
  traderFirstName: business.name, // or retrieve user's first name
  priceDisplay,
});
```

Also update `WorkerModule` imports if `EstimateFollowupEmailRenderer` is not already available via `EmailModule`.

## Warnings

### WR-01: No stale-revision guard in follow-up processor

**File:** `trade-flow-api/src/worker/processors/estimate-followup.processor.ts:66-68`
**Issue:** The `handleFollowup` method receives `revisionNumber` in the job data (`IFollowupJobData`) but never checks it against the current estimate's `revisionNumber`. If a follow-up job for revision 1 fires after the estimate has been revised to revision 2 (and the canceller failed due to a transient Redis error), the processor will send a follow-up for a stale revision. The status check (`FOLLOWUP_WORTHY_STATUSES`) alone does not protect against this because a revised estimate in "sent" status would still pass.

**Fix:** Add a revision guard after fetching the estimate:

```typescript
const { estimateId, revisionNumber, step } = job.data;
const estimate = await this.estimateRepository.findByIdOrFail(estimateId);

if (estimate.revisionNumber !== revisionNumber) {
  this.logger.log(`Follow-up skipped -- revision mismatch (job=${revisionNumber}, current=${estimate.revisionNumber})`, { estimateId });
  return;
}
```

### WR-02: Duplicated resolveFromEmail logic across two files

**File:** `trade-flow-api/src/worker/processors/estimate-followup.processor.ts:161-175`
**Issue:** The `resolveFromEmail()` method is duplicated verbatim between `EstimateFollowupProcessor` and `EstimateEmailSender` (lines 197-211). If the from-email resolution logic changes (e.g., new env var, different fallback), both copies must be updated independently. This is a maintenance hazard.

**Fix:** Extract `resolveFromEmail` into a shared utility or service in `@email/` and inject/import it in both consumers:

```typescript
// @email/utilities/resolve-from-email.utility.ts
export const resolveFromEmail = (configService: ConfigService): string => {
  const explicit = configService.get<string>("RESEND_FROM_EMAIL");
  if (explicit) return explicit;
  const corsOrigin = configService.get<string>("CORS_ORIGIN") ?? "";
  const domain = corsOrigin.replace(/^https?:\/\//, "").split(":")[0];
  if (!domain || domain === "localhost") return "onboarding@resend.dev";
  return `noreply@${domain}`;
};
```

### WR-03: viewUrl not HTML-escaped in follow-up email template replacement

**File:** `trade-flow-api/src/email/services/estimate-followup-email-renderer.service.ts:29`
**Issue:** The `viewUrl` value is inserted into the HTML template without escaping (line 29: `.replace(/\{\{\s*viewUrl\s*\}\}/g, data.viewUrl)`), while all other user-facing fields (`businessName`, `estimateNumber`, `priceDisplay`, `traderFirstName`) are passed through `escapeHtml()`. The `viewUrl` is used inside `href` attributes in the template. While the URL is currently constructed server-side from trusted config values and a token, failing to escape it creates a latent XSS vector if the URL construction ever changes or if a token contains unexpected characters.

**Fix:** Apply URL encoding or at minimum HTML-attribute escaping to the viewUrl:

```typescript
.replace(/\{\{\s*viewUrl\s*\}\}/g, this.escapeHtml(data.viewUrl))
```

## Info

### IN-01: Unused followupStep variable in processor (consequence of CR-01)

**File:** `trade-flow-api/src/worker/processors/estimate-followup.processor.ts:104`
**Issue:** `const followupStep = STEP_TO_LABEL[step]` is computed but only used in the log message on line 140. This is a consequence of CR-01 (wrong renderer used). Once CR-01 is fixed, this variable will be passed to the correct renderer and the issue resolves itself.

**Fix:** Will be resolved when CR-01 is addressed.

### IN-02: Type assertion `as never` used in test mock generators

**File:** `trade-flow-api/src/worker/test/processors/estimate-followup.processor.spec.ts:72-73`
**Issue:** The `createBusinessDto` and `createCustomerDto` helper functions use `as never` casts for enum fields (e.g., `status: "active" as never`, `customerType: "individual" as never`). Per project conventions, `as` type assertions should be avoided. While acceptable in tests, these could use the actual enum values for better type safety.

**Fix:** Import and use the actual enum values:

```typescript
import { BusinessStatus } from "@business/enums/business-status.enum";
// ...
status: BusinessStatus.ACTIVE,
```

---

_Reviewed: 2026-04-14T12:00:00Z_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: standard_
