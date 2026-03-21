# Phase 19: Customer Response - Research

**Researched:** 2026-03-21
**Domain:** Public quote response flow (accept/decline) + tradesperson notification emails
**Confidence:** HIGH

## Summary

Phase 19 adds accept/decline functionality to the public quote page and sends notification emails to the tradesperson. The existing codebase is exceptionally well-prepared: quote status transitions (SENT -> ACCEPTED/REJECTED) already work, the public controller pattern with `QuoteSessionAuthGuard` is established, status banners for accepted/rejected already exist in the UI, and the Resend email infrastructure is operational from Phase 18.

The primary work is: (1) new POST endpoints on the public controller for accept/reject, (2) a new service to orchestrate the response + send notification email, (3) a notification email HTML template, (4) adding `declineReason` to the quote entity/DTO, (5) frontend accept/decline buttons with inline state transitions, and (6) adding a `findByBusinessId` method to `BusinessUserRepository` to resolve the tradesperson's email for notifications.

**Primary recommendation:** Lean heavily on existing infrastructure. The `QuoteTransitionService` already handles status changes (including timestamps). The new `QuoteResponseService` (or `QuoteResponder`) orchestrates: validate response is allowed -> transition status -> send notification email. Frontend uses RTK Query mutations on the separate `publicQuoteApi` slice.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- **D-01:** One-click accept -- no confirmation dialog, no signature. Customer clicks Accept, transition happens immediately, inline success state shown.
- **D-02:** Decline shows an optional text field for a reason. Customer can submit with or without a reason.
- **D-03:** After responding, buttons are replaced with an inline status banner on the same page ("You accepted this quote" / "You declined this quote"). Quote details remain visible below.
- **D-04:** Accept/Decline buttons appear at the bottom of the quote card, below the totals section. Natural reading flow.
- **D-05:** Simple factual tone -- "[Customer] accepted your quote [Q-001] for [Job Title]." Links to the quote in the app.
- **D-06:** If the customer provided a decline reason, include it directly in the notification email body.
- **D-07:** New simple notification HTML template -- business branding header, message text, CTA button to view in app. Lighter than the quote email template.
- **D-08:** Expired quotes (validUntil date passed) show "This quote has expired" instead of response buttons.
- **D-09:** Response is final -- once accepted or declined, no reversal.
- **D-10:** Token stays active after response -- customer can revisit the link to see the quote and their response status.

### Claude's Discretion
- Button styling and colors (Accept = primary, Decline = secondary/destructive)
- Exact notification email template layout
- Loading states during response submission
- Error handling UX (network failure during response)
- Decline reason textarea placeholder text and character limit

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| RESP-02 | Customer can accept a quote with one click from the online view | Public POST endpoint + QuoteTransitionService (SENT->ACCEPTED already valid) + frontend Accept button with RTK mutation |
| RESP-03 | Customer can decline a quote with one click from the online view | Public POST endpoint + QuoteTransitionService (SENT->REJECTED already valid) + frontend Decline button with inline form |
| RESP-04 | Customer can provide an optional reason when declining a quote | New `declineReason` field on quote entity/DTO + decline request body + display in notification email |
| AUTO-02 | Quote status automatically transitions from Sent to Accepted when the customer accepts | QuoteTransitionService.transition() already sets acceptedAt timestamp; called from new public response service |
| AUTO-03 | Quote status automatically transitions from Sent to Rejected when the customer declines | QuoteTransitionService.transition() already sets rejectedAt timestamp; called from new public response service |
| NOTF-01 | Tradesperson receives an email notification when a customer accepts | New notification email template + QuoteResponseNotifier service + resolve tradesperson email via BusinessUserRepository |
| NOTF-02 | Tradesperson receives an email notification when a customer declines | Same notification service, different email subject/body content based on response type |
</phase_requirements>

## Standard Stack

### Core (Already in project)
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| NestJS | 11.x | API framework | Project standard |
| React | 19 | Frontend framework | Project standard |
| RTK Query | (via Redux Toolkit) | API state management | Existing `publicQuoteApi` slice for unauthenticated endpoints |
| Resend SDK | (installed) | Email sending | Phase 18 infrastructure |
| Maizzle | (installed) | Email template CSS inlining | Phase 18 pattern |
| class-validator | (installed) | Request validation | API standard |
| Luxon | (installed) | Date/time handling | API standard |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| lucide-react | (installed) | Icons | Loading spinner, check/x icons in status banners |
| shadcn/ui | (installed) | UI components | Button, Textarea, Card already installed |
| date-fns | (installed) | Date formatting (frontend) | Format response dates in banners |

