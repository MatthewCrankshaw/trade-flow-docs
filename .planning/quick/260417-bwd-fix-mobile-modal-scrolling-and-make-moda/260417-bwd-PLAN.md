---
phase: quick
plan: 260417-bwd
type: execute
wave: 1
depends_on: []
files_modified:
  - trade-flow-ui/src/components/ui/dialog.tsx
  - trade-flow-ui/src/components/ui/alert-dialog.tsx
  - trade-flow-ui/src/features/customers/components/CustomerFormDialog.tsx
  - trade-flow-ui/src/features/items/components/ItemFormDialog.tsx
autonomous: true
requirements: []
must_haves:
  truths:
    - "On mobile devices, dialogs take up the full width and height of the viewport"
    - "On mobile devices, dialog content that exceeds the screen height is scrollable"
    - "On desktop (sm+ breakpoint, 640px), dialogs remain centered with max-width and max-height constraints"
    - "On desktop, dialog content that exceeds max-height is scrollable"
    - "All existing dialog consumers continue to function without changes to their usage"
  artifacts:
    - path: "trade-flow-ui/src/components/ui/dialog.tsx"
      provides: "Mobile-fullscreen, scrollable DialogContent base component"
    - path: "trade-flow-ui/src/components/ui/alert-dialog.tsx"
      provides: "Scrollable AlertDialogContent with max-height constraint"
  key_links:
    - from: "trade-flow-ui/src/components/ui/dialog.tsx"
      to: "All *FormDialog and *Dialog consumer components"
      via: "DialogContent import"
      pattern: "import.*DialogContent.*from.*@/components/ui/dialog"
---

<objective>
Fix mobile modal scrolling and make dialogs fullscreen on mobile devices.

Purpose: On mobile, tall dialogs are currently unscrollable because the `DialogContent` uses fixed centering (`top-[50%] translate-y-[-50%]`) with no height constraint or overflow. Users cannot access form fields that extend beyond the viewport. The fix makes dialogs fullscreen on mobile (best UX for small screens) and adds proper scroll handling at the base component level so all consumers benefit automatically.

Output: Updated `dialog.tsx` and `alert-dialog.tsx` base components; cleaned up per-consumer scroll workarounds.
</objective>

<execution_context>
@.planning/quick/260417-bwd-fix-mobile-modal-scrolling-and-make-moda/260417-bwd-PLAN.md
</execution_context>

<context>
@trade-flow-ui/src/components/ui/dialog.tsx
@trade-flow-ui/src/components/ui/alert-dialog.tsx
@trade-flow-ui/src/features/customers/components/CustomerFormDialog.tsx
@trade-flow-ui/src/features/items/components/ItemFormDialog.tsx

<interfaces>
From trade-flow-ui/src/components/ui/dialog.tsx (current DialogContent classes):
```
"bg-background data-[state=open]:animate-in data-[state=closed]:animate-out data-[state=closed]:fade-out-0 data-[state=open]:fade-in-0 data-[state=closed]:zoom-out-95 data-[state=open]:zoom-in-95 fixed top-[50%] left-[50%] z-50 grid w-full max-w-[calc(100%-2rem)] translate-x-[-50%] translate-y-[-50%] gap-4 rounded-lg border p-6 shadow-lg duration-200 outline-none sm:max-w-lg"
```

Key issue: `top-[50%] translate-y-[-50%]` centers vertically but prevents scrolling when content overflows viewport.

From consumers that added their own scroll workarounds:
- CustomerFormDialog: `className="max-h-[90vh] overflow-y-auto sm:max-w-lg"`
- ItemFormDialog: `className="max-h-[90vh] overflow-y-auto sm:max-w-lg"`
</interfaces>
</context>

<tasks>

<task type="auto">
  <name>Task 1: Make DialogContent fullscreen on mobile with scroll support</name>
  <files>trade-flow-ui/src/components/ui/dialog.tsx</files>
  <action>
Update the `DialogContent` component's className in `dialog.tsx` to use a mobile-fullscreen, desktop-centered layout:

**Mobile (default, below `sm` / 640px):**
- Remove `top-[50%] left-[50%] translate-x-[-50%] translate-y-[-50%]` centering (these become desktop-only)
- Remove `max-w-[calc(100%-2rem)]` (not needed when fullscreen)
- Remove `rounded-lg border` (not needed when fullscreen)
- Add `inset-0` to fill the entire viewport
- Add `overflow-y-auto` for scrolling
- Keep `grid`, `gap-4`, `p-6`, `shadow-lg`

**Desktop (`sm:` and above):**
- Restore centered positioning: `sm:top-[50%] sm:left-[50%] sm:translate-x-[-50%] sm:translate-y-[-50%]`
- Restore `sm:max-w-lg` (default, consumers can override with className)
- Add `sm:max-h-[85vh]` to cap height on desktop
- Add `sm:inset-auto` to undo the mobile `inset-0`
- Restore `sm:rounded-lg sm:border`

The resulting className string for `DialogPrimitive.Content` should be:
```
"bg-background data-[state=open]:animate-in data-[state=closed]:animate-out data-[state=closed]:fade-out-0 data-[state=open]:fade-in-0 data-[state=closed]:zoom-out-95 data-[state=open]:zoom-in-95 fixed inset-0 z-50 grid w-full gap-4 p-6 shadow-lg duration-200 outline-none overflow-y-auto sm:inset-auto sm:top-[50%] sm:left-[50%] sm:translate-x-[-50%] sm:translate-y-[-50%] sm:max-w-lg sm:max-h-[85vh] sm:rounded-lg sm:border"
```

