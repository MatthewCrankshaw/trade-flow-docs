# Domain Pitfalls: Item Tax Rate Linkage

**Domain:** Adding tax rate ID references to items in a NestJS/MongoDB business management app
**Researched:** 2026-03-07

## Critical Pitfalls

Mistakes that cause rewrites or major issues.

### Pitfall 1: Replacing `defaultTaxRate` Without Updating Quote Line Item Factories

**What goes wrong:** The item entity currently stores `defaultTaxRate` as a number. Quote line item factories (`quote-standard-line-item-factory.service.ts` and `quote-bundle-line-item-factory.service.ts`) read `item.defaultTaxRate` directly to set the `taxRate` on new quote line items. If you replace `defaultTaxRate: number` with `defaultTaxRateId: string` but forget to update these factories, quote creation breaks silently -- line items get `undefined` or `null` tax rates instead of actual percentage values.

**Why it happens:** The item-to-quote pipeline is not obvious when you are focused on the item module. The coupling is through the `IItemDto` interface, which is consumed by the quote module's factories. There is no compile-time safety if you change a field name -- TypeScript catches the type change but only if the quote factories are re-checked.

**Consequences:** Quotes generate with zero or undefined tax rates. Since quotes involve money, this directly affects invoicing accuracy downstream. The bug may not surface until a user creates a quote and notices wrong totals.

**Prevention:**
- Before changing `IItemDto`, grep the entire codebase for every reference to `defaultTaxRate` (there are 15+ references across entity, DTO, repository mappers, request validators, response mappers, service validators, quote factories, default items creator, and merge utilities)
- The quote line item entity stores `taxRate: number` -- this is a snapshot value and should remain a number. The factory must resolve the tax rate ID to a percentage before setting it
- Write a test that creates a quote line item from an item with a tax rate reference, asserting the line item gets the correct numeric rate

**Detection:** Quote creation tests fail. If no quote tests exercise the full pipeline (item lookup through line item creation), the bug hides until manual testing.

**Phase relevance:** API phase -- must update quote factories in the same phase as the item schema change.

---

### Pitfall 2: Not Validating Tax Rate Exists on Item Create/Update

**What goes wrong:** Accepting a `defaultTaxRateId` without verifying it exists in the tax rates collection, or that it belongs to the same business. Items end up referencing tax rates that do not exist or belong to a different business.

**Why it happens:** MongoDB does not enforce foreign key constraints. The current item creator service validates `defaultTaxRate` only as a non-negative number -- there is no cross-collection lookup. When switching to an ID reference, developers may add string validation (`IsMongoId()`) but forget the existence check.

**Consequences:** Items reference nonexistent tax rates. When the UI tries to display the linked tax rate name, it gets a 404 or null. Quote creation using this item fails or produces wrong results.

