---
status: diagnosed
trigger: "Bundle component rows have non-uniform column alignment for amount, unit price, and total when there are many components"
created: 2026-03-14T20:30:00Z
updated: 2026-03-14T20:30:00Z
---

## Current Focus

hypothesis: CONFIRMED - Bundle components use flex divs with no fixed widths instead of table rows with column alignment
test: Read QuoteLineItemsTable.tsx lines 266-312
expecting: Component layout differs from parent table layout
next_action: Return root cause diagnosis

## Symptoms

expected: Bundle component rows display with uniform column alignment matching the parent table
actual: The bundle component aren't layed out in a very visually pleasing way, the amount and unit, price and total are horizontally aligned in a way that is not uniform when there are lots of components.
errors: None reported
reproduction: Test 3 in UAT - add a bundle item and view expanded components
started: Discovered during UAT

## Eliminated

## Evidence

- timestamp: 2026-03-14T20:32:00Z
  checked: QuoteLineItemsTable.tsx lines 266-312 (expanded bundle component rendering)
  found: Components render in a single colSpan TableCell as flex divs with gap-4 and no fixed column widths. The name uses flex-1, but type badge, quantity, unit price, tax rate, and total are all auto-sized spans with no width constraints.
  implication: When component names or values vary in length, all downstream columns shift horizontally, causing non-uniform alignment across rows.

- timestamp: 2026-03-14T20:32:00Z
  checked: QuoteLineItemsTable.tsx lines 133-146 (parent table header)
  found: Parent table uses actual TableHead elements with explicit widths (w-20 for Qty, w-28 for Unit Price, w-10 for actions) and text-right alignment on Line Total.
  implication: Parent rows align because the table enforces column widths. Component rows bypass this entirely by using a full-width colSpan cell with flex layout inside.

## Resolution

root_cause: Bundle component rows (lines 266-312 in QuoteLineItemsTable.tsx) render inside a single full-colSpan TableCell using flex divs with gap-4 and no fixed column widths. Unlike the parent table which enforces column alignment via TableHead widths (w-20, w-28, etc.), the component layout relies on content-dependent flex sizing. This means columns like unit price, tax rate, and line total shift horizontally based on the width of component names and quantity text, producing non-uniform alignment when there are many components with varying content lengths.
fix:
verification:
files_changed: []
