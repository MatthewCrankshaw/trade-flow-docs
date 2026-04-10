# Pitfalls Research: v1.8 Estimates

**Domain:** Adding estimates as a parallel document type alongside shipped quotes, with BullMQ delayed follow-ups, versioned revisions, and customer public pages -- in existing Trade Flow SaaS (NestJS 11 / React 19 / MongoDB / BullMQ 5.x / Redis / Resend / Dinero.js / Luxon)
**Researched:** 2026-04-10
**Overall confidence:** HIGH for UK legal, BullMQ, and Dinero.js findings (verified against official docs and ICO guidance). MEDIUM for conversion/revision patterns (derived from codebase conventions + MongoDB community patterns).

## Executive Summary

Adding estimates to Trade Flow looks superficially like "copy the quote feature and rename it," but that framing hides five categories of real risk: (1) UK legal exposure if the word "estimate" is used loosely enough that it reads as an offer, (2) BullMQ delayed-job lifecycle bugs where follow-ups fire after the estimate has already been revised/converted/declined, (3) race conditions on versioned revisions because the public token must resolve to the **latest** revision, not the one the customer was emailed, (4) quote/estimate type confusion at the shared code boundary, and (5) rounding artifacts on the contingency slider that would make the customer-facing price range look wrong.

The highest-consequence category is the first: the Consumer Rights Act 2015 treats estimates as non-binding guide prices, but UK case law is clear that **a badly worded estimate can become a binding offer** if a customer unconditionally accepts it, and courts can and do step in to enforce a "reasonable price" ceiling if the final invoice drifts significantly above the estimate. The email template, the customer page copy, and the slider's range display are all legal-surface-area, not just UX surface area.

The second-highest category is BullMQ delayed-job hygiene. Trade Flow's v1.4 worker infrastructure has shipped, but only for synchronous event processing (echo processor, Stripe webhook events). This milestone is the first production use of **delayed** jobs, which have a different failure surface: orphaned follow-ups after a revision, stalled jobs if the worker is killed mid-processing, clock skew on the 3/10/21 day schedule, and the silent-data-loss class of Redis eviction bugs that the v1.4 milestone already pre-empted with `maxmemory-policy noeviction`.

Most other risks are mitigable with disciplined phase tasks: a `DocumentType` discriminator at the type level, unique compound indexes for revisions, idempotency keys on the conversion endpoint, and `cn(Dinero.allocate)` for the contingency math rather than naive percentage multiplication.

## Critical Pitfalls

Mistakes that cause legal exposure, data loss, or user-visible incorrectness. These MUST be addressed in the roadmap.

---

### Pitfall 1: Estimate email/template wording creates a binding offer under UK law

**What goes wrong:** The customer-facing estimate page or email template uses language that, to a reasonable person, reads as a firm offer -- phrases like "Your price: £1,800", "Total cost", "Accept this estimate", or copying quote-template variables verbatim into the estimate template. Customer clicks "Book a site visit", treats it as acceptance, and later disputes any deviation from £1,800.

**Why it matters:** UK case law is clear that an estimate can become a binding contract where there is unconditional acceptance of the stated price, especially when the trader has not explicitly flagged the figure as indicative. Even where the estimate remains non-binding, the Consumer Rights Act 2015 s.51 ("reasonable price" term) constrains how far the final invoice can drift from the estimate -- courts have historically treated 10-20% overages as the rough ceiling of reasonableness where no unforeseen circumstances exist. The legal risk is not theoretical: it is the single most-disputed scenario in UK trades-consumer complaints.

**Consequences:**
- Tradesperson loses a small claims case because the estimate read as a quote
- Trade Flow becomes the evidence trail for the dispute (we store the exact copy the customer saw)
- Reputational risk if the product ships with copy that regulators or Trading Standards would flag

**Prevention:**
- Mandatory "From £X" or "£X - £Y" display on the customer page (never a single "Total" for estimates)
- Default email template must include explicit non-binding language: "This is an estimate, not a fixed price. The final cost may vary based on site conditions, materials, and labour time. A firm quote will be provided after a site visit."
- Customer page buttons MUST NOT include the word "Accept". Use "Book a site visit", "Send me a quote", "Not right now" as already scoped
- Legal-review pass on the default email template and the customer page copy
- Currency symbol on the contingency slider shows both ends of the range at all times
- Estimate PDF/email subject line must include the word "Estimate" (not "Quote")
- Store the exact rendered email HTML in the estimate document so future disputes can reference what the customer actually received

