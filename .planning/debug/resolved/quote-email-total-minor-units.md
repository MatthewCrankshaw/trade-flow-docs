---
status: resolved
trigger: "Quote email shows $43200.00 instead of $432.00 — minor units displayed as dollars without /100"
created: 2026-03-21T00:00:00Z
updated: 2026-03-21T00:00:00Z
---

## Current Focus

hypothesis: CONFIRMED — Frontend sends minor units (pence) but API treated them as major units (pounds), inflating all stored values by 100x
test: Fixed API to accept minor units from frontend and return minor units in responses
expecting: Email total now correctly formatted; public quote page shows correct totals
next_action: Await human verification

## Symptoms

expected: Quote email should show "$432.00" (the correct dollar amount)
actual: Quote email shows "$43200.00" — the minor unit (cents) value formatted as if it were dollars
errors: No runtime errors — the value is just wrong (off by factor of 100)
reproduction: Send a quote to a customer via email; the total in the email body is 100x too large
started: Likely introduced in Phase 18 (quote email sending) or Phase 19 (customer response)

## Eliminated

- hypothesis: Email template missing /100 division
  evidence: Email sender (line 49) correctly calls toMajorUnits().toFixed(2) — the issue is upstream (stored values are already 100x too large)
  timestamp: 2026-03-21

- hypothesis: Money value object toMajorUnits() is broken
  evidence: Traced through toDinero() and toUnit() — conversion math is correct for the stored _amount value
  timestamp: 2026-03-21

- hypothesis: QuoteTotalsCalculator computes wrong totals
  evidence: Calculator correctly sums lineItem.unitPrice.multiply(quantity) — but the unitPrice itself is already 100x too large
  timestamp: 2026-03-21

## Evidence

- timestamp: 2026-03-21
  checked: Frontend useCurrencyConversion hook (useCurrencyConversion.ts)
  found: fromInput() calls inputToCurrency() which returns minor units (pence). E.g. user enters "4.32" -> returns 432.
  implication: Frontend sends 432 (pence) to API for a 4.32 pound item

- timestamp: 2026-03-21
  checked: API item controller mapper (map-create-item-request-to-dto.utility.ts)
  found: Uses Money.fromMajorUnits(request.defaultPrice, DEFAULT_CURRENCY) — treats incoming 432 as pounds, not pence
  implication: Backend stores 432 "pounds" when user intended 4.32 pounds (432 pence). All values are 100x inflated.

- timestamp: 2026-03-21
  checked: Frontend formatAmount in useBusinessCurrency hook
  found: formatAmount() calls createMoney(amount, currencyCode) which treats input as minor units (pence)
  implication: Frontend displays API's "432 pounds" as "432 pence = 4.32 pounds" — appears correct to user but masks the backend inflation

- timestamp: 2026-03-21
  checked: Email sender (quote-email-sender.service.ts:49)
  found: Formats total as `$${quote.totals.total.toMajorUnits().toFixed(2)}` — uses inflated backend value directly
  implication: Email shows $43200.00 because backend thinks total is 43200 pounds (when it should be 432 pounds)

- timestamp: 2026-03-21
  checked: PublicQuoteRetriever (public-quote-retriever.service.ts)
  found: Bypasses QuoteTotalsCalculator — returns zero totals from QuoteRepository.toDto()
  implication: Secondary bug — public quote page shows $0.00 for all totals

- timestamp: 2026-03-21
  checked: All 303 API tests pass after fix, frontend typecheck and lint clean
  found: No regressions
  implication: Fix is safe to deploy

## Resolution

root_cause: Frontend-backend unit mismatch. The frontend sends prices in minor units (pence/cents) via inputToCurrency(), but the backend API controllers created Money objects using Money.fromMajorUnits(), treating those values as major units (pounds/dollars). This inflated all stored monetary values by 100x. The frontend masked this by also treating API response values as minor units when displaying (formatAmount uses createMoney which expects minor units), so the UI appeared correct. But when the backend directly formatted amounts (e.g., for the email template with toMajorUnits().toFixed(2)), the 100x inflation was exposed. Secondary bug: PublicQuoteRetriever bypassed QuoteTotalsCalculator, showing zero totals on the public quote page.

fix: |
  1. Changed API controllers to use Money.fromMinorUnits() instead of Money.fromMajorUnits() when accepting price values from requests (items create/update mappers, quote line item updater)
  2. Changed API controllers to return toMinorUnits() instead of toMajorUnits() in responses (quote controller totals + line items, item response mapper, public quote retriever)
  3. Fixed QuoteUpdater.updateLineItem() to use Money objects for arithmetic instead of raw numbers, converting incoming minor units properly
  4. Fixed PublicQuoteRetriever to calculate totals via QuoteTotalsCalculator (was returning zeros)
  5. Exported QuoteTotalsCalculator from QuoteModule so QuoteTokenModule can use it
  6. Updated PublicQuoteCard frontend to use format (minor unit formatter) instead of formatDecimal
  7. Fixed pre-existing test issues: controller spec missing QuoteResponseHandler mock, config mock using wrong key, test using wrong currency

verification: All 303 API tests pass. Frontend typecheck and lint clean.

files_changed:
  - trade-flow-api/src/item/controllers/mappers/map-create-item-request-to-dto.utility.ts
  - trade-flow-api/src/item/controllers/mappers/merge-existing-item-with-changes.utility.ts
  - trade-flow-api/src/item/controllers/mappers/map-item-to-response.utility.ts
  - trade-flow-api/src/quote/controllers/quote.controller.ts
  - trade-flow-api/src/quote/services/quote-updater.service.ts
  - trade-flow-api/src/quote/quote.module.ts
  - trade-flow-api/src/quote-token/services/public-quote-retriever.service.ts
  - trade-flow-api/src/quote-token/test/services/public-quote-retriever.service.spec.ts
  - trade-flow-api/src/quote-token/test/controllers/public-quote.controller.spec.ts
  - trade-flow-api/src/quote/test/services/quote-email-sender.service.spec.ts
  - trade-flow-ui/src/features/public-quote/components/PublicQuoteCard.tsx
  - trade-flow-ui/src/features/public-quote/types/public-quote.types.ts
