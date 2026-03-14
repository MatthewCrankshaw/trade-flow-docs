---
status: diagnosed
trigger: "Line item delete is a hard delete from the database instead of a soft delete (setting status to deleted)"
created: 2026-03-14T20:30:00Z
updated: 2026-03-14T20:35:00Z
---

## Current Focus

hypothesis: CONFIRMED - Repository.delete() calls writer.deleteOne() (hard delete) instead of updating status
test: Read repository and service code
expecting: Hard delete operation
next_action: Return diagnosis

## Symptoms

expected: Deleting a line item soft-deletes it (status set to "deleted") rather than hard-deleting from the database
actual: The item is immediately removed and the quote totals update, but the item is hard deleted from the database
errors: None reported
reproduction: Test 7 in UAT - delete a line item and check database
started: Discovered during UAT

## Eliminated

## Evidence

- timestamp: 2026-03-14T20:32:00Z
  checked: QuoteLineItemRepository.delete() method (line 83-87)
  found: Uses writer.deleteOne() which permanently removes the document from MongoDB
  implication: This is the direct cause - hard delete instead of soft delete

- timestamp: 2026-03-14T20:32:00Z
  checked: QuoteLineItemRepository.deleteByParentLineItemId() method (line 89-95)
  found: Uses writer.deleteMany() for cascade bundle deletion - same hard delete problem
  implication: Bundle component cascade deletion also hard deletes

- timestamp: 2026-03-14T20:33:00Z
  checked: QuoteUpdater.deleteLineItem() service method (lines 99-125)
  found: Calls repository.delete() and repository.deleteByParentLineItemId() directly, no status update logic
  implication: Service layer has no soft delete logic either

- timestamp: 2026-03-14T20:34:00Z
  checked: QuoteLineItemStatus enum
  found: Only has PENDING, APPROVED, REJECTED - no DELETED status value exists
  implication: The enum needs a DELETED value added

- timestamp: 2026-03-14T20:34:00Z
  checked: QuoteTotalsCalculator.calculateTotals() and findAllByQuoteId()
  found: Neither filters by status - they process all line items for a quote
  implication: After adding soft delete, queries must filter out deleted items

## Resolution

root_cause: Three-layer problem: (1) QuoteLineItemRepository.delete() and deleteByParentLineItemId() use hard delete operations (deleteOne/deleteMany) instead of updating status to DELETED. (2) QuoteLineItemStatus enum lacks a DELETED value. (3) findAllByQuoteId() and QuoteTotalsCalculator do not filter by status, so even after adding soft delete, deleted items would still appear in queries and totals.
fix:
verification:
files_changed: []
