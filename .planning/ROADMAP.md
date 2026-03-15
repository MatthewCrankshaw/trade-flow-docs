# Roadmap: Trade Flow

## Milestones

- v1.0 Scheduling -- Phases 1-8 (shipped 2026-03-07)
- v1.1 Item Tax Rate Linkage -- Phases 9-10 (shipped 2026-03-08)
- v1.2 Bundles & Quotes -- Phases 11-14 (shipped 2026-03-15)
- v1.3 Send Quotes -- Phases 15-20 (in progress)

## Phases

<details>
<summary>v1.0 Scheduling (Phases 1-8) -- SHIPPED 2026-03-07</summary>

- [x] Phase 1: Visit Type Backend (2/2 plans) -- completed 2026-02-23
- [x] Phase 2: Visit Type Management UI (2/2 plans) -- completed 2026-02-28
- [x] Phase 3: Schedule Data Model and Create API (2/2 plans) -- completed 2026-03-01
- [x] Phase 4: Schedule Status and CRUD API (3/3 plans) -- completed 2026-03-01
- [x] Phase 5: Schedule Creation UI (2/2 plans) -- completed 2026-03-07
- [x] Phase 6: Schedule List and Detail UI (2/2 plans) -- completed 2026-03-07
- [x] Phase 7: Schedule Edit and Management UI (2/2 plans) -- completed 2026-03-07
- [x] Phase 8: Job Detail Integration (1/1 plan) -- completed 2026-03-07

Full details: `.planning/milestones/v1.0-ROADMAP.md`

</details>

<details>
<summary>v1.1 Item Tax Rate Linkage (Phases 9-10) -- SHIPPED 2026-03-08</summary>

- [x] Phase 9: Item Tax Rate API (4/4 plans) -- completed 2026-03-08
- [x] Phase 10: Item Tax Rate UI (2/2 plans) -- completed 2026-03-08

Full details: `.planning/milestones/v1.1-ROADMAP.md`

</details>

<details>
<summary>v1.2 Bundles & Quotes (Phases 11-14) -- SHIPPED 2026-03-15</summary>

- [x] Phase 11: Bundle Bug Fix and Foundation (1/1 plan) -- completed 2026-03-08
- [x] Phase 12: Bundle Component Editing (2/2 plans) -- completed 2026-03-08
- [x] Phase 13: Quote API Integration (3/3 plans) -- completed 2026-03-14
- [x] Phase 14: Quote Detail and Line Items (6/6 plans) -- completed 2026-03-14

Full details: `.planning/milestones/v1.2-ROADMAP.md`

</details>

### v1.3 Send Quotes (In Progress)

**Milestone Goal:** Enable tradespeople to send quotes to customers via email and receive automatic accept/reject responses back into Trade Flow.

- [x] **Phase 15: Quote Deletion** - Tradesperson can delete draft quotes with confirmation (completed 2026-03-15)
- [ ] **Phase 16: Token Infrastructure and Public API** - Secure token system for customer quote access without login
- [ ] **Phase 17: Customer Quote Page** - Customer can view a quote online via secure link
- [ ] **Phase 18: Quote Email Sending** - Tradesperson can send quotes to customers via email
- [ ] **Phase 19: Customer Response** - Customer can accept or reject a quote and tradesperson is notified
- [ ] **Phase 20: PDF Generation** - Professional PDF generated for quotes with download and email attachment

## Phase Details

### Phase 15: Quote Deletion
**Goal**: Tradesperson can remove unwanted draft quotes from their system
**Depends on**: Nothing (independent of all other v1.3 work)
**Requirements**: QMGT-01
**Success Criteria** (what must be TRUE):
  1. User can delete a quote that is in Draft status from the quote detail page
  2. User sees a confirmation dialog before deletion proceeds
  3. Deleted quote no longer appears in the quote list
  4. User cannot delete a quote that is in Sent, Accepted, or Rejected status
**Plans**: 2 plans

Plans:
- [ ] 15-01-PLAN.md — Add DELETED status, transition, deletedAt field, filter from list (API)
- [ ] 15-02-PLAN.md — Delete UI on detail page and list page with confirmation dialog

### Phase 16: Token Infrastructure and Public API
**Goal**: Secure foundation for customers to access quotes without logging in
**Depends on**: Nothing (infrastructure phase, no UI)
**Requirements**: RESP-01 (partial -- backend only)
**Success Criteria** (what must be TRUE):
  1. A cryptographically secure token can be generated for any quote
  2. Public API endpoint returns quote data when given a valid token (no authentication required)
  3. Expired or revoked tokens return a clear error, not quote data
  4. Public endpoint exposes only customer-safe fields (no internal IDs, no business-sensitive data)
**Plans**: TBD

Plans:
- [ ] 16-01: TBD

