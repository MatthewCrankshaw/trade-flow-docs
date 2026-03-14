# Requirements: Trade Flow

**Defined:** 2026-03-08
**Core Value:** A job is the centre of the business -- Trade Flow helps tradespeople run their entire business from first call to final payment in one simple, structured system.

## v1.2 Requirements

Requirements for Bundles & Quotes milestone. Each maps to roadmap phases.

### Bundle Fixes

- [x] **BNDL-01**: User can create a bundle item without errors (unit defaults to "bundle")
- [x] **BNDL-02**: User can edit bundle components on an existing bundle (add/remove items, change quantities)
- [x] **BNDL-03**: User can search and filter items when selecting bundle components via a searchable dropdown
- [x] **BNDL-04**: User can view bundle components in a structured list showing item name, quantity, and unit

### Quote Management

- [x] **QUOT-01**: User can create a new quote linked to a job and customer
- [ ] **QUOT-02**: User can view a list of all quotes with real API data (replacing mock data)
- [ ] **QUOT-03**: User can view quote detail with line items and calculated totals
- [x] **QUOT-04**: User can transition quote status (Draft → Sent → Accepted/Rejected)

### Quote Line Items

- [ ] **QLIT-01**: User can add standard items (material, labour, fee) to a quote
- [ ] **QLIT-02**: User can add bundle items to a quote (creates parent + component line items)
- [ ] **QLIT-03**: User can view bundle line items as a rolled-up line that expands to show individual components
- [ ] **QLIT-04**: User can view quote totals (subtotal, tax, total) calculated from line items

## Future Requirements

Deferred to future release. Tracked but not in current roadmap.

### Quote Extensions

- **QEXT-01**: User can delete a quote
- **QEXT-02**: User can duplicate a quote
- **QEXT-03**: User can update/remove individual line items on a quote
- **QEXT-04**: User can apply discounts at quote level
- **QEXT-05**: User can generate a PDF from a quote
- **QEXT-06**: Quote number auto-generated with sequential numbering

### Bundle Extensions

- **BEXT-01**: User can toggle components as optional when adding bundle to quote
- **BEXT-02**: User can choose between fixed and component-based pricing strategy in UI

## Out of Scope

Explicitly excluded. Documented to prevent scope creep.

| Feature | Reason |
|---------|--------|
| Invoice generation | Separate milestone -- quotes must ship first |
| Payment tracking | Depends on invoicing |
| Quote PDF export | Future requirement -- focus on core CRUD first |
| Editing individual bundle component line items on quotes | Industry best practice: modify the parent bundle, not individual components |
| Nested bundles (bundles containing bundles) | Imposed limitation to reduce complexity |
| Quote approval workflow (customer-facing) | Tradespeople manage quotes directly, no customer portal |
| Batch line item operations | Over-engineering for solo operator scale |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| BNDL-01 | Phase 11 | Complete |
| BNDL-02 | Phase 12 | Complete |
| BNDL-03 | Phase 11 | Complete |
| BNDL-04 | Phase 12 | Complete |
| QUOT-01 | Phase 13 | Complete |
| QUOT-02 | Phase 13 | Pending |
| QUOT-03 | Phase 14 | Pending |
| QUOT-04 | Phase 13 | Complete |
| QLIT-01 | Phase 14 | Pending |
| QLIT-02 | Phase 14 | Pending |
| QLIT-03 | Phase 14 | Pending |
| QLIT-04 | Phase 14 | Pending |

**Coverage:**
- v1.2 requirements: 12 total
- Mapped to phases: 12
- Unmapped: 0

---
*Requirements defined: 2026-03-08*
*Last updated: 2026-03-08 after roadmap creation*
