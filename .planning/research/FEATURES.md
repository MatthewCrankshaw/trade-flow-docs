# Feature Landscape

**Domain:** Bundle management and quote creation for sole-trader business management
**Researched:** 2026-03-08

## Table Stakes

Features users expect. Missing = product feels incomplete.

### Bundle Management

| Feature | Why Expected | Complexity | Dependencies | Notes |
|---------|--------------|------------|--------------|-------|
| Fix bundle creation bug (unit field) | Broken CRUD is a blocker; users cannot create bundles at all | Low | BundleItemForm, API item creation | Unit field defaults to "bundle" instead of respecting component units. Likely a hardcoded default in form or API. |
| Component editing on existing bundles | Currently read-only in edit mode with a "coming soon" message. Users who make mistakes or need to update bundles have no recourse. | Medium | BundleComponentsList (mode="edit" path), API update endpoint for bundleConfig | The `isReadOnly = mode === "edit"` guard in BundleComponentsList needs removal. Requires API support for PATCH/PUT on bundleConfig.components. Form currently skips bundleConfig on edit (`if (!isEditMode)`). |
| Searchable item dropdown for component selection | Current Select dropdown is a flat list that becomes unusable with 20+ items. Tradespeople build item catalogues of 50-200+ items. | Medium | Command+Popover pattern (exists in CreateJobDialog), items API | Reuse the exact Command+Popover combobox pattern from CreateJobDialog. Filter to non-bundle, active items. Show item type badge + unit + price in each option row. |
| Improved bundle component display | Current "bullet + name x qty unit" is functional but lacks price visibility and visual hierarchy | Low | ItemsTable expanded row, currency formatting (useBusinessCurrency exists) | Show per-component line total (qty x defaultPrice). Add subtotal row. Display-only change to existing expandable row in ItemsTable. |

### Quote UI -- API Integration

| Feature | Why Expected | Complexity | Dependencies | Notes |
|---------|--------------|------------|--------------|-------|
| Quote listing from API | Currently hardcoded mock data. Users see fake quotes that cannot be interacted with. | Low | QuotesDataView, QuotesTable, RTK Query (pattern established across all features) | Create quoteApi.ts with RTK Query endpoints matching existing API routes: GET business/:businessId/quotes, GET quote/:quoteId. Follow itemApi.ts pattern exactly. |
| Quote creation dialog | Users need to create quotes to use the feature at all | Medium | Customer selector (reuse Command+Popover from CreateJobDialog), quote API POST endpoint exists | Dialog with: customer picker (searchable), title, optional notes, optional validUntil date. Submit creates draft quote via API. Auto-generate title from customer name like jobs do. |
| Quote detail view | After creating a quote, users need to see it with its line items, totals, and status | Medium | Quote GET endpoint exists, line item response includes parent/child hierarchy | Page or sheet showing: header (number, title, status badge, customer, dates), line items table, totals summary (subtotal, tax, total). |
| Add line items to quote | A quote with no line items is useless. Users need to add items/bundles from their catalogue. | Medium | POST quote/:quoteId/line-item endpoint exists, item picker (same Command+Popover pattern), quote detail refresh | Item picker + quantity input. On submit, POST to add-line-item endpoint. Refresh quote detail to show new line item with calculated pricing. |
| Bundle line items on quotes (expandable) | When a bundle is added as a line item, users need to see what is included without cluttering the main line items list | Medium | API already returns parent/child hierarchy with `components` array on bundle line items, expandable row pattern exists in ItemsTable | Show bundle as single rolled-up row (name, total price). Chevron expands to show component breakdown indented below. Reuse the Fragment + expanded TableRow pattern from ItemsTable. |
| Quote totals display | Users need to see subtotal, tax, and total. This is fundamental to any quoting tool. | Low | API already calculates and returns totals object with subTotal, taxTotal, total | Display in a summary section at the bottom of the quote detail view. Format with useBusinessCurrency. |

## Differentiators

Features that set product apart. Not expected in v1.2, but add clear value.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Inline quantity editing on quote line items | Edit quantity directly in the line items table without opening a dialog. Faster workflow for tradespeople adjusting quantities on-site. | Medium | Would need a PATCH endpoint for line item quantity (does not exist yet). Defer to next iteration. |
| Quote duplication | Tradespeople often quote similar jobs. "Duplicate quote" saves significant time vs re-creating from scratch. | Low | API would need a clone endpoint. Pure convenience feature. |
| Quick-add from recent items | Show recently used items at the top of the item picker. Tradespeople quote the same 10-15 items repeatedly. | Low | Client-side tracking of recently selected items, or API endpoint for usage frequency. Adds to the Command+Popover item picker. |
| Bundle price override on quote | Allow overriding the component-based calculated price with a fixed amount for a specific quote (e.g., discount the whole bundle). | Medium | Would need discount/override field on bundle parent line item. API discount infrastructure exists (discountAmount field) but may not be wired to UI. |
| PDF/print quote export | Tradespeople need to send quotes to customers. Currently no way to share. | High | Requires PDF generation (server-side or client-side), template design, branding. Major feature on its own. Candidate for next milestone. |
| Quote notes per line item | Add context to individual line items (e.g., "Rinnai Infinity 26 -- customer prefers this model") | Low | Would need notes field on line item entity (does not exist). Minor API + UI change. |

## Anti-Features

Features to explicitly NOT build for v1.2.

