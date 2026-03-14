---
status: diagnosed
trigger: "Tax is not being calculated for component-based bundles"
created: 2026-03-14T20:30:00Z
updated: 2026-03-14T20:45:00Z
---

## Current Focus

hypothesis: Money.percentageOf() has a math bug causing it to return 0 for most inputs
test: Traced the calculation path from bundle creation through totals
expecting: percentageOf divides Dinero by minor units integer, causing near-zero results
next_action: Return diagnosis

## Symptoms

expected: Tax is calculated for component-based bundles
actual: Tax is not being calculated for component-based bundles
errors: None reported
reproduction: Test 10 in UAT - add a component-based bundle and check tax values
started: Discovered during UAT for phase 14

## Eliminated

## Evidence

- timestamp: 2026-03-14T20:35:00Z
  checked: QuoteTotalsCalculator.calculateTotals() - how bundle tax is computed at quote level
  found: Skips component line items (parentLineItemId check), calculates tax for parent using lineItem.taxRate
  implication: Parent bundle's taxRate must be correct for tax to appear in totals

- timestamp: 2026-03-14T20:37:00Z
  checked: QuoteBundleLineItemFactory.buildParentLineItem() - how parent taxRate is set
  found: effectiveTaxRate = parentTaxAmount.percentageOf(parentLineTotal) on line 135
  implication: percentageOf must return the correct percentage for tax to work

- timestamp: 2026-03-14T20:40:00Z
  checked: Money.percentageOf() implementation (line 121-128)
  found: "this.toDinero().divide(base.toMinorUnits()).multiply(100)" -- divides a Dinero object by base.toMinorUnits() which is an integer in minor units (e.g. 10000 for $100). This means a $20 tax on $100 base computes as Dinero($20)/10000*100 = Dinero($0.002)*100 which rounds to 0, not the expected 20.
  implication: This is the root cause. percentageOf always returns 0 (or near-zero) for realistic monetary values.

- timestamp: 2026-03-14T20:42:00Z
  checked: Where percentageOf is used in the codebase
  found: Only used in quote-bundle-line-item-factory.service.ts line 135
  implication: Bug is isolated to bundle tax rate calculation path

## Resolution

root_cause: Money.percentageOf() method has a math bug. It divides a Dinero instance by base.toMinorUnits() (an integer count of minor currency units), causing extreme precision loss. For example, $20 tax on a $100 base: Dinero(2000 cents).divide(10000).multiply(100) = 0 instead of 20. This causes the parent bundle line item to be saved with taxRate=0, and QuoteTotalsCalculator then computes zero tax for the bundle.
fix:
verification:
files_changed: []
