# Requirements: Trade Flow

**Defined:** 2026-03-15
**Core Value:** A job is the centre of the business -- Trade Flow helps tradespeople run their entire business from first call to final payment in one simple, structured system.

## v1.3 Requirements

Requirements for Send Quotes milestone. Each maps to roadmap phases.

### Quote Delivery

- [x] **DLVR-01a**: User can send a quote to a customer via email with a link to view the quote online
- ~~**DLVR-01b**~~: *(deferred — PDF generation removed from v1.3)*
- [x] **DLVR-02**: User can configure a default quote email template in business settings with variable placeholders (customer name, job title, business name, user name)
- [x] **DLVR-03**: User can review and edit the pre-filled email message in a send dialog before sending, with template variables automatically resolved
- [x] **DLVR-04**: User can re-send a quote that is already in Sent status
- [x] **DLVR-05**: User can see when a customer has viewed the quote online (viewed indicator with timestamp)

### Customer Response

- [x] **RESP-01**: Customer can view the full quote online via a secure link without logging in
- [x] **RESP-02**: Customer can accept a quote with one click from the online view
- [x] **RESP-03**: Customer can decline a quote with one click from the online view
- [x] **RESP-04**: Customer can provide an optional reason when declining a quote
- ~~**RESP-05**~~: *(deferred — PDF generation removed from v1.3)*

### Notifications

- [x] **NOTF-01**: Tradesperson receives an email notification when a customer accepts a quote
- [x] **NOTF-02**: Tradesperson receives an email notification when a customer declines a quote

### ~~PDF Generation~~ *(deferred)*

- ~~**PDF-01**~~: *(deferred — PDF generation removed from v1.3)*

### Quote Management

- [x] **QMGT-01**: User can delete a quote (soft delete, only from Draft status)

### Status Automation

- [x] **AUTO-01**: Quote status automatically transitions from Draft to Sent when the user sends the quote
- [x] **AUTO-02**: Quote status automatically transitions from Sent to Accepted when the customer accepts
- [x] **AUTO-03**: Quote status automatically transitions from Sent to Rejected when the customer declines

## Future Requirements

Deferred to future release. Tracked but not in current roadmap.

### Quote Workflow

- **QWRK-01**: Days since sent display on quote list for follow-up prioritization
- **QWRK-02**: Quote duplication (create new quote from existing)

### Advanced Delivery

- **ADVD-01**: Automated follow-up reminder emails for unresponded quotes
- **ADVD-02**: SMS quote delivery

### Customer Portal

- **CUST-01**: Persistent customer login/portal for all quotes and invoices
- **CUST-02**: Electronic signature capture on quote acceptance

### PDF Generation

- **PDF-01**: System generates a professional PDF for a quote showing business details, customer details, quote number/date, line items, bundle summaries, subtotal/tax/total, and validity date
- **DLVR-01b**: Sent quote emails include the PDF as an attachment
- **RESP-05**: Customer can download the quote as a PDF from the online view

### Quote Options

- **QOPT-01**: Quote versioning (multiple versions per job)
- **QOPT-02**: Quote-level discounts

## Out of Scope

Explicitly excluded. Documented to prevent scope creep.

| Feature | Reason |
|---------|--------|
| Customer account/login system | Token-based access is frictionless; customers interact once per quote |
| Quote versioning/options | Significant complexity; sole tradespeople typically send one quote |
| Electronic signature | Legal complexity varies by jurisdiction; timestamp + IP sufficient for trade quotes |
| Automated follow-up reminders | Adds scheduling infrastructure; premature for solo tradesperson tool |
| Real-time notifications (WebSocket) | Significant new infrastructure; email notification covers 95% of need |
| Rich HTML quote template builder | Template builders are entire products; one clean PDF template is sufficient |
| Quote-level discounts | Deferred from v1.2; not part of send flow |
| Deposit collection on acceptance | Requires invoicing system (not yet built) |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| QMGT-01 | Phase 15 | Complete |
| RESP-01 | Phase 16 + 17 | Complete |
| DLVR-05 | Phase 17 | Complete |
| RESP-05 | Phase 17 | Complete |
| DLVR-01a | Phase 18 | Complete |
| DLVR-01b | Deferred | — |
| DLVR-02 | Phase 18 | Complete |
| DLVR-03 | Phase 18 | Complete |
| DLVR-04 | Phase 18 | Complete |
| AUTO-01 | Phase 18 | Complete |
| RESP-02 | Phase 19 | Complete |
| RESP-03 | Phase 19 | Complete |
| RESP-04 | Phase 19 | Complete |
| AUTO-02 | Phase 19 | Complete |
| AUTO-03 | Phase 19 | Complete |
| NOTF-01 | Phase 19 | Complete |
| NOTF-02 | Phase 19 | Complete |
| PDF-01 | Deferred | — |
| RESP-05 | Deferred | — |

**Coverage:**
- v1.3 requirements: 18 total
- Mapped to phases: 18
- Unmapped: 0

---
*Requirements defined: 2026-03-15*
*Last updated: 2026-03-15 after roadmap creation*