Also update `DialogHeader` to add `shrink-0` so it does not collapse when content scrolls.
Also update `DialogFooter` to add `shrink-0` so it stays visible at the bottom.

Do NOT change the component's TypeScript interface or props -- only the default className values.
  </action>
  <verify>
    <automated>cd /Users/mattc/Documents/projects/agent/trade-flow-docs/trade-flow-ui && npm run typecheck && npm run lint</automated>
  </verify>
  <done>
- DialogContent renders fullscreen (inset-0) on mobile with overflow-y-auto for scrolling
- DialogContent renders centered with max-height and scroll on desktop (sm+)
- DialogHeader and DialogFooter have shrink-0 to stay fixed during scroll
- All existing imports compile without changes
  </done>
</task>

<task type="auto">
  <name>Task 2: Apply same scroll fix to AlertDialogContent and clean up consumer workarounds</name>
  <files>trade-flow-ui/src/components/ui/alert-dialog.tsx, trade-flow-ui/src/features/customers/components/CustomerFormDialog.tsx, trade-flow-ui/src/features/items/components/ItemFormDialog.tsx</files>
  <action>
**alert-dialog.tsx:**
Apply the same mobile-fullscreen and desktop-centered pattern to `AlertDialogContent`:
- Replace `top-[50%] left-[50%] translate-x-[-50%] translate-y-[-50%] max-w-[calc(100%-2rem)]` with mobile-first `inset-0 overflow-y-auto`
- Add `sm:inset-auto sm:top-[50%] sm:left-[50%] sm:translate-x-[-50%] sm:translate-y-[-50%] sm:max-h-[85vh] sm:rounded-lg sm:border`
- Remove `rounded-lg border` from base (add to `sm:` variant)
- Keep `sm:max-w-lg` as before

The resulting className for `AlertDialogPrimitive.Content`:
```
"bg-background data-[state=open]:animate-in data-[state=closed]:animate-out data-[state=closed]:fade-out-0 data-[state=open]:fade-in-0 data-[state=closed]:zoom-out-95 data-[state=open]:zoom-in-95 fixed inset-0 z-50 grid w-full gap-4 p-6 shadow-lg duration-200 overflow-y-auto sm:inset-auto sm:top-[50%] sm:left-[50%] sm:translate-x-[-50%] sm:translate-y-[-50%] sm:max-w-lg sm:max-h-[85vh] sm:rounded-lg sm:border"
```

Also update `AlertDialogHeader` and `AlertDialogFooter` to add `shrink-0`.

**CustomerFormDialog.tsx:**
- Change `className="max-h-[90vh] overflow-y-auto sm:max-w-lg"` to `className="sm:max-w-lg"` (remove `max-h-[90vh] overflow-y-auto` since the base now handles it)

**ItemFormDialog.tsx:**
- Same change: remove `max-h-[90vh] overflow-y-auto` from the DialogContent className, keeping only the `sm:max-w-lg` if present
  </action>
  <verify>
    <automated>cd /Users/mattc/Documents/projects/agent/trade-flow-docs/trade-flow-ui && npm run ci</automated>
  </verify>
  <done>
- AlertDialogContent has mobile-fullscreen and desktop-centered layout matching DialogContent
- CustomerFormDialog and ItemFormDialog no longer have per-consumer scroll workarounds
- `npm run ci` passes (tests, lint, format, typecheck)
  </done>
</task>

<task type="checkpoint:human-verify" gate="blocking">
  <what-built>Mobile-fullscreen dialogs with scroll support applied to all dialog base components</what-built>
  <how-to-verify>
    1. Open the app on a mobile device or use Chrome DevTools device toolbar (toggle with Ctrl+Shift+M / Cmd+Shift+M)
    2. Set viewport to a small mobile size (e.g., iPhone SE, 375x667)
    3. Navigate to Customers page and tap "Add Customer" -- the dialog should fill the entire screen and the form should be scrollable if content exceeds viewport height
    4. Expand the "Billing Address" section to make the form taller -- verify you can scroll down to see all fields and the footer buttons
    5. Navigate to Items and tap "Add Item" -- same fullscreen + scroll behavior
    6. Open any dialog on desktop width (>640px) -- verify it appears centered with rounded corners and border, not fullscreen
    7. Test alert dialogs (e.g., delete confirmation) on both mobile and desktop
  </how-to-verify>
  <resume-signal>Type "approved" or describe any issues with the mobile/desktop dialog behavior</resume-signal>
</task>

</tasks>

<verification>
- `npm run ci` passes in trade-flow-ui (tests, lint, format, typecheck)
- On mobile viewport: dialogs fill entire screen, content scrolls, header/footer stay visible
- On desktop viewport: dialogs centered, max-width constrained, rounded corners, scrollable if content exceeds 85vh
- No consumer components required changes to their DialogContent usage (except removing now-redundant scroll workarounds)
</verification>

<success_criteria>
- All dialogs across the app are scrollable on mobile when content exceeds viewport height
- All dialogs render fullscreen on mobile (< 640px) for optimal touch UX
- All dialogs retain centered, constrained layout on desktop (>= 640px)
- CI passes with zero failures
</success_criteria>

<output>
After completion, create `.planning/quick/260417-bwd-fix-mobile-modal-scrolling-and-make-moda/260417-bwd-SUMMARY.md`
</output>