No new packages need to be installed for this phase.

## Architecture Patterns

### Backend: New Files Needed

```
trade-flow-api/src/
├── quote-token/
│   ├── controllers/
│   │   └── public-quote.controller.ts          # MODIFY: add POST accept/decline endpoints
│   ├── requests/
│   │   └── respond-quote.request.ts            # NEW: validation for decline reason
│   └── services/
│       └── quote-response-handler.service.ts   # NEW: orchestrate response + notification
├── quote/
│   ├── entities/
│   │   └── quote.entity.ts                     # MODIFY: add declineReason field
│   ├── data-transfer-objects/
│   │   └── quote.dto.ts                        # MODIFY: add declineReason field
│   ├── repositories/
│   │   └── quote.repository.ts                 # MODIFY: persist declineReason in update $set
│   └── services/
│       └── quote-transition.service.ts         # NO CHANGE: already handles SENT->ACCEPTED/REJECTED
├── email/
│   ├── templates/
│   │   └── notification-email.html             # NEW: lighter template for notifications
│   └── services/
│       └── notification-email-renderer.service.ts  # NEW: render notification template
├── business/
│   ├── repositories/
│   │   └── business-user.repository.ts         # MODIFY: add findByBusinessId method
│   └── business.module.ts                      # MODIFY: export BusinessUserRepository
```

### Frontend: New Files Needed

```
trade-flow-ui/src/
├── features/public-quote/
│   ├── api/
│   │   └── publicQuoteApi.ts                   # MODIFY: add respondToQuote mutation
│   ├── components/
│   │   ├── PublicQuoteCard.tsx                  # MODIFY: add response buttons section
│   │   ├── PublicQuoteResponseButtons.tsx       # NEW: Accept/Decline button component
│   │   ├── DeclineReasonForm.tsx               # NEW: inline decline form
│   │   └── QuoteExpiredNotice.tsx              # NEW: expired quote banner
│   └── types/
│       └── public-quote.types.ts               # MODIFY: add response types
```

### Pattern 1: Public Response Endpoint

**What:** POST endpoints on the existing `PublicQuoteController` for accept/decline, using the same `QuoteSessionAuthGuard` + `ThrottlerGuard` pattern.

**When to use:** All customer-initiated actions on public quotes.

**Example:**
```typescript
// Source: Existing public-quote.controller.ts pattern
@Throttle({ default: { limit: 10, ttl: 60000 } })  // Stricter rate limit for mutations
@UseGuards(QuoteSessionAuthGuard)
@Post("quote/:token/accept")
public async acceptQuote(
  @Req() request: { quoteToken: IQuoteTokenDto },
): Promise<IResponse<IPublicQuoteResponse>> {
  try {
    const response = await this.quoteResponseHandler.accept(request.quoteToken);
    return createResponse([response]);
  } catch (error) {
    throw createHttpError(error);
  }
}
```

### Pattern 2: Response Handler Service (Orchestrator)

**What:** New service in `quote-token` module that orchestrates: validate quote is respondable -> call `QuoteTransitionService` -> store decline reason -> send notification email -> return updated public quote.

**Key design decision:** The `QuoteTransitionService.transition()` requires an `IUserDto` (for authorization). Since this is a public endpoint (no authenticated user), a new method or approach is needed. Options:
1. Add a `publicTransition()` method to `QuoteTransitionService` that skips authorization (recommended -- cleanest separation)
2. Create a dedicated `QuoteRepository.updateStatus()` that the response handler calls directly

**Recommended approach (Option 1):**
```typescript
// New method on QuoteTransitionService -- no authUser param, no policy check
public async publicTransition(quoteId: string, targetStatus: QuoteStatus, declineReason?: string): Promise<IQuoteDto> {
  const existing = await this.quoteRepository.findByIdOrFail(quoteId);

  if (!isValidTransition(existing.status, targetStatus)) {
    throw new InvalidRequestError(/* ... */);
  }

  const updated: IQuoteDto = { ...existing, status: targetStatus };
  if (targetStatus === QuoteStatus.ACCEPTED) {
    updated.acceptedAt = DateTime.now();
  } else if (targetStatus === QuoteStatus.REJECTED) {
    updated.rejectedAt = DateTime.now();
    updated.declineReason = declineReason;
  }

  return this.quoteRepository.update(updated);
}
```