### Phase 17: Customer Quote Page
**Goal**: Customer can view a quote online and see that their visit was recorded
**Depends on**: Phase 16 (token infrastructure provides the public API)
**Requirements**: RESP-01 (frontend), RESP-05, DLVR-05
**Success Criteria** (what must be TRUE):
  1. Customer can open a quote link in their browser and see all quote details (line items, totals, business info) without logging in
  2. Customer can download the quote as a PDF from the online view (placeholder until Phase 20 delivers real PDF -- shows download button wired to endpoint)
  3. Tradesperson can see when a customer has viewed the quote (viewed indicator with timestamp on quote detail page)
  4. Page displays cleanly on mobile devices (tradespeople's customers often view on phones)
**Plans**: TBD

Plans:
- [ ] 17-01: TBD

### Phase 18: Quote Email Sending
**Goal**: Tradesperson can send a quote to their customer via email with one action
**Depends on**: Phase 16 (token generation for email link), Phase 17 (landing page the email links to)
**Requirements**: DLVR-01, DLVR-02, DLVR-03, DLVR-04, AUTO-01
**Success Criteria** (what must be TRUE):
  1. User can send a quote from the quote detail page and the customer receives an email with a link to view the quote online
  2. User can review and edit the pre-filled email message in a send dialog before sending
  3. User can configure a default quote email template in business settings with variable placeholders
  4. Quote status automatically transitions from Draft to Sent when the email is successfully sent
  5. User can re-send a quote that is already in Sent status (customer receives a new email)
**Plans**: TBD

Plans:
- [ ] 18-01: TBD

### Phase 19: Customer Response
**Goal**: Customer can accept or reject a quote and the tradesperson is immediately notified
**Depends on**: Phase 17 (customer quote page hosts the response buttons), Phase 18 (customers need to receive emails to respond)
**Requirements**: RESP-02, RESP-03, RESP-04, AUTO-02, AUTO-03, NOTF-01, NOTF-02
**Success Criteria** (what must be TRUE):
  1. Customer can accept a quote with one click from the online view and sees a confirmation
  2. Customer can decline a quote with one click and optionally provide a reason
  3. Quote status automatically updates to Accepted or Rejected based on customer response
  4. Tradesperson receives an email notification when a customer accepts or declines
  5. A quote that has already been responded to shows the current status instead of response buttons
**Plans**: TBD

Plans:
- [ ] 19-01: TBD

### Phase 20: PDF Generation
**Goal**: Professional quote PDFs are available for download and attached to emails
**Depends on**: Phase 18 (PDF attachment added to existing email flow)
**Requirements**: PDF-01
**Success Criteria** (what must be TRUE):
  1. System generates a professional PDF showing business details, customer details, quote number/date, all line items with bundle summaries, subtotal/tax/total, and validity date
  2. User can download the PDF from the quote detail page
  3. Customer can download the PDF from the online quote view
  4. Sent quote emails include the PDF as an attachment
**Plans**: TBD

Plans:
- [ ] 20-01: TBD

## Progress

**Execution Order:**
Phases execute in numeric order: 15 -> 16 -> 17 -> 18 -> 19 -> 20

| Phase | Milestone | Plans Complete | Status | Completed |
|-------|-----------|----------------|--------|-----------|
| 1. Visit Type Backend | v1.0 | 2/2 | Complete | 2026-02-23 |
| 2. Visit Type Management UI | v1.0 | 2/2 | Complete | 2026-02-28 |
| 3. Schedule Data Model and Create API | v1.0 | 2/2 | Complete | 2026-03-01 |
| 4. Schedule Status and CRUD API | v1.0 | 3/3 | Complete | 2026-03-01 |
| 5. Schedule Creation UI | v1.0 | 2/2 | Complete | 2026-03-07 |
| 6. Schedule List and Detail UI | v1.0 | 2/2 | Complete | 2026-03-07 |
| 7. Schedule Edit and Management UI | v1.0 | 2/2 | Complete | 2026-03-07 |
| 8. Job Detail Integration | v1.0 | 1/1 | Complete | 2026-03-07 |
| 9. Item Tax Rate API | v1.1 | 4/4 | Complete | 2026-03-08 |
| 10. Item Tax Rate UI | v1.1 | 2/2 | Complete | 2026-03-08 |
| 11. Bundle Bug Fix and Foundation | v1.2 | 1/1 | Complete | 2026-03-08 |
| 12. Bundle Component Editing | v1.2 | 2/2 | Complete | 2026-03-08 |
| 13. Quote API Integration | v1.2 | 3/3 | Complete | 2026-03-14 |
| 14. Quote Detail and Line Items | v1.2 | 6/6 | Complete | 2026-03-14 |
| 15. Quote Deletion | 2/2 | Complete    | 2026-03-15 | - |
| 16. Token Infrastructure and Public API | v1.3 | 0/0 | Not started | - |
| 17. Customer Quote Page | v1.3 | 0/0 | Not started | - |
| 18. Quote Email Sending | v1.3 | 0/0 | Not started | - |
| 19. Customer Response | v1.3 | 0/0 | Not started | - |
| 20. PDF Generation | v1.3 | 0/0 | Not started | - |