**Prevention:**
- Follow the exact pattern from `ScheduleCreatorService.create()` -- it validates `visitTypeId` by calling `visitTypeRetriever.findByIdOrFail()` in a try/catch, wrapping the error with a domain-specific error code
- Add the same validation to `ItemCreatorService.validateCommonFields()` for non-bundle items
- Add the same validation to the update path (currently `ItemUpdaterService.update()` does no field-level validation -- this is a gap that needs fixing)
- Validate the tax rate belongs to the same business (the retriever's access control should handle this, but verify)

**Detection:** Integration tests that attempt to create items with invalid tax rate IDs. Unit tests that mock the tax rate retriever to throw.

**Phase relevance:** API phase -- implement alongside the field change, not after.

---

### Pitfall 3: Forgetting to Update the Default Items Creator

**What goes wrong:** The `DefaultBusinessItemsCreatorService` creates items on business onboarding with `defaultTaxRate: 0` hardcoded. After the schema change, this code still tries to set a numeric value instead of a tax rate ID, breaking onboarding for new users.

**Why it happens:** The default items creator is in the `business` module, not the `item` module. It is easy to miss when focused on item CRUD changes. The current code sets `defaultTaxRate: 0` for all default labour, fee, and material items, and `null` for bundles.

**Consequences:** New business creation fails or creates items with invalid tax rate references. Since this runs during onboarding, it blocks the entire user registration flow.

**Prevention:**
- The default items creator must reference the business's default tax rate (which is created by `DefaultTaxRatesCreatorService` in the same onboarding flow)
- Ensure the tax rates are created BEFORE default items (check the order in `BusinessCreatorService.create()` -- currently tax rates are created after items, which would cause a dependency ordering bug)
- Either reorder the calls so `defaultTaxRatesCreator.createDefaultTaxRates()` runs before `defaultItemsCreator.createDefaultItems()`, or make the items creator look up the default tax rate

**Detection:** Create a new business in development and verify items have valid tax rate references. Integration test for the full onboarding flow.

**Phase relevance:** API phase -- must fix ordering and update default items creator.

---

## Moderate Pitfalls

### Pitfall 4: Bundle Items and Tax Rate References

**What goes wrong:** Bundle items currently set `defaultTaxRate: null` because bundles derive tax from their components. When switching to `defaultTaxRateId`, the validation logic must still enforce that bundle items do NOT have a tax rate ID, and that each component item has one. The existing validation checks `item.defaultTaxRate !== null` for bundles -- this condition needs updating.

**Prevention:**
- Update `validateCommonFields()` to check `defaultTaxRateId !== null` instead of `defaultTaxRate !== null` for the bundle exclusion rule
- Ensure the bundle component validation also verifies each component item has a valid tax rate reference
- The `CreateItemRequest` uses `@ValidateIf((o) => o.type !== ItemType.BUNDLE)` for `defaultTaxRate` -- update this to validate the new ID field with the same conditional

**Phase relevance:** API phase -- update all bundle-related validation logic.

---

### Pitfall 5: UI Form Loads Tax Rates Before They Exist

**What goes wrong:** The item create/edit form needs a dropdown of tax rates. If the `useGetTaxRatesQuery` call fails or returns empty (no tax rates exist yet), the user sees an empty dropdown with no way to create an item. This is especially problematic during onboarding if the ordering fix from Pitfall 3 is not applied.

**Prevention:**
- Handle the empty tax rates case in the UI -- show a message like "Create a tax rate first" with a link to the tax rates section
- Consider pre-selecting the business's default tax rate (the one with `isDefault: true`) when creating a new item
- Handle the loading state for the tax rates query -- do not show the form until tax rates are loaded
- Consider what happens if the only tax rate a business has is `disabled` -- the dropdown should only show enabled tax rates

**Phase relevance:** UI phase -- design the form with these edge cases.

---

### Pitfall 6: Losing the Numeric Tax Rate During API Response Transition

**What goes wrong:** The API currently returns `defaultTaxRate: 20` (a number) in the item response. After the change, it returns `defaultTaxRateId: "abc123"`. The UI needs the actual rate value to display "20% VAT" on items, but now only has an ID. Developers either (a) force the UI to make a separate API call for every item to resolve the tax rate name, causing N+1 queries, or (b) forget to include tax rate details in the item response at all.

**Prevention:**
- Include resolved tax rate details in the item response: return both `defaultTaxRateId` and a nested `defaultTaxRate: { id, name, rate, rateType }` object in the API response
- Alternatively, populate the tax rate in the repository layer when fetching items (MongoDB aggregation with `$lookup`, or manual population in the service layer)
- The UI item type should include the resolved tax rate, not just the ID
- Do NOT make the UI call `getTaxRate` for each item individually -- this creates N+1 API calls on the items list page

**Phase relevance:** API phase (response shape) and UI phase (consuming the response).

---

### Pitfall 7: `merge-existing-item-with-changes` Utility Handles the Field Wrong

**What goes wrong:** The `mergeExistingItemWithChanges` utility merges update request fields with existing item data. It currently has a `mergeDefaultTaxRate` function that handles the numeric merge. When switching to a tax rate ID, this merge logic needs updating to handle the string ID correctly -- especially the distinction between `undefined` (field not in request, keep existing) and `null` (explicitly clearing, which should not be valid for non-bundle items).

**Prevention:**
- Update the merge utility to handle `defaultTaxRateId` with the same undefined-vs-null pattern used by `visitTypeId` in `merge-existing-schedule-with-changes.utility.ts`
- For non-bundle items, `null` should NOT be a valid value for the tax rate reference (unlike `visitTypeId` on schedules, which is optional). Validate this in the service layer
- Update the corresponding unit tests

**Phase relevance:** API phase -- update mapper utilities.

---

### Pitfall 8: Valibot Schema Still Validates as Numeric String

**What goes wrong:** The UI's `itemFormSchema` validates `defaultTaxRate` as a numeric string with `Number.parseFloat()`. After switching to a tax rate ID dropdown, this validation is wrong -- the form value is now a selected ID string, not a number to parse. If the schema is not updated, form submission fails validation.

**Prevention:**
- Replace the `defaultTaxRate` field in `itemFormSchema` with a `defaultTaxRateId` field validated as a non-empty string (required for non-bundle items)
- The bundle item form schema does not include tax rate fields and should remain unchanged
- Update the `ItemFormValues` type and all components consuming it (`MaterialItemForm`, `LabourItemForm`, `FeeItemForm`)

**Phase relevance:** UI phase -- update schema before updating form components.

---

## Minor Pitfalls

### Pitfall 9: RTK Query Cache Invalidation Gap

**What goes wrong:** When a tax rate is updated (name or rate percentage changes), item list displays become stale -- they still show the old tax rate details. RTK Query's `providesTags` on items does not know about tax rate changes.

**Prevention:**
- When the tax rate mutation invalidates `TaxRate` tags, also invalidate `Item` tags (or at least `{ type: "Item", id: "LIST" }`) since items display resolved tax rate data
- Alternatively, ensure the item list fetches fresh data when navigating to it (RTK Query's `refetchOnMountOrArgChange` can help)

**Phase relevance:** UI phase -- configure cache invalidation when wiring up the tax rate dropdown.

---

### Pitfall 10: Disabled Tax Rates on Existing Items

**What goes wrong:** A user assigns tax rate "Standard VAT 20%" to items, then later disables that tax rate. The items still reference it. When displaying the item, the UI either shows nothing (if it filters to enabled-only) or shows a disabled tax rate. When editing the item, the dropdown (showing only enabled rates) does not include the currently selected rate, making it look like no rate is set.

**Prevention:**
- The API should NOT prevent disabling a tax rate that is referenced by items (this is intentional -- it just means new items should not use it)
- The item edit form dropdown should include the currently-selected tax rate even if disabled, with a visual indicator like "(disabled)" suffix
- The item display should show the tax rate normally even if disabled -- the reference is still valid
- Consider: should creating a NEW item with a disabled tax rate be allowed? Probably not -- validate that the referenced tax rate has `status: enabled` on create, but allow it on existing items (update path should not re-validate status of an unchanged tax rate ID)

**Phase relevance:** Both API and UI phases -- validation differs between create and update paths.

---

## Phase-Specific Warnings

| Phase Topic | Likely Pitfall | Mitigation |
|-------------|---------------|------------|
| API schema change | Quote factories break (Pitfall 1) | Grep all `defaultTaxRate` references; update quote factories to resolve ID to rate |
| API validation | Missing existence check (Pitfall 2) | Follow `visitTypeId` validation pattern from schedule module |
| API onboarding | Default items creator breaks (Pitfall 3) | Fix creation ordering; default items reference default tax rate |
| API bundle logic | Bundle validation conditions wrong (Pitfall 4) | Update all `defaultTaxRate` checks to use new field name |
| API response shape | N+1 queries for tax rate details (Pitfall 6) | Include resolved tax rate in item response |
| API merge utility | Merge handles field type wrong (Pitfall 7) | Update merge to handle string ID with undefined/null semantics |
| UI form schema | Valibot validates number not ID (Pitfall 8) | Replace numeric validation with string ID validation |
| UI empty state | No tax rates available (Pitfall 5) | Handle empty/loading states; pre-select default |
| UI cache | Stale items after tax rate edit (Pitfall 9) | Cross-invalidate Item tags on TaxRate mutations |
| UI disabled rates | Edit dropdown missing current rate (Pitfall 10) | Include disabled current rate in dropdown |

## Summary: Top 3 Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Quote factories break silently | HIGH | Quotes generate with wrong tax amounts | Update factories in same phase as schema change; add integration test |
| Onboarding flow breaks | HIGH | New users cannot create a business | Fix tax rate/item creation ordering; test full onboarding |
| N+1 API calls for tax rate display | MEDIUM | Item list page is slow; poor UX | Resolve tax rate details server-side in item response |

---
*Research completed: 2026-03-07*
