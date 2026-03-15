---
status: resolved
trigger: "items with long names are truncated in SearchableItemPicker dropdown with no way to see full name"
created: 2026-03-08T00:00:00Z
updated: 2026-03-08T00:00:00Z
---

## Current Focus

hypothesis: Two compounding issues cause truncation with no discoverability - (1) the `truncate` class on the item name span applies `text-overflow: ellipsis; overflow: hidden; white-space: nowrap`, and (2) no `title` attribute exists to reveal the full name on hover. The PopoverContent default width of `w-72` (288px) constrains available space, and the flex row layout with Badge + price leaves limited room for the name.
test: Read CSS classes and component structure
expecting: Confirm truncate class and missing title attribute
next_action: Apply fix - add title attribute and widen popover

## Symptoms

expected: Users can see the full item name in the dropdown, or at least hover to reveal it
actual: Long item names are truncated with ellipsis and there is no tooltip/title to see the full text
errors: None (visual/UX issue)
reproduction: Open SearchableItemPicker, have items with names longer than ~25 characters
started: Since component was created

## Eliminated

(none)

## Evidence

- timestamp: 2026-03-08T00:00:00Z
  checked: SearchableItemPicker.tsx line 122
  found: Item name span has `className="flex-1 truncate"` - the Tailwind `truncate` class applies `overflow: hidden; text-overflow: ellipsis; white-space: nowrap`
  implication: Long names are forcibly clipped to single line with ellipsis

- timestamp: 2026-03-08T00:00:00Z
  checked: SearchableItemPicker.tsx line 122
  found: No `title` attribute on the span element containing item.name
  implication: No way to discover full name on hover

- timestamp: 2026-03-08T00:00:00Z
  checked: popover.tsx line 31
  found: PopoverContent has default `w-72` (288px fixed width)
  implication: The popover width is narrow; after Badge and price span, very little space remains for item name

- timestamp: 2026-03-08T00:00:00Z
  checked: command.tsx line 148
  found: CommandItem uses `flex items-center gap-2` layout
  implication: The flex row with gap means Badge + price + gap consume ~120px, leaving only ~168px for the name

- timestamp: 2026-03-08T00:00:00Z
  checked: SearchableItemPicker.tsx line 102
  found: PopoverContent has `className="p-0"` but no width override
  implication: Default w-72 from popover.tsx applies, compounding the truncation

## Resolution

root_cause: Two issues combine to make long names unreadable: (1) The item name span uses `truncate` class which clips text with ellipsis and prevents wrapping, and (2) there is no `title` attribute on the span, so users cannot hover to see the full name. The PopoverContent's default `w-72` (288px) width further constrains available space.
fix: Add `title={item.name}` to the name span so the full name is visible on hover, and widen the PopoverContent to give more room for names.
verification: Lint and typecheck pass (no new errors). Visual verification needed by user.
files_changed:
  - trade-flow-ui/src/components/SearchableItemPicker.tsx
