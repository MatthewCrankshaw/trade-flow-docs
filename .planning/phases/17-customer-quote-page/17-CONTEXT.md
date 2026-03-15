# Phase 17: Customer Quote Page - Context

**Gathered:** 2026-03-15
**Status:** Ready for planning

<domain>
## Phase Boundary

Customer-facing page where customers view a quote via secure token link without logging in. Includes: standalone quote view page (frontend route at `/quote/view/{token}`), view tracking (firstViewedAt on token record), viewed indicator on tradesperson's quote detail page, and PDF download button placeholder. API endpoint `GET /v1/public/quote/:token` already exists from Phase 16.

</domain>

<decisions>
## Implementation Decisions

### Page layout and structure
- Single card layout: business header, quote details, line items table/cards, totals footer, validity date, PDF button
- Standalone page with no Trade Flow app chrome (no sidebar, no header, no login UI) -- this is a customer page, not the tradesperson's app
- Clean background with the card centered -- customer should only see the quote content
- Route lives outside the `AuthenticatedLayout` wrapper in App.tsx

### Line items display
- Bundles show as rolled-up single lines only -- customer does not see individual bundle components
- Desktop: table layout for line items (item name, qty, unit price, total)
- Mobile: stacked cards for line items (reuse QuoteLineItemsCardList pattern) -- customers are likely viewing on phones
- Item type labels (material, labour, fee) included for transparency

### Error and expiry states
- Friendly and helpful tone for all error messages -- these are tradespeople's customers
- Show business name in error messages when available (e.g., "This quote from Smith Plumbing has expired. Contact them for an updated link.") -- requires API tweak to return business name with expired/revoked token error responses
- Invalid/unknown token: 404 page with generic "Quote not found" message
- Loading state: skeleton matching the card layout sections (header, line items area, totals)

### Already-responded quotes
- Accepted/rejected quotes still show full quote content with a status banner at top
- Banner text: "You accepted this quote on [date]" or "You declined this quote on [date]"
- Quote data remains viewable as a reference -- customer doesn't need to dig through email

### View tracking
- Record firstViewedAt timestamp on the quote_tokens record on first visit only (set if null)
- No view count -- simple binary "has been viewed" with timestamp
- Update happens in the public endpoint handler when serving the quote
- Tradesperson sees a badge with timestamp on quote detail page: eye icon + "Viewed" + "15 Mar at 2:34pm"
- View indicator on detail page only -- not on the quotes list page (keep list clean)

### PDF download placeholder
- "Download PDF" button visible but disabled with tooltip: "PDF download coming soon"
- Positioned at bottom of card, after totals section
- Phase 20 will enable this button and wire it to the real PDF endpoint

### Claude's Discretion
- Exact card styling and spacing (use existing Card component patterns)
- How to fetch business name for error states (token record has quoteId -> quote has businessId, or add businessName to token record)
- Loading skeleton exact shimmer layout
- Error page illustration/icon choice
- Mobile breakpoint handling details

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Public API (Phase 16 output -- already built)
- `trade-flow-api/src/quote-token/controllers/public-quote.controller.ts` -- Public endpoint at GET /v1/public/quote/:token, response mapping, error handling
- `trade-flow-api/src/quote-token/responses/public-quote.response.ts` -- IPublicQuoteResponse and IPublicQuoteLineItemResponse interfaces (defines what data is available)
- `trade-flow-api/src/quote-token/services/quote-token-retriever.service.ts` -- Token lookup, expiry/revocation checks
- `trade-flow-api/src/quote-token/entities/quote-token.entity.ts` -- Token entity (firstViewedAt field needs adding here)

### Frontend routing and app shell
- `trade-flow-ui/src/App.tsx` -- Route definitions, AuthenticatedLayout wrapper (new public route goes OUTSIDE this wrapper)

### Existing quote UI patterns (for consistency)
- `trade-flow-ui/src/features/quotes/components/QuoteLineItemsCardList.tsx` -- Mobile card layout for line items (pattern to follow)
- `trade-flow-ui/src/features/quotes/components/QuoteLineItemsTable.tsx` -- Desktop table layout for line items
- `trade-flow-ui/src/features/quotes/components/QuoteActionStrip.tsx` -- Quote detail page actions (where viewed badge goes)

### UI component library
- `trade-flow-ui/src/components/ui/card.tsx` -- Card component for page layout
- `trade-flow-ui/src/components/ui/skeleton.tsx` -- Skeleton component for loading state
- `trade-flow-ui/src/components/ui/badge.tsx` -- Badge component for viewed indicator
- `trade-flow-ui/src/components/ui/table.tsx` -- Table component for line items

### Architecture and conventions
- `.planning/codebase/CONVENTIONS.md` -- Naming patterns, module structure
- `trade-flow-ui/CLAUDE.md` -- Frontend tech stack, coding conventions, design system references
- `trade-flow-ui/.github/copilot-instructions/design-system/` -- Visual design system (colours, typography, spacing)

### Requirements
- `.planning/REQUIREMENTS.md` -- RESP-01 (frontend portion), RESP-05 (PDF download placeholder), DLVR-05 (view tracking)

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `Card` component (`ui/card.tsx`): Base card with header/content/footer sections -- use for main quote card
- `Skeleton` component (`ui/skeleton.tsx`): Shimmer loading placeholder -- use for loading state
- `Badge` component (`ui/badge.tsx`): Small label/tag -- use for "Viewed" indicator on tradesperson's detail page
- `Table` component (`ui/table.tsx`): Table with header/body/row -- use for desktop line items
- `QuoteLineItemsCardList` pattern: Mobile card layout for line items -- follow this pattern for customer mobile view
- `Toaster` / `sonner.tsx`: Toast notifications -- available if needed for error states
- `useCurrency` hook: Currency formatting -- needed for displaying prices

### Established Patterns
- Feature-based modules: `src/features/quotes/` for quote-related components
- RTK Query for API calls: `quoteApi.ts` pattern with `injectEndpoints`
- Mobile-first responsive: 768px breakpoint, card layout below / table above
- Path alias `@/` for imports from `src/`

### Integration Points
- `App.tsx` Routes: New `/quote/view/:token` route OUTSIDE `AuthenticatedLayout`
- New page component: `src/pages/PublicQuotePage.tsx` (or similar)
- New API slice: Public quote fetch (separate from authenticated quoteApi -- no auth headers)
- `QuoteActionStrip.tsx` or quote detail header: Add "Viewed" badge when firstViewedAt exists
- `trade-flow-api` public controller: Add firstViewedAt update logic on token lookup

</code_context>

<specifics>
## Specific Ideas

No specific requirements -- open to standard approaches. Follow existing patterns from the codebase. The key principle is that this is a CUSTOMER page, not the tradesperson's app -- it should feel clean, professional, and standalone.

</specifics>

<deferred>
## Deferred Ideas

- View count tracking (multiple visits) -- could indicate customer interest, future enhancement
- Activity timeline on quote detail page -- structured history of quote events (created, sent, viewed, accepted/rejected)
- "Powered by Trade Flow" footer branding on customer page -- future consideration

</deferred>

---

*Phase: 17-customer-quote-page*
*Context gathered: 2026-03-15*