| Anti-Feature | Why Avoid | What to Do Instead |
|--------------|-----------|-------------------|
| Quote acceptance/rejection workflow | Explicitly out of scope per PROJECT.md. Adds status transition complexity, notification logic, and customer-facing concerns. | Keep quotes in draft status only for v1.2. Status field exists in the model for future use. |
| Drag-and-drop line item reordering | Over-engineering for the number of line items a tradesperson typically has (5-15). Adds complexity with bundle parent/child relationships. | Use the natural order from API (creation order). Add sort-by-type later if needed. |
| Complex pricing rules (tiered, volume, conditional) | CPQ-level pricing is enterprise software. Tradespeople use simple unit price x quantity. | Use existing flat pricing. Component-based bundle pricing already handles the main use case. |
| Customer-facing quote view/portal | Out of scope per PROJECT.md. Requires auth, branding, acceptance workflow. | Future milestone candidate alongside PDF export. |
| Multi-currency quotes | Solo tradespeople operate in one currency. Adding currency selection adds unnecessary complexity. | Use business default currency (already established pattern). |
| Line item grouping/sections | Grouping line items into sections (e.g., "Labour", "Materials") adds UI complexity without proportional value for small quotes. | Bundle items already provide logical grouping. Keep flat list with type badges. |
| Optional bundle component selection on quotes | The API supports `isOptional` on bundle components, but building UI for customers to select/deselect optional components adds significant complexity. | Always include all components for v1.2. Optional component UI is a future differentiator. |
| Direct component line item editing on quotes | Modifying individual component quantities/prices within a bundle on a quote breaks the bundle's integrity and causes pricing inconsistencies. | All changes to bundle line items should go through the parent bundle. Matches commercetools best practice: restrict direct modifications to component line items. |

## Feature Dependencies

```
Fix bundle creation bug
  (no dependencies -- do first, unblocks all bundle features)

Component editing on existing bundles
  --> Requires: Fix bundle creation bug (establishes correct bundle data)
  --> Requires: API PATCH/PUT for bundleConfig (verify exists or build)

Searchable item dropdown for bundles
  --> Requires: Command+Popover pattern (exists in jobs feature)
  --> Can be built independently of other bundle features

Improved bundle component display
  --> Requires: Existing expandable row pattern (exists in ItemsTable)
  --> Can be built independently

Quote API integration (listing)
  --> Requires: RTK Query setup for quotes (quoteApi.ts -- does not exist yet)
  --> Unblocks all other quote UI features

Quote creation dialog
  --> Requires: Quote API integration (for createQuote mutation)
  --> Requires: Customer data (useGetCustomersQuery exists)

Quote detail view
  --> Requires: Quote API integration (for getQuote query)
  --> Requires: Quote creation (need quotes to view)

Add line items to quote
  --> Requires: Quote detail view (need a place to display them)
  --> Requires: Searchable item dropdown (same picker component, reuse)
  --> Requires: Items API (exists)

Bundle line items on quotes (expandable)
  --> Requires: Add line items to quote
  --> Requires: Improved bundle component display pattern (same expand/collapse)

Quote totals display
  --> Requires: Quote detail view
  --> API already calculates totals
```

## MVP Recommendation

### Must-have for v1.2 (in build order):

1. **Fix bundle creation bug** -- unblocks everything, low effort
2. **Quote API integration (RTK Query)** -- unblocks all quote UI work
3. **Searchable item dropdown** -- needed for both bundle component editing AND quote line item adding; build as reusable component once
4. **Component editing on existing bundles** -- completes bundle CRUD
5. **Improved bundle component display** -- polish existing feature
6. **Quote creation dialog** -- enables quote workflow
7. **Quote detail view with totals** -- shows created quotes
8. **Add line items to quote** -- makes quotes useful
9. **Bundle line items on quotes (expandable)** -- completes bundle-quote integration

### Defer to v1.3+:

- **PDF/print export** -- high complexity, needs design work, but is the most requested feature for quoting tools in the trades space
- **Quote acceptance/rejection workflow** -- explicitly deferred per PROJECT.md
- **Inline quantity editing** -- nice-to-have, not blocking
- **Quote duplication** -- convenience, not blocking

### Reusable component opportunity:

The **searchable item picker** (Command+Popover with item search, type badges, price display) should be built as a shared component since it is needed in:
- Bundle component selection (replacing current flat Select dropdown)
- Quote line item addition (new feature)
- Potentially future features (invoice line items, purchase orders)

The pattern already exists in CreateJobDialog for customers and job types. The item picker variant needs: type icon/badge per item, unit display, default price display, filter to non-bundle active items only.

## Sources

- Codebase analysis: BundleItemForm.tsx, BundleComponentsList.tsx, ItemsTable.tsx, CreateJobDialog.tsx, QuotesTable.tsx, QuotesDataView.tsx (existing patterns and current limitations)
- Codebase analysis: quote.controller.ts, quote-line-item.entity.ts, bundle-config.dto.ts (API capabilities and data model)
- [Extensiv Bundle/Kit UI](https://help.extensiv.com/om-products/creating-bundleskits-through-the-ui) -- bundle component editing patterns (add/remove/edit quantity)
- [commercetools Bundle Management](https://docs.commercetools.com/foundry/best-practice-guides/product-bundles) -- restrict direct component line item modifications, apply changes via parent
- [Trax Group ERP Bundle Issues](https://traxgroup.com/erp-bundle-management-issues/) -- common pitfalls with bundle pricing, component tracking, and revenue distribution
- [Shadcn Combobox Patterns](https://shadcnstudio.com/docs/components/combobox) -- searchable dropdown variants with async, grouped, and creatable options
- [YourTradebase Quoting](https://www.yourtradebase.com/electricians/quote-software) -- tradesperson-specific quoting patterns, template reuse
- [Extraflow Quoting](https://extraflow.io/industries/estimating-and-quoting-software-for-plumbing) -- plumber quoting workflows with saved item catalogues

---
*Research completed: 2026-03-08*