**Detection:** Manual copy review in Phase 1; automated test asserting the word "Estimate" (not "Quote") appears in the email subject and the non-binding disclaimer is present in the rendered body.

**Phase:** Phase 1 (Data model + legal copy baseline) and Phase 5 (Email template)

**Sources:**
- [Consumer Rights Act 2015, Part 1, Chapter 4 -- Services](https://www.legislation.gov.uk/ukpga/2015/15/part/1/chapter/4) (HIGH confidence -- statute)
- [Contractor Exceeds Original Quote: What Are Your Rights? -- Hegarty Solicitors](https://hegarty.co.uk/news/contractor-exceeds-original-quote-what-are-your-rights) (MEDIUM -- solicitor analysis)
- [Demystifying Estimates and Quotes in the UK -- iDecorate](https://www.idecorateuk.co.uk/post/estimating-vs-quoting) (MEDIUM -- practitioner guidance)
- [Estimates -- Beware! -- BTO Solicitors](https://www.bto.co.uk/blog/estimates-%E2%80%93-beware!.aspx) (MEDIUM -- UK law firm commentary)

---

### Pitfall 2: Delayed follow-up jobs orphaned after revision/conversion/decline

**What goes wrong:** A tradesperson sends an estimate on day 0. BullMQ schedules three delayed jobs (3d, 10d, 21d). On day 2 the customer declines, OR the tradesperson revises the estimate, OR converts it to a quote. The already-scheduled jobs are not cancelled and the customer receives "Still thinking about your estimate?" follow-ups days after they already declined or booked a site visit.

**Why it matters:** This is brand-damaging for the tradesperson (they look automated and uncaring), and operationally confusing (the customer thinks they were ignored). It is also the *easy* bug to ship because BullMQ does not automatically link jobs to domain entities -- you have to remember to `remove()` them.

**Consequences:**
- Customer receives "nudge" email after already responding
- Customer receives 21-day follow-up for an estimate converted to a quote on day 4
- Tradesperson manually disables follow-ups because they do not trust them

**Prevention:**
- **Deterministic jobId pattern:** `followup:{estimateId}:{revisionNumber}:{stage}` (e.g. `followup:65abc:3:10d`). This gives idempotency on `Queue.add()` (BullMQ ignores duplicates when jobId matches) AND enables targeted cancellation via `Queue.remove(jobId)`
- On every status transition out of `Sent`/`Viewed` (to Responded/Converted/Declined/Lost), the service MUST call a `cancelPendingFollowups(estimateId, revisionNumber)` helper
- On `Revise Estimate`, cancel follow-ups for the *previous* revision before scheduling new ones for the new revision
- Worker processor MUST re-read the current estimate state from MongoDB at execution time and short-circuit with a log line if the estimate is no longer in a state that warrants a follow-up ("estimate 65abc in status Declined, skipping follow-up")
- Do NOT rely solely on cancellation -- the worker-level guard is defence in depth
- Unit test: "scheduling follow-up, then transitioning to Declined, then fast-forwarding time, asserts no email send"

**Detection:** Integration test running through every status transition with a mocked BullMQ queue, asserting `queue.remove()` was called. Worker logs should show explicit "skipping follow-up" lines as a signal.

**Phase:** Phase 4 (Follow-up sequence)

**Sources:**
- [BullMQ -- Job Ids documentation](https://docs.bullmq.io/guide/jobs/job-ids) (HIGH -- official)
- [BullMQ -- Removing Jobs](https://docs.bullmq.io/guide/queues/removing-jobs) (HIGH -- official)
- [BullMQ -- Deduplication](https://docs.bullmq.io/guide/jobs/deduplication) (HIGH -- official)

---

### Pitfall 3: Customer public token resolves to stale revision after revise

**What goes wrong:** Estimate v1 is sent with token `abc123`. Customer views it. Tradesperson revises (creates revision v2) -- the UX says "edit and resend," suggesting the customer should see new data. The customer clicks the original email link again. The token `abc123` is bound to revision v1, not "the estimate as a concept," and the customer sees the old figures.

**Why it matters:** This is user-invisible until a customer complaint like "you told me £1,800 but now you're saying £2,100." The revision model is explicitly designed to be invisible to the user -- but the public token MUST respect that invisibility by always resolving to the latest revision of the parent estimate, not the specific revision snapshot that existed when the token was issued.

**Consequences:**
- Customer disputes the price because they are looking at stale data
- Confusion between tradesperson ("I sent you the revised version") and customer ("I'm looking at the original link you sent me")
- Legal risk compounds with Pitfall 1 if the stale version is the one the customer "accepts"

**Prevention:**
- Token is bound to `parentEstimateId`, NOT to a specific revision document
- Public endpoint `GET /public/estimates/:token` MUST query: "find the estimate where `parentEstimateId = X` ordered by `revisionNumber` desc, limit 1"
- Compound index: `{ parentEstimateId: 1, revisionNumber: -1 }` to make this query an indexed lookup
- If revising is intended to reset customer-trust (e.g. "send a fresh link"), that must be an explicit product decision -- for now, always latest
- On revise, re-send email with the same original token (not a new one), so the customer's email trail continues to work
- The firstViewedAt tracking should be on the parent (or cumulative across revisions), so revising does not clear view history

**Detection:** Integration test: create estimate v1, generate token, create revision v2, hit the public endpoint with the v1 token, assert response shows v2 data.

**Phase:** Phase 2 (Revisions + public page)

**Sources:**
- [MongoDB -- Document Versioning design pattern](https://www.mongodb.com/docs/manual/data-modeling/design-patterns/data-versioning/document-versioning/) (HIGH -- official)
- Trade Flow v1.3 QuoteSessionAuthGuard pattern (HIGH -- internal codebase)

---

### Pitfall 4: BullMQ jobs lost on worker restart due to Redis persistence misconfig

**What goes wrong:** Redis is restarted (redeploy, OOM, Railway container recycle) and the in-flight delayed follow-up jobs are gone. The tradesperson sent 50 estimates yesterday and will get zero follow-ups tomorrow. No error, no log, silent data loss.

**Why it matters:** The v1.4 milestone correctly set `maxmemory-policy noeviction` to avoid BullMQ silently losing keys, but BullMQ *also* requires Redis persistence (AOF or RDB) to survive restarts. Ephemeral Redis was fine for v1.4 because webhook jobs were processed within seconds; delayed jobs live in Redis for up to 21 days and MUST survive restarts.

**Consequences:**
- All scheduled follow-ups silently deleted on deploy
- No alert; tradesperson finds out via missed customer engagement weeks later
- Product looks buggy in a way that is very hard to debug after the fact

**Prevention:**
- Production Redis (Railway/Upstream) MUST have AOF persistence enabled (`appendonly yes`, `appendfsync everysec`)
- Add a production smoke test: schedule a job with 60s delay, restart Redis, verify job still fires
- Document the Redis persistence requirement in `.planning/codebase/ARCHITECTURE.md` and the infra README
- Consider adding a startup check that logs the Redis persistence config at worker boot
- **QueueScheduler is NOT needed** -- it was deprecated in BullMQ 2.0+; delayed jobs work without it in the current version (5.x)
- Use BullMQ's job completion/failure events to surface failures to logs; do not rely on "no news is good news"

**Detection:** Production smoke test on each deploy. Periodic check that `INFO persistence` on Redis reports AOF enabled.

**Phase:** Phase 4 (Follow-up sequence) -- but the infra work must happen before the first follow-up ships

**Sources:**
- [BullMQ -- Going to Production](https://docs.bullmq.io/guide/going-to-production) (HIGH -- official)
- [BullMQ -- QueueScheduler deprecation](https://docs.bullmq.io/guide/queuescheduler) (HIGH -- official)
- Trade Flow v1.4 key decision: `maxmemory-policy noeviction` (HIGH -- internal)

---

### Pitfall 5: Contingency slider percentage math produces rounding artifacts

**What goes wrong:** Base price is £1,799.99 (179,999 pence). Contingency is 10%. Naive math: `179999 * 0.1 = 17999.9`, rounded to `18000` pence = £180.00, so high end = £1,979.99. But if another dev uses `Math.round(179999 * 1.10)` they get `197999` pence = £1,979.99. And if a third place in the code uses `179999 * 1.1` without rounding, they get a floating-point tail like `197998.9000...`. Three different "correct" answers appear in three places (API, list view, customer page).

**Why it matters:** Money bugs are brand-damaging, and the API uses Dinero.js v1.9.1 while the UI uses Dinero v2.0.0-alpha.14 -- these have **different APIs**. Dinero v2 removed `percentage()` and `divide()` in favour of `allocate()`, so the pattern that works in the UI doesn't exist in the API, and vice versa.

**Consequences:**
- API-computed totals drift from UI-computed totals by 1p
- Customer page shows "£1,979.99 - £2,159.99" but the email shows "£1,980.00 - £2,160.00"
- Customer disputes the 1p and tradesperson cannot explain it
- Unit test flakiness from floating-point comparisons

**Prevention:**
- **Single source of truth:** compute the contingency range on the API only; the UI displays whatever the API returns
- On API (dinero.js v1.9.1): use `.percentage(10)` method -- v1 has it; do not use raw math
- On UI (dinero.js v2.0.0-alpha.14): use `allocate([100, 10])` to split into a 100:10 ratio (allocate assigns remainders to the largest share, preserving precision)
- NEVER multiply by 1.10; always add a computed contingency amount to the base
- Contingency must be computed from `baseSubtotalIncTax` (not the pre-tax subtotal) to match customer expectation
- Zero-base-price guard: if `baseSubtotal === 0`, contingency is also 0 (no divide-by-zero in ratio-based allocations)
- Negative contingency impossible at the type level (slider is `0..30` step `5`; API DTO validates `@Min(0) @Max(30)`)
- Unit tests with edge cases: £0.00, £1,799.99, £1,800.00, £10.00 at 0%/5%/30%

**Detection:** Golden-file unit tests in both repos with known inputs and expected pence outputs; property-based test that computes the range both ways and asserts equality.

**Phase:** Phase 1 (Data model) and Phase 3 (Create/edit UI)

**Sources:**
- [Dinero.js v2 is out! -- Sarah Dayan](https://www.sarahdayan.com/blog/dinerojs-v2-is-out) (HIGH -- author)
- [Dinero.js v2 -- Amount concept](https://v2.dinerojs.com/docs/core-concepts/amount) (HIGH -- official)
- [Dinero.js v1 -- npm](https://www.npmjs.com/package/dinero.js/v/1.9.1) (HIGH -- official)

---

### Pitfall 6: Estimate → Quote conversion references stale or deleted data

**What goes wrong:** User converts an estimate to a quote on day 5. The conversion copies line items, customer ref, job ref, and tax rate ref into a new quote. On day 6, the user edits the source estimate (creating revision v2), or deletes an item referenced by a line item, or changes a tax rate. The quote now contains dangling references or reflects old data. Alternatively, the user double-clicks "Convert to Quote" and creates two identical quotes.

**Why it matters:** The conversion is a point-in-time snapshot contract: once converted, the quote is its own document and MUST be independent of subsequent estimate changes. Any linkage creates a quote that silently changes after the customer has agreed to it.

**Consequences:**
- Two quotes created from one double-click, customer receives two emails with two different numbers
- Quote subtotal changes after the customer has already accepted
- Orphaned references on item deletion break the quote detail page

**Prevention:**
- **Snapshot, don't reference:** the conversion denormalises line item data (name, unit price, quantity, tax rate percentage) into the quote document -- never a live reference to the estimate or to item/tax-rate collections
- Tax rate percentage copied as a literal number into the line item snapshot (same pattern as existing quote behaviour per v1.1 key decision)
- Back-link from quote → source estimate is a simple `sourceEstimateId` field, stored for audit but NOT dereferenced at read time
- **Idempotency key on conversion endpoint:** client sends a deterministic conversion key (e.g. `estimateId + revisionNumber + 'convert'`) in an `Idempotency-Key` header; API stores the resulting quote ID for 24h and returns the same quote on retry
- Double-click safety: UI disables the "Convert" button as soon as it is clicked; API enforces single-conversion via the idempotency key
- After conversion, the source estimate transitions to `Converted` status; further revisions of a converted estimate are blocked (or allowed but flagged in the audit trail as "converted estimate revised")
- Pull conversion data from the **latest revision** of the estimate, not whatever happens to be loaded in the UI

**Detection:** Integration test: convert estimate, modify source, re-read quote, assert quote unchanged. Double-convert test: fire two concurrent conversion requests with the same idempotency key, assert one quote exists.

**Phase:** Phase 5 (Conversion)

**Sources:**
- Trade Flow v1.2 denormalisation pattern (HIGH -- internal decision)
- [BullMQ Deduplication pattern](https://docs.bullmq.io/guide/jobs/deduplication) (HIGH -- general idempotency reference)

---

### Pitfall 7: DocumentType discrimination bugs let estimate pass as quote

**What goes wrong:** Because quotes and estimates share the creation UI and the line-item data model, a developer adds a feature like "mark as accepted" which is valid for quotes but not estimates. The code path uses the shared type and the new logic accidentally applies to estimates too. Or the router has `/quotes/:id` and `/estimates/:id` and a stale link produces a 404 or, worse, loads a quote into an estimate page.

**Why it matters:** Type confusion is the classic "shared abstraction" failure mode. It is the reason DDD practitioners warn against sharing models across bounded contexts. Trade Flow is small enough that sharing is pragmatic, but the sharing boundary MUST be policed at the type level.

**Consequences:**
- "Accept" button appears on an estimate
- Estimate status transitions to `Accepted` (a quote-only state) via a shared code path
- Estimate rendered as "Quote Q-2026-0045" because routing used the wrong prefix

**Prevention:**
- **Discriminated union at the type level:** `type Document = Quote | Estimate` where each has a literal `documentType: 'quote' | 'estimate'` field, and services NEVER accept the union as input -- always one or the other
- Separate collections (`quotes`, `estimates`) with separate repositories (already scoped)
- Separate RTK Query slices (`quotesApi`, `estimatesApi`) with no shared endpoints
- Separate routes: `/jobs/:jobId/quotes/:quoteId` and `/jobs/:jobId/estimates/:estimateId` with separate page components
- Shared React components (e.g. `DocumentFormDialog`) take `documentType` as a required prop and branch internally with exhaustive switch
- `never` check on the discriminator ensures adding a new document type is a compile-time error
- Status enum values prefixed or fully separate: estimate statuses include `SiteVisitRequested`, `Converted`, `Lost`; quote statuses include `Accepted`, `Rejected`. No overlap except `Draft`, `Sent`, `Viewed`, `Expired`

**Detection:** TypeScript exhaustiveness check; integration test that hits `/quotes/:estimateId` and asserts 404.

**Phase:** Phase 1 (Data model) and Phase 3 (Create/edit UI)

**Sources:** TypeScript discriminated unions pattern (HIGH -- language feature); Trade Flow v1.2 quote module pattern (HIGH -- internal)

---

## Moderate Pitfalls

### Pitfall 8: Counter collection race condition on E-YYYY-NNN generation

**What goes wrong:** Two estimates created simultaneously both read counter = 5, both write counter = 6, both get number E-2026-006. One is now orphaned / duplicated in the UI.

**Prevention:** Use MongoDB `findOneAndUpdate` with `$inc` and `upsert: true` on a dedicated `estimate_counters` collection, exactly mirroring the `quote_counters` pattern from v1.2. This is already scoped -- just ensure the pattern is copied verbatim, not reimplemented. Key is per-business-per-year: `_id: { businessId, year }`.

**Phase:** Phase 1

---

### Pitfall 9: View tracking double-counts across devices

**What goes wrong:** Customer opens the estimate on phone (view 1), then desktop (view 2), then clicks the link in the email again (view 3). `firstViewedAt` correctly fires once, but if any "view count" is persisted, it inflates to 3.

**Prevention:** Store ONLY `firstViewedAt` (set once, never overwritten via `$setOnInsert`-style query or conditional update: `{ $set: { firstViewedAt: now }, $currentDate: ... }` with a filter `{ firstViewedAt: { $exists: false } }`). Do not ship a view counter. Matches the v1.3 quote pattern.

**Phase:** Phase 2

---

### Pitfall 10: Follow-up fires after estimate expiry

**What goes wrong:** Estimate valid for 14 days. 21-day follow-up fires on day 21, nudging the customer to respond to an expired estimate. Customer clicks through and sees "This estimate has expired" -- confusing and unprofessional.

**Prevention:**
- Worker checks estimate expiry at job execution time and short-circuits if expired
- OR: only schedule follow-ups that fall before the expiry date (skip 21-day follow-up if `expiresAt < now + 21d`)
- Decide: should the 10-day follow-up also be skipped if it falls within 3 days of expiry? Product decision for Phase 4

**Phase:** Phase 4

---

### Pitfall 11: Timezone/DST handling for "3 days from send"

**What goes wrong:** Estimate sent at 23:30 on Friday. 3-day follow-up scheduled for 23:30 Monday. DST transition over the weekend shifts the actual delivery time. Or the tradesperson is in the UK but the server is UTC and the follow-up arrives at 00:30 on Tuesday local time.

**Prevention:**
- BullMQ delays are computed in UTC milliseconds -- safe from DST
- Follow-ups scheduled as relative delay (`3 * 24 * 60 * 60 * 1000`) not absolute time, so DST is moot
- If product wants "9am local time of the tradesperson's business" delivery, that's a separate feature and should be explicitly deferred to a future milestone
- Use Luxon for any human-readable timestamps in email bodies; API DTOs already use Luxon per v1.6 decision

**Phase:** Phase 4

---

### Pitfall 12: Concurrent edit race when two devices revise the same estimate

**What goes wrong:** User opens estimate on phone and desktop, edits both, saves phone first (revision 2), then saves desktop (also revision 2). Either: (a) two revision-2 documents exist, (b) one silently overwrites the other, (c) unique index conflict surfaces a cryptic 500.

**Prevention:**
- Compound unique index on `{ parentEstimateId: 1, revisionNumber: 1 }` -- second save fails cleanly
- On index conflict, API returns a friendly 409 Conflict: "This estimate was updated elsewhere. Refresh and try again."
- UI catches 409 and shows a non-destructive "Refresh to see latest changes" banner
- For solo operators (current Trade Flow user base) this is rare but not impossible (phone + desktop is common)

**Phase:** Phase 2

---

### Pitfall 13: Email spam filter triggered by "estimate" keywords

**What goes wrong:** Default estimate subject line "Your estimate from {{businessName}}" lands in spam more often than the equivalent quote template because spam filters weight "estimate" higher (it appears in lead-gen spam).

**Prevention:**
- Pre-launch: send default template through a spam-score tester (Mail Tester, GlockApps)
- Avoid all-caps, multiple exclamation marks, "FREE", in the template
- Ensure SPF/DKIM/DMARC on the Resend sending domain (already configured per v1.3)
- Allow tradesperson to customise subject line in the estimate settings template
- Include the job title or customer name in the default subject to feel personal: `Estimate for {{jobTitle}} - {{businessName}}`

**Phase:** Phase 5

---

### Pitfall 14: Customer clicks "Book site visit" after tradesperson marked as Lost

**What goes wrong:** Tradesperson marks estimate as `Lost` (customer went silent). Customer -- 3 days later -- clicks "Book site visit" from the original email. Race condition: the public page allows the action even though the state has moved on.

**Prevention:**
- Public endpoint checks current estimate status before accepting customer action
- If estimate is in a terminal state (`Lost`, `Converted`, `Declined`, `Expired`), show a friendly "This estimate is no longer active -- please contact us directly" message
- Treat customer-initiated actions as state-transition requests that can fail; UI shows the failure gracefully
- Tradesperson notified when a customer action was blocked due to state mismatch (optional, Phase 6)

**Phase:** Phase 2 / Phase 3

---

### Pitfall 15: Soft-deleted estimate follow-ups still in BullMQ

**What goes wrong:** Estimate soft-deleted (status DELETED). Scheduled follow-up jobs are not automatically cancelled. Worker fires, reads the soft-deleted estimate, sends a follow-up email for a deleted record.

**Prevention:**
- Soft-delete is a status transition like any other and MUST trigger `cancelPendingFollowups`
- Worker short-circuit at execution time if status === DELETED (defence in depth)
- Cover in the same integration test suite as Pitfall 2

**Phase:** Phase 4

---

## Minor Pitfalls

### Pitfall 16: Customer page URL format leaks entity IDs

**What goes wrong:** If the URL is `/public/estimates/65abc-revision-2` instead of a random token, customers can enumerate other estimates or reason about revision counts.

**Prevention:** Token is a cryptographically random value per v1.3 pattern; URL contains ONLY the token, no IDs. Already scoped -- just do not break the pattern.

**Phase:** Phase 2

---

### Pitfall 17: Date format confusion (UK DD/MM/YYYY vs ISO)

**What goes wrong:** Customer in the UK sees "04/10/2026" and reads it as 4 October (correct UK) vs the API serialising as "2026-04-10" and a developer assuming everything is ISO.

**Prevention:**
- All API DTOs use Luxon DateTime serialised as ISO 8601 per v1.6
- UI formats dates for display using Luxon `.toLocaleString(DateTime.DATE_FULL)` with GB locale
- Email templates MUST use the formatted display, never raw ISO
- Test with a British date in April to catch MM/DD/YYYY mix-ups visually

**Phase:** Phase 5

---

### Pitfall 18: Tracking pixel in estimate email triggers PECR/GDPR concerns

**What goes wrong:** A tracking pixel is added to the estimate email for view analytics. This is the transactional-vs-marketing PECR grey area. For purely transactional purposes (estimate delivered) this is generally fine, but the ICO updated guidance on Storage and Access Technologies (SAT) in September 2025 and January 2026, and the safer course is to avoid marketing-adjacent tracking in transactional mail.

**Prevention:**
- Do NOT add a tracking pixel to the estimate email
- "Viewed" tracking comes from the customer loading the public estimate page (not the email), which is a customer-initiated action and clearly transactional
- The `firstViewedAt` field reflects page load, not email open -- align with v1.3 quote pattern
- If email open tracking is desired later, document it as a settings-level opt-in and treat it as a future feature

**Phase:** Phase 5 (ensure template does not sneak in a pixel)

**Sources:**
- [ICO -- Guidance on direct marketing using electronic mail](https://ico.org.uk/for-organisations/direct-marketing-and-privacy-and-electronic-communications/guidance-on-direct-marketing-using-electronic-mail/) (HIGH -- regulator)
- [Transactional vs Marketing Emails under PECR](https://wdps.co.uk/transactional-vs-marketing-emails-pecr/) (MEDIUM)

---

### Pitfall 19: GDPR "right to be forgotten" vs estimate audit trail

**What goes wrong:** Customer requests deletion under UK GDPR. Estimate contains the customer's name, email, and the rendered HTML of the email we sent them. Blanket deletion removes the audit trail that protects the tradesperson in a dispute.

**Prevention:**
- Document the lawful basis for retention: contract / legitimate interest for dispute-resolution up to statutory limitation periods (6 years in England/Wales, 5 in Scotland)
- Soft-redact approach: on erasure request, replace PII fields with `[redacted]` but retain structural data (amounts, dates, status history)
- This is a data-protection policy decision, not a code change -- flag in the roadmap for product/legal review
- Not a v1.8 blocker but document the exposure

**Phase:** Flag for future milestone (data protection policy), not v1.8 code

---

### Pitfall 20: Mongoose `populate` chain creates N+1 on estimate list

**What goes wrong:** Estimate list endpoint populates customer, job, business, each line item's tax rate. At 20 estimates per page this is 80+ round-trips.

**Prevention:**
- Denormalise `customerName`, `jobTitle`, `customerEmail` into the estimate document (v1.2 pattern)
- Tax rate percentage stored literally on each line item, not as a reference (v1.1/v1.2 pattern)
- On denormalised-field source change (e.g. customer renamed), update via background job or accept eventual consistency for historical documents
- List endpoint has ZERO populate calls -- matches quote list pattern

**Phase:** Phase 1

---

### Pitfall 21: Cookie banner on public estimate page

**What goes wrong:** Public estimate page uses any non-essential cookies (analytics, preferences) and triggers the PECR cookie consent requirement. Adding a banner creates friction and dilutes the "one-click response" UX.

**Prevention:**
- Public estimate page ships with ZERO non-essential cookies
- Strictly-necessary session cookie (if any) is fine without consent
- No Google Analytics, no Hotjar, no third-party embeds
- If product wants analytics later, it must go through a consent flow -- treat as future feature

**Phase:** Phase 2

---

## Phase-Specific Warnings

| Phase Topic | Likely Pitfall | Mitigation |
|-------------|---------------|------------|
| Phase 1: Data model + E-YYYY-NNN + DocumentType | Pitfall 7 (type confusion), Pitfall 8 (counter race), Pitfall 12 (revision unique index), Pitfall 20 (N+1) | Discriminated unions at type level, copy quote counter pattern verbatim, compound unique index on `(parentEstimateId, revisionNumber)`, denormalise from day one |
| Phase 2: Revisions + public page | Pitfall 3 (stale token), Pitfall 14 (customer action vs Lost), Pitfall 16 (URL format), Pitfall 21 (cookies) | Token resolves to latest revision, state-check on customer actions, cryptographic token, zero non-essential cookies |
| Phase 3: Create/edit UI + contingency slider | Pitfall 5 (rounding), Pitfall 7 (type confusion), Pitfall 12 (concurrent edit 409) | API owns the math, discriminated props, handle 409 gracefully |
| Phase 4: Follow-up sequence (BullMQ delayed jobs) | Pitfall 2 (orphaned jobs), Pitfall 4 (Redis persistence), Pitfall 10 (post-expiry firing), Pitfall 11 (timezone), Pitfall 15 (soft-delete follow-ups) | Deterministic jobIds, production AOF, worker-level state guards, relative delays, unified cancellation helper |
| Phase 5: Email template + legal copy | Pitfall 1 (binding language), Pitfall 13 (spam filters), Pitfall 17 (date format), Pitfall 18 (tracking pixels) | Mandatory non-binding disclaimer, spam-score check, Luxon + GB locale, no tracking pixel |
| Phase 6: Conversion to Quote | Pitfall 6 (stale snapshot, double-click) | Snapshot denormalisation, `Idempotency-Key` header, convert from latest revision only |

---

## Research Flags

### Needs deeper research before implementation

1. **Default estimate email template copy:** should be reviewed by someone with UK consumer law familiarity before shipping. Not blocking v1.8 technically but blocking legal-risk-wise. Consider a /gsd:research pass in Phase 5.

2. **ICO guidance January 2026:** The ICO's formal statement on storage and access technologies is expected January 2026. Check before Phase 5 whether it affects estimate page cookies or email tracking assumptions.

3. **"Follow-up delivery time" product decision:** is 3 days "72 hours after send" or "9am the morning 3 days later"? The research above assumes relative delay (simpler, DST-safe), but product may want local-time delivery for better engagement. Phase 4 decision point.

4. **GDPR right-to-be-forgotten policy:** Not a v1.8 code task but the product should have a documented stance before v1.8 ships, because estimates materially expand the PII audit trail.

---

## Sources

**UK Legal (HIGH / MEDIUM):**
- [Consumer Rights Act 2015, Part 1 Chapter 4 (Services)](https://www.legislation.gov.uk/ukpga/2015/15/part/1/chapter/4) -- HIGH
- [Contractor Exceeds Original Quote -- Hegarty Solicitors](https://hegarty.co.uk/news/contractor-exceeds-original-quote-what-are-your-rights) -- MEDIUM
- [Estimates -- Beware! BTO Solicitors](https://www.bto.co.uk/blog/estimates-%E2%80%93-beware!.aspx) -- MEDIUM
- [Difference Between a Quote and Estimate: UK Legal Guide -- Go Legal Ai](https://go-legal.ai/difference-between-a-quote-and-estimate-uk-legal-guide/) -- MEDIUM
- [Can an Accepted Estimate Become a Binding Contract?](https://www.justanswer.com/uk-law/1o202-hello-typed-estimate-building-work-customer.html) -- LOW (forum)

**BullMQ (HIGH -- all official):**
- [BullMQ -- Going to Production](https://docs.bullmq.io/guide/going-to-production)
- [BullMQ -- Delayed Jobs](https://docs.bullmq.io/guide/jobs/delayed)
- [BullMQ -- Job Ids](https://docs.bullmq.io/guide/jobs/job-ids)
- [BullMQ -- Deduplication](https://docs.bullmq.io/guide/jobs/deduplication)
- [BullMQ -- Removing Jobs](https://docs.bullmq.io/guide/queues/removing-jobs)
- [BullMQ -- QueueScheduler (deprecated)](https://docs.bullmq.io/guide/queuescheduler)

**Dinero.js (HIGH):**
- [Dinero.js v2 is out! -- Sarah Dayan](https://www.sarahdayan.com/blog/dinerojs-v2-is-out)
- [Dinero.js v2 -- Amount concept](https://v2.dinerojs.com/docs/core-concepts/amount)
- [Dinero.js v1.9.1 on npm](https://www.npmjs.com/package/dinero.js/v/1.9.1)

**UK Data Protection (HIGH / MEDIUM):**
- [ICO -- Direct marketing using electronic mail](https://ico.org.uk/for-organisations/direct-marketing-and-privacy-and-electronic-communications/guidance-on-direct-marketing-using-electronic-mail/) -- HIGH
- [ICO -- Storage and access technologies guidance](https://ico.org.uk/for-organisations/direct-marketing-and-privacy-and-electronic-communications/guidance-on-the-use-of-storage-and-access-technologies/) -- HIGH
- [Transactional vs Marketing Emails under PECR -- WDPS](https://wdps.co.uk/transactional-vs-marketing-emails-pecr/) -- MEDIUM

**MongoDB Versioning (HIGH -- official):**
- [MongoDB -- Document Versioning design pattern](https://www.mongodb.com/docs/manual/data-modeling/design-patterns/data-versioning/document-versioning/)

**Internal (HIGH):**
- Trade Flow v1.2 key decisions (quote counter, denormalisation, tax rate resolution)
- Trade Flow v1.3 key decisions (token-based public access, QuoteSessionAuthGuard, Maizzle email templates)
- Trade Flow v1.4 key decisions (Redis `noeviction`, BullMQ worker infrastructure)
- Trade Flow v1.6 key decisions (Luxon standardisation)
