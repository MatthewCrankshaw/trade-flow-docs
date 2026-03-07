# Requirements: Trade Flow -- Item Tax Rate Linkage

**Defined:** 2026-03-07
**Core Value:** A job is the centre of the business -- Trade Flow helps tradespeople run their entire business from first call to final payment

## v1 Requirements

Requirements for the item tax rate linkage milestone. Each maps to roadmap phases.

### Item Tax Rate Reference

- [ ] **ITAX-01**: Item entity stores a tax rate ID reference instead of a numeric tax rate value
- [ ] **ITAX-02**: Creating an item validates that the referenced tax rate exists and belongs to the business
- [ ] **ITAX-03**: Updating an item validates the new tax rate ID if changed
- [ ] **ITAX-04**: Item API response includes the tax rate ID (UI resolves details client-side)
- [ ] **ITAX-05**: Default items created during onboarding reference the correct default tax rate

### Quote Integration

- [ ] **QUOT-01**: Quote line item factory resolves tax rate percentage from the referenced tax rate ID when creating quote lines

### Item UI

- [ ] **ITMUI-01**: Item create form shows a tax rate dropdown populated from the business's tax rates
- [ ] **ITMUI-02**: Item edit form shows a tax rate dropdown with the current tax rate pre-selected
- [ ] **ITMUI-03**: Item list/detail displays the linked tax rate name and percentage

## v2 Requirements

Deferred to future release. Tracked but not in current roadmap.

### Tax Rate Management

- **TAXM-01**: Archiving a tax rate warns if active items reference it
- **TAXM-02**: Archived tax rate name/percentage still displays on items that reference it

## Out of Scope

| Feature | Reason |
|---------|--------|
| Multiple tax rates per item | Sole traders deal with simple VAT/no-VAT; over-engineering |
| Tax rate inline creation from item form | Adds complexity; user can create tax rates in settings first |
| Data migration for existing items | App not in production; no existing user data to migrate |
| Server-side tax rate embedding in item response | UI resolves from RTK Query cache; no need for server joins |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| ITAX-01 | Phase 9 | Pending |
| ITAX-02 | Phase 9 | Pending |
| ITAX-03 | Phase 9 | Pending |
| ITAX-04 | Phase 9 | Pending |
| ITAX-05 | Phase 9 | Pending |
| QUOT-01 | Phase 9 | Pending |
| ITMUI-01 | Phase 10 | Pending |
| ITMUI-02 | Phase 10 | Pending |
| ITMUI-03 | Phase 10 | Pending |

**Coverage:**
- v1 requirements: 9 total
- Mapped to phases: 9
- Unmapped: 0

---
*Requirements defined: 2026-03-07*
*Last updated: 2026-03-07 after roadmap creation*