### Pattern 3: Tradesperson Email Resolution

**What:** Resolve the tradesperson's email address from a `businessId` to send notification emails.

**Flow:** `quote.businessId` -> `BusinessUserRepository.findByBusinessId()` (new method) -> first `userId` -> `UserRepository.findById()` -> `user.email`

**Why this path:** The `BusinessUser` join table is the canonical link between businesses and users. For a sole trader app there's exactly one user per business.

**Required changes:**
1. Add `findByBusinessId(businessId: string)` to `BusinessUserRepository`
2. Export `BusinessUserRepository` from `BusinessModule`
3. Import into `QuoteTokenModule` (which hosts the response handler)

### Pattern 4: Frontend Inline State Machine

**What:** The response buttons component manages its own local state: `idle` -> `accepting`/`declining` -> `success` -> shows banner. No page reload needed.

**Example:**
```typescript
type ResponseState = "idle" | "accepting" | "declining" | "accepted" | "declined";

// After successful mutation, update local state to show banner
// Also invalidate the getPublicQuote query so revisits show correct state
```

### Anti-Patterns to Avoid

- **Do NOT create a separate controller** for response endpoints. Add to the existing `PublicQuoteController` -- same auth guard, same throttling, same module.
- **Do NOT use the authenticated QuoteTransitionService.transition()** for public responses. It requires `IUserDto` and checks authorization policies, which don't apply to customer actions.
- **Do NOT create a separate RTK Query API slice** for mutations. Add mutations to the existing `publicQuoteApi` slice.
- **Do NOT add a confirmation dialog** for Accept (D-01 explicitly prohibits this).
- **Do NOT use `variant="destructive"` for Decline** -- UI spec says `variant="outline"` (it's a choice, not a destructive action).

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Status transition validation | Custom if/else chains | `isValidTransition()` from `quote-transitions.ts` | Already handles all valid transitions + terminal states |
| Timestamp setting on transition | Manual timestamp logic | Extend `QuoteTransitionService` with public method | Centralized transition logic, DRY |
| Email HTML inlining | Manual style attributes | Maizzle via `QuoteEmailRenderer` pattern | CSS-to-inline conversion is complex, already solved |
| Rate limiting | Custom request counting | NestJS `@Throttle` decorator (already on controller) | Battle-tested, per-IP tracking |
| Quote expiry check (frontend) | Complex date logic | `date-fns` `isPast(parseISO(validUntil))` | Simple, already imported |
| Request validation | Manual field checking | class-validator decorators on request class | Project standard |

## Common Pitfalls

### Pitfall 1: QuoteTransitionService Authorization Mismatch
**What goes wrong:** Trying to call the existing `transition()` method from a public endpoint without an `IUserDto` causes a compilation error or forces creating a fake user.
**Why it happens:** The existing method is designed for authenticated tradesperson actions with policy checks.
**How to avoid:** Add a separate `publicTransition()` method (or a new dedicated service method) that validates the transition is allowed but skips user authorization. The transition itself (SENT -> ACCEPTED/REJECTED) is already validated by `isValidTransition()`.
**Warning signs:** If you find yourself constructing a fake `IUserDto`, you're going down the wrong path.

### Pitfall 2: Race Condition on Double-Click Accept
**What goes wrong:** Customer double-clicks Accept, two requests arrive, second one tries ACCEPTED -> ACCEPTED which is an invalid transition.
**Why it happens:** No idempotency protection.
**How to avoid:** Two layers: (1) Frontend disables buttons during request, (2) Backend checks current status before transitioning -- if already ACCEPTED, return success (idempotent) rather than throwing an error. Same for REJECTED.
**Warning signs:** 422 errors in production from "Cannot transition from accepted to accepted."

### Pitfall 3: Notification Email Fails But Response Succeeds
**What goes wrong:** Quote transitions to ACCEPTED but email sending throws (Resend API down, key invalid). Customer sees success, tradesperson never gets notified.
**Why it happens:** Email sending is in the same transaction as the status update.
**How to avoid:** The status transition is the critical operation -- it should succeed even if email fails. Wrap email sending in try/catch, log the failure, but return success to the customer. The tradesperson will see the updated status when they next view the quote in the app.
**Warning signs:** If email failure causes a 500 to the customer.

### Pitfall 4: Expired Quote Not Checked on Backend
**What goes wrong:** Frontend checks `validUntil` and hides buttons, but a crafted API request to an expired quote still succeeds.
**Why it happens:** Relying only on frontend validation.
**How to avoid:** Backend must also check `validUntil` before allowing a response. If the quote has expired, return 422 with a clear error.
**Warning signs:** Status transitions on quotes past their validity date.

### Pitfall 5: Missing declineReason in Repository Update
**What goes wrong:** `declineReason` is set on the DTO but the `QuoteRepository.update()` method's `$set` doesn't include it, so it's never persisted.
**Why it happens:** The `update()` method explicitly lists which fields go in `$set`. Adding a new DTO field without updating the repository is easy to miss.
**How to avoid:** Update both `toEntity()` and the `$set` in `update()` to include `declineReason`.
**Warning signs:** Decline reason shows as undefined when re-fetching the quote.

### Pitfall 6: Status Banner Date Shows Wrong Value
**What goes wrong:** The existing accepted/rejected banners in `PublicQuoteCard` use `quote.quoteDate` for the banner text ("You accepted this quote on {date}"). This shows the quote date, not the acceptance/rejection date.
**Why it happens:** The `IPublicQuoteResponse` doesn't currently include `acceptedAt`/`rejectedAt` timestamps.
**How to avoid:** Add `acceptedAt` and `rejectedAt` to `IPublicQuoteResponse` so the banners can show the correct response date. Alternatively, use the response from the mutation call which will have the current date.
**Warning signs:** Banner says "You accepted this quote on 15 March 2026" when they actually accepted on 21 March 2026.

## Code Examples

### Backend: Decline Request Validation
```typescript
// Source: Project pattern from create-business.request.ts
import { IsOptional, IsString, MaxLength } from "class-validator";

export class DeclineQuoteRequest {
  @IsOptional()
  @IsString()
  @MaxLength(500)
  reason?: string;
}
```

### Backend: Notification Email Data Interface
```typescript
// Source: Follows QuoteEmailData pattern from quote-email-renderer.service.ts
export interface NotificationEmailData {
  businessName: string;
  message: string;
  viewUrl: string;   // Link to quote in the authenticated app
}
```

### Backend: Notification Email Renderer
```typescript
// Source: Follows QuoteEmailRenderer pattern
@Injectable()
export class NotificationEmailRenderer {
  public async render(data: NotificationEmailData): Promise<string> {
    const templatePath = path.join(__dirname, "../templates/notification-email.html");
    const template = fs.readFileSync(templatePath, "utf-8");

    const populated = template
      .replace(/\{\{\s*businessName\s*\}\}/g, this.escapeHtml(data.businessName))
      .replace(/\{\{\{\s*message\s*\}\}\}/g, data.message)  // Triple braces for pre-formatted HTML
      .replace(/\{\{\s*viewUrl\s*\}\}/g, data.viewUrl);

    // Maizzle inlining with fallback (same pattern as QuoteEmailRenderer)
    try {
      const maizzle = await (Function('return import("@maizzle/framework")')() as Promise<Record<string, unknown>>);
      // ...
    } catch (_maizzleError) {
      return populated;
    }
  }
}
```

### Frontend: RTK Query Mutation
```typescript
// Source: Follows existing publicQuoteApi pattern
respondToQuote: builder.mutation<PublicQuoteResponse, { token: string; action: "accept" | "decline"; reason?: string }>({
  query: ({ token, action, reason }) => ({
    url: `/v1/public/quote/${token}/${action}`,
    method: "POST",
    body: action === "decline" ? { reason } : undefined,
  }),
  transformResponse: (response: StandardResponse<PublicQuoteResponse>) => {
    if (response.data && response.data.length > 0) {
      return response.data[0];
    }
    throw new Error("No response data returned");
  },
}),
```

### Frontend: Expiry Check
```typescript
// Source: date-fns already imported in PublicQuoteCard
import { isPast, parseISO } from "date-fns";

const isExpired = quote.validUntil && isPast(parseISO(quote.validUntil));
const canRespond = quote.status === "sent" && !isExpired;
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| N/A -- first implementation | N/A | N/A | No migration needed |

All existing infrastructure is current and in use. No deprecated patterns to worry about.

## Open Questions

1. **Tradesperson email resolution path**
   - What we know: `BusinessUser` join table links `businessId` to `userId`, and `UserRepository` can look up users by ID. Users have an `email` field.
   - What's unclear: Whether to add `findByBusinessId()` to `BusinessUserRepository` or use a different lookup path.
   - Recommendation: Add `findByBusinessId()` to `BusinessUserRepository` and export it from `BusinessModule`. This is the cleanest approach and follows existing patterns.

2. **Notification email "View Quote" URL**
   - What we know: The notification email should link the tradesperson to the quote in the authenticated app (not the public link).
   - What's unclear: The exact URL pattern for the quote detail page in the authenticated app.
   - Recommendation: Use `${CORS_ORIGIN}/quotes/${quoteId}` pattern (matching the app's routing). This can be verified by checking the router configuration.

3. **IPublicQuoteResponse: acceptedAt/rejectedAt timestamps**
   - What we know: The existing status banners use `quoteDate` which is incorrect for the response date.
   - What's unclear: Whether to add these fields to the public response or handle it differently.
   - Recommendation: Add `acceptedAt` and `rejectedAt` string fields to `IPublicQuoteResponse`. The frontend inline state machine can use `new Date()` for the initial banner after responding, then the server-provided date on subsequent visits.

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Jest (configured in trade-flow-api) |
| Config file | `trade-flow-api/jest.config.ts` (if exists) or `package.json` jest config |
| Quick run command | `cd trade-flow-api && npm run test -- --testPathPattern="<pattern>" --no-coverage` |
| Full suite command | `cd trade-flow-api && npm run test` |

### Phase Requirements to Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| RESP-02 | Accept endpoint transitions quote SENT->ACCEPTED | unit | `npm run test -- --testPathPattern="quote-response-handler" --no-coverage` | Wave 0 |
| RESP-03 | Decline endpoint transitions quote SENT->REJECTED | unit | Same as above | Wave 0 |
| RESP-04 | Decline reason stored on quote | unit | Same as above | Wave 0 |
| AUTO-02 | Accept sets acceptedAt timestamp | unit | Covered by transition service spec (exists) | Existing |
| AUTO-03 | Decline sets rejectedAt timestamp | unit | Covered by transition service spec (exists) | Existing |
| NOTF-01 | Notification email sent on accept | unit | `npm run test -- --testPathPattern="quote-response-handler" --no-coverage` | Wave 0 |
| NOTF-02 | Notification email sent on decline (with/without reason) | unit | Same as above | Wave 0 |

### Sampling Rate
- **Per task commit:** `cd trade-flow-api && npm run test -- --testPathPattern="quote-response|notification-email" --no-coverage`
- **Per wave merge:** `cd trade-flow-api && npm run test && cd ../trade-flow-ui && npm run typecheck && npm run lint`
- **Phase gate:** Full suite green before `/gsd:verify-work`

### Wave 0 Gaps
- [ ] `trade-flow-api/src/quote-token/test/services/quote-response-handler.service.spec.ts` -- covers RESP-02, RESP-03, RESP-04, NOTF-01, NOTF-02
- [ ] `trade-flow-api/src/email/test/services/notification-email-renderer.service.spec.ts` -- covers notification template rendering
- [ ] `trade-flow-api/src/quote/test/services/quote-transition.service.spec.ts` -- may need update for new `publicTransition` method (existing file likely exists)

## Sources

### Primary (HIGH confidence)
- Existing codebase files -- all patterns verified by reading actual source code
- `trade-flow-api/src/quote-token/controllers/public-quote.controller.ts` -- public endpoint pattern
- `trade-flow-api/src/quote/services/quote-transition.service.ts` -- transition logic
- `trade-flow-api/src/quote/enums/quote-transitions.ts` -- SENT->ACCEPTED/REJECTED already valid
- `trade-flow-api/src/email/services/email-sender.service.ts` -- Resend email sending
- `trade-flow-api/src/email/services/quote-email-renderer.service.ts` -- template rendering pattern
- `trade-flow-ui/src/features/public-quote/` -- frontend public quote infrastructure
- `.planning/phases/19-customer-response/19-CONTEXT.md` -- user decisions
- `.planning/phases/19-customer-response/19-UI-SPEC.md` -- visual/interaction contract

### Secondary (MEDIUM confidence)
- None needed -- all findings verified from codebase

### Tertiary (LOW confidence)
- None

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- all libraries already in use, no new dependencies
- Architecture: HIGH -- patterns directly observed from existing code, clear extension points
- Pitfalls: HIGH -- derived from actual code structure analysis (e.g., missing `declineReason` in `$set`, `QuoteTransitionService` auth requirement)

**Research date:** 2026-03-21
**Valid until:** 2026-04-21 (stable -- no external dependencies changing)
