---
status: resolved
trigger: "SearchableItemPicker dropdown list not scrollable; affects all cmdk-based dropdowns"
created: 2026-03-08T00:00:00Z
updated: 2026-03-14T00:00:00Z
---

## Current Focus

hypothesis: CONFIRMED AND VERIFIED - Radix Dialog scroll lock intercepts wheel events from Popover content. Fix: modal={true} on Popover.
test: User verified in browser
expecting: N/A - resolved
next_action: Archive session

## Symptoms

expected: User can scroll through the full list of items in SearchableItemPicker and other cmdk-based dropdowns
actual: Cannot scroll down in the item list; affects all dropdown/searchable pickers
errors: None (visual/interaction bug)
reproduction: Open any searchable picker dropdown with enough items to overflow
started: Since implementation

## Eliminated

- hypothesis: Global CSS override on overflow properties
  evidence: No global CSS in index.css or tw-animate-css targets overflow/max-height on cmdk elements
  timestamp: 2026-03-08

- hypothesis: tw-animate-css interfering with overflow
  evidence: Inspected tw-animate.css dist - contains only animation-related utilities, no overflow/height rules
  timestamp: 2026-03-08

- hypothesis: cmdk ships its own CSS that overrides Tailwind classes
  evidence: cmdk v1.1.1 ships no CSS files (no style field in package.json, no .css in dist)
  timestamp: 2026-03-08

- hypothesis: Tailwind v4 arbitrary value max-h-[300px] not generating CSS
  evidence: Same syntax max-h-[90vh] used in CustomerFormDialog.tsx and ItemFormDialog.tsx and works
  timestamp: 2026-03-08

- hypothesis: cmdk --cmdk-list-height CSS variable overriding max-height
  evidence: cmdk only sets the CSS variable on the element but does NOT apply height:var(--cmdk-list-height) - that's opt-in per docs
  timestamp: 2026-03-08

- hypothesis: CSS classes on CommandList are wrong or missing
  evidence: CommandList has correct "max-h-[300px] scroll-py-1 overflow-x-hidden overflow-y-auto" classes
  timestamp: 2026-03-08

- hypothesis: Radix Primitive.div applies conflicting inline styles
  evidence: Read @radix-ui/react-primitive source - renders plain div with spread props, no styles
  timestamp: 2026-03-08

- hypothesis: Standalone Popover issue (not Dialog-related)
  evidence: Traced SearchableItemPicker usage: BundleComponentsList -> BundleItemForm -> ItemFormDialog (a Dialog). ALL affected pickers are inside Dialogs.
  timestamp: 2026-03-08

## Evidence

- timestamp: 2026-03-08
  checked: CommandList component in command.tsx (line 83-97)
  found: Has correct classes "max-h-[300px] scroll-py-1 overflow-x-hidden overflow-y-auto"
  implication: The base CSS configuration for scrolling IS correct

- timestamp: 2026-03-08
  checked: Command component in command.tsx (line 14-28)
  found: Has "overflow-hidden" and "h-full" in className
  implication: Parent overflow-hidden creates clipping context; does not prevent child scroll

- timestamp: 2026-03-08
  checked: SearchableItemPicker usage chain
  found: Used in BundleComponentsList -> BundleItemForm -> ItemFormDialog (Dialog component)
  implication: ALL cmdk pickers in the app are inside Dialog components

- timestamp: 2026-03-08
  checked: CreateJobDialog structure
  found: Two Popover+Command instances inside a Dialog, both missing modal prop and p-0 on PopoverContent
  implication: Radix Dialog scroll lock intercepts wheel events from reaching Popover content

- timestamp: 2026-03-08
  checked: shadcn-ui/ui GitHub discussions #4175 and issues #4759, #6988
  found: Well-documented issue - Radix Dialog's react-remove-scroll captures mouse wheel events, preventing scrolling inside Popovers nested in Dialogs. Fix: add modal={true} to Popover.
  implication: This is a known Radix UI interaction issue with a documented solution

- timestamp: 2026-03-08
  checked: cmdk v1.1.1 List component source code
  found: No inline height/overflow styles applied. ResizeObserver sets --cmdk-list-height CSS variable only.
  implication: cmdk is not the cause; the CSS scroll mechanism is correct

- timestamp: 2026-03-14
  checked: Human verification of fix
  found: User confirmed scroll works correctly in cmdk-based dropdowns inside Dialog components with modal={true} on Popover
  implication: Fix is verified end-to-end

## Resolution

root_cause: Radix Dialog's scroll locking mechanism (react-remove-scroll) intercepts mouse wheel events and prevents them from reaching the Popover's CommandList scroll container. All cmdk-based pickers in the app (SearchableItemPicker and CreateJobDialog's customer/job-type pickers) are rendered inside Dialog components. The CommandList CSS (max-h-[300px] overflow-y-auto) is correct, but the Dialog's event interception prevents the native scroll from working. This is a well-documented issue in the shadcn/Radix ecosystem (shadcn-ui/ui #4175, #4759, #6988).

fix: |
  1. Added modal prop to Popover in SearchableItemPicker.tsx - makes the Popover manage its own focus/scroll context, bypassing Dialog's scroll lock
  2. Added modal prop to both Popover instances in CreateJobDialog.tsx (customer picker and job type picker)
  3. Added className="p-0" to both PopoverContent instances in CreateJobDialog.tsx (removes extra padding around Command component, matching SearchableItemPicker pattern)

verification: User confirmed fix works - scroll operates correctly in all cmdk-based dropdowns inside Dialog components
files_changed:
  - trade-flow-ui/src/components/SearchableItemPicker.tsx (added modal to Popover)
  - trade-flow-ui/src/features/jobs/components/CreateJobDialog.tsx (added modal to 2 Popovers, added p-0 to 2 PopoverContents)
