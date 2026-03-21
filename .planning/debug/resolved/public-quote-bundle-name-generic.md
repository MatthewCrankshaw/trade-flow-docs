---
status: resolved
trigger: "public-quote-bundle-name-generic"
created: 2026-03-21T00:00:00Z
updated: 2026-03-21T00:00:00Z
---

## Current Focus

hypothesis: CONFIRMED — line 100 of public-quote-retriever.service.ts sets `name: lineItem.unit` which uses the unit of measurement (e.g., "Bundle", "each") instead of the actual item name
test: Verified by reading IQuoteLineItemDto (no name field) and IItemDto (has name field)
expecting: Fix by injecting ItemRepository and looking up item names
next_action: Apply fix — inject ItemRepository, look up item names, update mapping

## Symptoms

expected: Bundle line items should show their actual item name (e.g., "Painting Bundle")
actual: All bundle line items show "name": "Bundle" in the API response
errors: No errors — just wrong data being returned
reproduction: View any quote with a bundle line item via GET /v1/public/quote/:token
started: Likely since public quote retriever was created (Phase 19)

## Eliminated

## Evidence

- timestamp: 2026-03-21T00:01:00Z
  checked: public-quote-retriever.service.ts line 100
  found: `name: lineItem.unit` — sets name to the unit field (e.g., "Bundle", "each", "hour")
  implication: This is the direct cause — unit is not the item name

- timestamp: 2026-03-21T00:02:00Z
  checked: IQuoteLineItemDto interface
  found: No `name` field exists — only `itemId`, `unit`, `type`, etc.
  implication: Item name must be fetched from the Item entity via itemId

- timestamp: 2026-03-21T00:03:00Z
  checked: IItemDto interface
  found: Has `name` field (the actual item name) and `unit` field (unit of measurement)
  implication: The fix needs to look up items by their IDs and use `item.name`

## Resolution

root_cause: public-quote-retriever.service.ts line 100 sets `name: lineItem.unit` — the unit of measurement (e.g., "Bundle") — instead of looking up the actual item name from the Item entity via lineItem.itemId
fix: Inject ItemRepository, batch-fetch item names for all line items, use item.name in the mapping
verification: All 8 tests pass (3 new tests specifically verify item name mapping). Typecheck passes.
files_changed:
  - trade-flow-api/src/quote-token/services/public-quote-retriever.service.ts
  - trade-flow-api/src/quote-token/test/services/public-quote-retriever.service.spec.ts
  - trade-flow-api/src/item/item.module.ts
  - trade-flow-api/src/quote-token/quote-token.module.ts
