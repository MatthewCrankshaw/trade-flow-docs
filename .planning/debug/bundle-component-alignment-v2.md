---
status: diagnosed
phase: 14
trigger: "Bundle component rows desktop view — type, quantity, unit, price, tax rate, total columns do not line up with parent table columns"
created: 2026-03-14T00:00:00Z
updated: 2026-03-14T00:00:00Z
---

## Current Focus

hypothesis: confirmed — see Resolution
test: static code analysis of TableHead definitions vs grid template
expecting: n/a (diagnosis complete)
next_action: n/a

## Symptoms

expected: Bundle component rows (expanded) align their Type, Qty, Unit, Unit Price, Tax %, and Line Total cells with the corresponding parent table header columns
actual: The component row columns are visually offset and do not align with the parent table headers
errors: none (visual layout bug only)
reproduction: Expand a bundle line item in the quote detail view on desktop; observe component rows vs table headers
started: introduced in phase 14 plan 14-04 (flex -> CSS grid refactor)

## Evidence

- timestamp: 2026-03-14
  checked: TableHeader / TableHead elements in QuoteLineItemsTable.tsx (lines 136-145)
  found: >
    Parent table has 7 data columns (+ 1 optional delete column):
    1. Item          — no explicit width (table auto-sizes)
    2. Type          — no explicit width
    3. Qty           — w-20 (5rem / 80px)
    4. Unit          — no explicit width
    5. Unit Price    — w-28 (7rem / 112px)
    6. Tax %         — no explicit width
    7. Line Total    — no explicit width, text-right
    8. (Delete btn)  — w-10 (2.5rem / 40px), editable mode only
  implication: these 7-8 columns are rendered by the native <table> layout engine

- timestamp: 2026-03-14
  checked: component row grid definition (lines 284-313)
  found: >
    gridTemplateColumns: "1fr auto 5rem auto 7rem auto auto"
    That is 7 grid tracks mapping to these children:
    track 1 — <span> componentName           (1fr)
    track 2 — <Badge> type label             (auto)
    track 3 — "qty x unit" in one span       (5rem)
    track 4 — empty <span /> spacer          (auto)
    track 5 — unitPrice                      (7rem)
    track 6 — taxRate                        (auto)
    track 7 — lineTotal                      (auto)
  implication: the grid attempts to mirror the 7 parent columns but the mapping is wrong

- timestamp: 2026-03-14
  checked: grid container placement (lines 267-272)
  found: >
    The component rows sit inside:
      <TableRow>
        <TableCell colSpan={isEditable ? 8 : 7} className="py-3">
          <div className="ml-6 space-y-2">
            <div className="grid ..." style={gridTemplateColumns}>
    The ml-6 class applies 1.5rem (24px) of left margin to the wrapper div,
    shifting the entire grid's left edge 24px to the right of the TableCell's
    content edge.
  implication: even if column widths matched exactly, the 24px left indent
    pushes every column rightward, misaligning track 1 from the Item header

## Eliminated

- hypothesis: widths for Unit Price column are wrong
  evidence: parent uses w-28 = 7rem; grid track 5 is also 7rem — these match
  timestamp: 2026-03-14

- hypothesis: widths for Qty column are wrong
  evidence: parent uses w-20 = 5rem; grid track 3 is also 5rem — these match as
    individual values; however, the structural issue (see root cause) makes the
    positional alignment wrong regardless
  timestamp: 2026-03-14

## Resolution

root_cause: >
  There are two compounding structural misalignment problems:

  PROBLEM 1 — Qty and Unit are merged into a single grid track.
    The parent table has separate columns for Qty (col 3, w-20 = 5rem) and
    Unit (col 4, no explicit width). The component row grid collapses both into
    a single track: "{component.quantity} x {component.unit}" rendered in one
    <span> with a 5rem track width. The compensating empty <span /> spacer in
    track 4 uses "auto", which collapses to near-zero because it has no content.
    This means:
      - Grid tracks 1-2 (Name, Type) can potentially align with parent cols 1-2
      - Grid track 3 (5rem merged Qty+Unit) cannot span both parent col 3 AND
        col 4 simultaneously — it only occupies 5rem while the actual two-column
        span is 5rem + (Unit column width)
      - Everything from track 4 onward (spacer, Unit Price, Tax %, Total) is
        shifted leftward relative to where the parent columns actually are

  PROBLEM 2 — The ml-6 (1.5rem / 24px) indent on the container div.
    The grid <div> is wrapped in a parent <div className="ml-6 space-y-2">.
    This adds 24px of left margin inside the TableCell, moving the grid's
    left edge away from the table's left column edge. Column 1 of the grid
    (the name track) therefore starts 24px to the right of the Item header.
    All subsequent tracks are offset by this same 24px, in addition to any
    width-based misalignment from Problem 1.

  COMBINED EFFECT:
    Every component row column is:
    (a) shifted right by 24px from the indent (Problem 2), AND
    (b) further misaligned from Unit Price onward because the Qty+Unit merge
        does not consume the same total horizontal space as the two separate
        parent columns (Problem 1).

  The approach of using a free-standing CSS grid inside a colSpan cell to
  achieve column alignment with the parent <table> is fundamentally difficult
  because table column widths are determined by the browser's table layout
  algorithm at runtime — they cannot be reliably predicted or matched with
  static grid track values.

fix: (not applied — diagnosis only)
verification: (not applied — diagnosis only)
files_changed: []
