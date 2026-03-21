# Phase 17: Customer Quote Page - Research

**Researched:** 2026-03-16
**Domain:** React frontend (public page), NestJS backend (view tracking + error enrichment)
**Confidence:** HIGH

## Summary

Phase 17 delivers a customer-facing quote view page at `/quote/view/:token` that operates entirely outside the authenticated app shell. The public API endpoint (`GET /v1/public/quote/:token`) already exists from Phase 16 and returns `IPublicQuoteResponse` with business name, customer name, line items, and totals. The frontend work is a new standalone page using existing shadcn/ui components (Card, Table, Badge, Skeleton, Button, Separator, Alert). The backend work is small: add `firstViewedAt` field to the quote_tokens collection, update it on first fetch, enrich error responses with business name for expired/revoked tokens, and expose `firstViewedAt` through the authenticated quote response for the tradesperson's "Viewed" badge.

A critical design consideration is that the public page cannot use `useBusinessCurrency()` (which depends on authenticated user context and business lookup). Instead, the public page needs a standalone currency formatter. The API already returns totals as major unit numbers (decimals), so the frontend can format these with `Intl.NumberFormat` directly or use the existing `formatCurrencyByCode` / `useCurrencyFormatter` hook which doesn't require auth.

**Primary recommendation:** Build the public quote page as a self-contained feature module (`src/features/public-quote/`) with its own RTK Query API slice that uses a separate `fetchBaseQuery` with no auth headers. Use `useCurrencyFormatter("GBP")` for currency formatting since the public API does not return currency info (hardcoded to GBP for now, matching the only supported currency).

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- Single card layout: business header, quote details, line items table/cards, totals footer, validity date, PDF button
- Standalone page with no Trade Flow app chrome (no sidebar, no header, no login UI) -- this is a customer page, not the tradesperson's app
- Clean background with the card centered -- customer should only see the quote content
- Route lives outside the `AuthenticatedLayout` wrapper in App.tsx
- Bundles show as rolled-up single lines only -- customer does not see individual bundle components
- Desktop: table layout for line items (item name, qty, unit price, total)
- Mobile: stacked cards for line items (reuse QuoteLineItemsCardList pattern)
- Item type labels (material, labour, fee) included for transparency
- Friendly and helpful tone for all error messages
- Show business name in error messages when available (requires API tweak to return business name with expired/revoked token error responses)
- Invalid/unknown token: 404 page with generic "Quote not found" message
- Loading state: skeleton matching the card layout sections
- Accepted/rejected quotes still show full quote content with a status banner at top
- Record firstViewedAt timestamp on the quote_tokens record on first visit only (set if null)
- No view count -- simple binary "has been viewed" with timestamp
- Update happens in the public endpoint handler when serving the quote
- Tradesperson sees a badge with timestamp on quote detail page: eye icon + "Viewed" + timestamp
- View indicator on detail page only -- not on the quotes list page
- "Download PDF" button visible but disabled with tooltip: "PDF download coming soon"

### Claude's Discretion
- Exact card styling and spacing (use existing Card component patterns)
- How to fetch business name for error states (token record has quoteId -> quote has businessId, or add businessName to token record)
- Loading skeleton exact shimmer layout
- Error page illustration/icon choice
- Mobile breakpoint handling details

### Deferred Ideas (OUT OF SCOPE)
- View count tracking (multiple visits)
- Activity timeline on quote detail page
- "Powered by Trade Flow" footer branding on customer page
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| RESP-01 | Customer can view the full quote online via a secure link without logging in | Public page at `/quote/view/:token` using existing `GET /v1/public/quote/:token` API; standalone route outside AuthenticatedLayout; new public RTK Query slice without auth headers |
| RESP-05 | Customer can download the quote as a PDF from the online view | Disabled "Download PDF" button placeholder with tooltip; Phase 20 enables real functionality |
| DLVR-05 | User can see when a customer has viewed the quote online (viewed indicator with timestamp) | Add `firstViewedAt` to quote_tokens entity/DTO/repository; update on first public fetch; expose via authenticated quote response; "Viewed" badge in QuoteActionStrip |
</phase_requirements>

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| React | 19 | UI framework | Already in project |
| React Router DOM | 7.x | Routing (`/quote/view/:token`) | Already in project |
| RTK Query | (via Redux Toolkit) | API data fetching for public endpoint | Already used for all API calls |
| Tailwind CSS | 4.x | Styling | Already in project |
| shadcn/ui | New York preset | UI components (Card, Table, Badge, etc.) | Already in project |
| lucide-react | latest | Icons (Eye, Download, AlertCircle, FileX) | Already in project |
| date-fns | latest | Date formatting for timestamps | Already used in QuoteDetailPage |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| dinero.js | latest | Currency formatting (via `useCurrencyFormatter`) | Public page needs currency formatting without auth context |
| Luxon | (API side) | DateTime handling for firstViewedAt | Already used for all date fields in API |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Separate public API slice | Reusing existing `apiSlice` | Existing apiSlice always attaches Firebase auth token; public endpoint must NOT send auth headers (would fail or expose user info). Separate base query is cleaner. |

**Installation:**
No new packages needed. All dependencies already installed.

## Architecture Patterns

### Recommended Project Structure
```
trade-flow-ui/src/
├── features/
│   └── public-quote/
│       ├── api/
│       │   └── publicQuoteApi.ts        # Separate RTK Query slice, no auth
│       ├── components/
│       │   ├── PublicQuoteCard.tsx       # Main quote card layout
│       │   ├── PublicQuoteLineItems.tsx  # Responsive line items (table/cards)
│       │   ├── PublicQuoteTotals.tsx     # Totals section
│       │   ├── PublicQuoteSkeleton.tsx   # Loading skeleton
│       │   └── PublicQuoteError.tsx      # Error state cards
│       ├── types/
│       │   └── public-quote.types.ts    # IPublicQuoteResponse mirror
│       └── index.ts                     # Barrel exports
├── pages/
│   └── PublicQuotePage.tsx              # Route page component

trade-flow-api/src/
└── quote-token/
    ├── entities/
    │   └── quote-token.entity.ts        # Add firstViewedAt?: Date
    ├── data-transfer-objects/
    │   └── quote-token.dto.ts           # Add firstViewedAt?: DateTime
    ├── repositories/
    │   └── quote-token.repository.ts    # Add updateFirstViewedAt method
    └── controllers/
        └── public-quote.controller.ts   # Add firstViewedAt update + error enrichment
```

### Pattern 1: Separate Public API Slice (No Auth)
**What:** Create a dedicated RTK Query API slice with `fetchBaseQuery` that does NOT attach Firebase auth tokens.
**When to use:** Any public (unauthenticated) API endpoint.
**Example:**
```typescript
// Source: Existing apiSlice pattern adapted for public use
import { createApi, fetchBaseQuery } from "@reduxjs/toolkit/query/react";

const publicBaseQuery = fetchBaseQuery({
  baseUrl: import.meta.env.VITE_API_BASE_URL || "http://localhost:3000",
  // NO prepareHeaders — no auth token
});

export const publicApiSlice = createApi({
  reducerPath: "publicApi",
  baseQuery: publicBaseQuery,
  tagTypes: [],
  endpoints: () => ({}),
});
```

### Pattern 2: Route Outside AuthenticatedLayout
**What:** Add public route before the `AuthenticatedLayout` wrapper in App.tsx.
**When to use:** Pages that don't require authentication (customer-facing pages).
**Example:**
```typescript
// In App.tsx
<Routes>
  <Route path="/login" element={<LoginPage />} />
  <Route path="/quote/view/:token" element={<PublicQuotePage />} />

  {/* Authenticated routes with onboarding context */}
  <Route element={<AuthenticatedLayout />}>
    {/* ... existing authenticated routes ... */}
  </Route>
</Routes>
```

### Pattern 3: Currency Formatting Without Auth Context
**What:** Use `useCurrencyFormatter("GBP")` instead of `useBusinessCurrency()` for public pages.
**When to use:** When the page has no authenticated user/business context.
**Why:** `useBusinessCurrency()` calls `useGetCurrentUserQuery()` and `useGetBusinessQuery()` which require auth. The public API returns totals as major unit numbers (decimals like 150.50). Need `formatCurrencyByCode` or `useCurrencyFormatter` which accept a currency code directly.
**Critical note:** The public API response returns amounts in **major units** (decimals), while the authenticated API returns them in **minor units** (cents). The public page must use `createMoneyFromDecimal` (not `createMoney`) or format directly with `Intl.NumberFormat`.

### Pattern 4: firstViewedAt Update in Public Controller
**What:** Set `firstViewedAt` on the quote_tokens record during the public quote fetch, only if null.
**When to use:** On successful quote retrieval (not on error/expired/revoked paths).
**Example:**
```typescript
// In PublicQuoteController.findByToken, after successful validation:
if (!tokenDto.firstViewedAt) {
  await this.quoteTokenRepository.updateFirstViewedAt(tokenDto.id);
}
```

### Pattern 5: Business Name in Error Responses
**What:** For expired/revoked tokens, look up the business name via quoteId -> quote -> businessId -> business to include in the error response.
**When to use:** Token is valid but expired or revoked.
**Recommended approach:** After finding the token but before throwing the expired/revoked error, look up the quote and business to get the business name. Return it in the error details field.
```typescript
// Enrich error with business name
if (this.quoteTokenRetriever.isExpired(tokenDto) || this.quoteTokenRetriever.isRevoked(tokenDto)) {
  let businessName = "";
  try {
    const quote = await this.quoteRepository.findByIdOrFail(tokenDto.quoteId);
    const business = await this.businessRepository.findByIdOrFail(quote.businessId);
    businessName = business.name;
  } catch { /* fall through with empty name */ }

  const code = this.quoteTokenRetriever.isRevoked(tokenDto) ? "TOKEN_REVOKED" : "TOKEN_EXPIRED";
  throw new HttpException(
    createErrorResponse([{ code, message: "...", details: businessName }]),
    HttpStatus.GONE,
  );
}
```

### Anti-Patterns to Avoid
- **Using authenticated API slice for public endpoints:** The existing `apiSlice` always attaches Firebase auth tokens. Using it for public endpoints would either fail (if user not logged in) or unnecessarily expose auth info.
- **Using `useBusinessCurrency()` on public page:** This hook depends on authenticated user context. Will fail or return incorrect defaults on public pages.
- **Treating public API amounts as minor units:** The public API `mapToPublicResponse` calls `.toMajorUnits()` on all money values. The amounts are decimals, not cents. Using `createMoney()` (which expects minor units) would display 100x the actual amount.
- **Updating firstViewedAt on every request:** Must check if null first. Only set once on first view.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Currency formatting | Manual `toFixed(2)` + currency symbol | `useCurrencyFormatter` hook + dinero.js | Locale-aware formatting, rounding edge cases |
| Responsive table/cards | Custom media query logic | Tailwind `md:` responsive classes + `hidden md:block` / `block md:hidden` pattern | Matches existing DataView pattern in codebase |
| Loading skeletons | Custom CSS animations | shadcn `Skeleton` component | Already styled and animated with shimmer effect |
| Error state UI | Custom error layouts | Consistent pattern with centered card, icon, heading, body, optional action | Matches existing empty state patterns |

## Common Pitfalls

### Pitfall 1: Major vs Minor Units Mismatch
**What goes wrong:** Public API returns amounts in major units (e.g., 150.50) but developer uses `createMoney()` which expects minor units (e.g., 15050), resulting in prices displayed 100x too high.
**Why it happens:** The authenticated API returns minor units, so it's easy to assume all APIs do the same.
**How to avoid:** The public response `IPublicQuoteResponse` has `totals.subTotal`, `totals.taxTotal`, `totals.total` in major units (decimals). Use `createMoneyFromDecimal()` or format directly with `Intl.NumberFormat`. Same for `unitPrice` and `lineTotal` on line items.
**Warning signs:** Prices showing as hundreds of pounds when they should be a few pounds.

### Pitfall 2: Public Route Must Precede Catch-All
**What goes wrong:** The catch-all `<Route path="*" element={<Navigate to="/dashboard" replace />} />` redirects `/quote/view/:token` to dashboard before the public route is matched.
**Why it happens:** Route ordering matters in React Router. The catch-all is currently last, but the public route must be placed before it and outside the AuthenticatedLayout wrapper.
**How to avoid:** Place the public route between the `/login` route and the `AuthenticatedLayout` wrapper in App.tsx.
**Warning signs:** Navigating to `/quote/view/abc123` redirects to `/dashboard` or login page.

### Pitfall 3: Redux Store Required Even for Public Pages
**What goes wrong:** Public RTK Query slice needs the Redux store provider to work. If the public page is rendered completely outside the `<Provider>`, RTK Query hooks will crash.
**Why it happens:** The public page is outside AuthenticatedLayout but still inside `<Provider store={store}>` in App.tsx (which wraps all routes). This is fine as-is.
**How to avoid:** Ensure the public route is inside `<BrowserRouter>` and `<Provider>` in App.tsx (it will be, since those wrap all routes). The public API slice needs to be added to the Redux store's middleware and reducer.
**Warning signs:** "Could not find store" errors when loading public page.

### Pitfall 4: Public API Slice Registration
**What goes wrong:** Creating a new `createApi` instance but forgetting to add its reducer and middleware to the Redux store.
**Why it happens:** Existing code only has one `apiSlice`. A second `createApi` needs its own reducer path and middleware registration.
**How to avoid:** Add `publicApiSlice.reducer` to the store's reducer map and `publicApiSlice.middleware` to the middleware chain in the store configuration.
**Warning signs:** RTK Query hooks return undefined or stale data; network requests not made.

### Pitfall 5: Authenticated Quote Response Missing Token Info
**What goes wrong:** Adding `firstViewedAt` to the token entity but not exposing it through the authenticated quote response, so the tradesperson's QuoteActionStrip can't display the "Viewed" badge.
**Why it happens:** The authenticated quote endpoint (`GET /v1/quote/:quoteId`) returns quote data but doesn't include token information.
**How to avoid:** Either add a separate endpoint to fetch token status for a quote, or include `firstViewedAt` in the authenticated quote response. The simplest approach is to add a `viewedAt?: string` field to the quote response that pulls from the most recent non-revoked token's `firstViewedAt`.
**Warning signs:** "Viewed" badge never shows even after customer views the quote.

## Code Examples

### Public API Slice
```typescript
// trade-flow-ui/src/features/public-quote/api/publicQuoteApi.ts
import { createApi, fetchBaseQuery } from "@reduxjs/toolkit/query/react";
import type { PublicQuoteResponse, StandardResponse } from "./types";

const publicBaseQuery = fetchBaseQuery({
  baseUrl: import.meta.env.VITE_API_BASE_URL || "http://localhost:3000",
});

export const publicQuoteApi = createApi({
  reducerPath: "publicQuoteApi",
  baseQuery: publicBaseQuery,
  endpoints: (builder) => ({
    getPublicQuote: builder.query<PublicQuoteResponse, string>({
      query: (token) => `/v1/public/quote/${token}`,
      transformResponse: (response: StandardResponse<PublicQuoteResponse>) => {
        if (response.data && response.data.length > 0) {
          return response.data[0];
        }
        throw new Error("No quote data returned");
      },
    }),
  }),
});

export const { useGetPublicQuoteQuery } = publicQuoteApi;
```

### Public Quote Page Structure
```typescript
// trade-flow-ui/src/pages/PublicQuotePage.tsx
import { useParams } from "react-router-dom";
import { useGetPublicQuoteQuery } from "@/features/public-quote";
import { PublicQuoteCard } from "@/features/public-quote/components/PublicQuoteCard";
import { PublicQuoteSkeleton } from "@/features/public-quote/components/PublicQuoteSkeleton";
import { PublicQuoteError } from "@/features/public-quote/components/PublicQuoteError";

export default function PublicQuotePage() {
  const { token } = useParams<{ token: string }>();
  const { data, isLoading, error } = useGetPublicQuoteQuery(token!, { skip: !token });

  return (
    <div className="min-h-screen bg-background py-6 sm:py-12">
      <div className="mx-auto max-w-3xl p-4 sm:p-6">
        {isLoading && <PublicQuoteSkeleton />}
        {error && <PublicQuoteError error={error} />}
        {data && <PublicQuoteCard quote={data} />}
      </div>
    </div>
  );
}
```

### firstViewedAt Repository Method
```typescript
// In QuoteTokenRepository
public async updateFirstViewedAt(tokenId: string): Promise<void> {
  await this.writer.updateOne<IQuoteTokenEntity>(
    QuoteTokenRepository.COLLECTION,
    { _id: new ObjectId(tokenId), firstViewedAt: { $exists: false } } as never,
    { $set: { firstViewedAt: new Date() } } as never,
  );
}
```

### Responsive Line Items Pattern
```typescript
// Desktop table (hidden on mobile)
<div className="hidden md:block">
  <Table>
    <TableHeader>
      <TableRow>
        <TableHead>Item</TableHead>
        <TableHead className="text-right">Qty</TableHead>
        <TableHead className="text-right">Unit Price</TableHead>
        <TableHead className="text-right">Total</TableHead>
      </TableRow>
    </TableHeader>
    <TableBody>
      {lineItems.map((item) => (
        <TableRow key={item.name}>
          <TableCell>
            <div className="flex items-center gap-2">
              {item.name}
              <Badge variant="outline" className="text-xs">{typeLabels[item.type]}</Badge>
            </div>
          </TableCell>
          <TableCell className="text-right">{item.quantity}</TableCell>
          <TableCell className="text-right">{formatDecimal(item.unitPrice)}</TableCell>
          <TableCell className="text-right font-semibold">{formatDecimal(item.lineTotal)}</TableCell>
        </TableRow>
      ))}
    </TableBody>
  </Table>
</div>

// Mobile cards (hidden on desktop)
<div className="block md:hidden space-y-4">
  {lineItems.map((item) => (
    <Card key={item.name}>
      <CardContent>
        <div className="flex items-start justify-between gap-2">
          <span className="font-semibold">{item.name}</span>
          <Badge variant="outline" className="text-xs">{typeLabels[item.type]}</Badge>
        </div>
        <div className="flex items-center justify-between text-sm">
          <span className="text-muted-foreground">{item.quantity} x {formatDecimal(item.unitPrice)}</span>
          <span className="font-semibold">{formatDecimal(item.lineTotal)}</span>
        </div>
      </CardContent>
    </Card>
  ))}
</div>
```

### Viewed Badge on QuoteActionStrip
```typescript
// Added to QuoteActionStrip, inline with existing action buttons
{quote.viewedAt && (
  <span className="flex items-center gap-1 text-xs text-muted-foreground">
    <Eye className="h-3 w-3" />
    <span>Viewed</span>
    <span className="text-muted-foreground/60">·</span>
    <span>{format(parseISO(quote.viewedAt), "d MMM 'at' h:mm a")}</span>
  </span>
)}
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Separate store for public pages | Same Redux store, separate API slice | RTK Query pattern | Public pages share store but use unauthenticated API slices |
| Window.fetch for public calls | RTK Query with separate baseQuery | Project convention | Consistent data fetching pattern, caching, loading states |

## Open Questions

1. **How should authenticated quote response expose firstViewedAt?**
   - What we know: The tradesperson's QuoteDetailPage uses `useGetQuoteQuery` which hits the authenticated `GET /v1/quote/:quoteId` endpoint. The token's `firstViewedAt` is on a separate collection (quote_tokens).
   - What's unclear: Should the authenticated quote response include a `viewedAt` field (joined from quote_tokens), or should there be a separate API call to get token status?
   - Recommendation: Add `viewedAt?: string` to the authenticated quote response. The backend can look up the most recent non-revoked token for the quote and include its `firstViewedAt`. This avoids an extra API call on the frontend and keeps the viewed badge simple.

2. **Currency code for public page formatting**
   - What we know: The public API response does not include a currency code. The only supported currency is GBP. `IPublicQuoteResponse` returns amounts as decimal numbers (major units).
   - What's unclear: Should the API return the currency code, or should the frontend hardcode GBP?
   - Recommendation: For now, use `useCurrencyFormatter("GBP")` which already exists. When multi-currency support is added, the public API response should include a `currency` field. This is a minor future enhancement and doesn't block Phase 17.

3. **Error response structure for business name**
   - What we know: Current error responses use `{ code, message, details }` where details is a string. Business name needs to be in the error response for expired/revoked tokens.
   - What's unclear: Use the `details` field for business name (simple) or add a structured `meta` field (more correct)?
   - Recommendation: Use `details` field as a JSON string (e.g., `JSON.stringify({ businessName: "Smith Plumbing" })`) or just the business name string directly. The frontend can parse it. Simple approach: put business name directly in `details` field since it's already a string and the frontend only needs this one piece of info.

## Sources

### Primary (HIGH confidence)
- `trade-flow-api/src/quote-token/controllers/public-quote.controller.ts` -- Verified existing public endpoint, response mapping, error handling
- `trade-flow-api/src/quote-token/responses/public-quote.response.ts` -- Verified IPublicQuoteResponse interface (amounts in major units)
- `trade-flow-api/src/quote-token/entities/quote-token.entity.ts` -- Verified current entity (no firstViewedAt yet)
- `trade-flow-ui/src/services/api.ts` -- Verified existing API slice with auth headers
- `trade-flow-ui/src/App.tsx` -- Verified route structure and AuthenticatedLayout pattern
- `trade-flow-ui/src/hooks/useCurrency.ts` -- Verified `useBusinessCurrency` depends on auth, `useCurrencyFormatter` does not
- `trade-flow-ui/src/features/quotes/components/QuoteLineItemsCardList.tsx` -- Verified mobile card pattern
- `trade-flow-ui/src/features/quotes/components/QuoteActionStrip.tsx` -- Verified action strip where "Viewed" badge goes
- `trade-flow-ui/src/pages/QuoteDetailPage.tsx` -- Verified quote detail page structure
- `.planning/phases/17-customer-quote-page/17-UI-SPEC.md` -- UI design contract with spacing, typography, color, copy

### Secondary (MEDIUM confidence)
- RTK Query createApi documentation -- Multiple API slices pattern with separate reducerPath

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH - All libraries already in project, no new dependencies needed
- Architecture: HIGH - Patterns verified against existing codebase code
- Pitfalls: HIGH - Identified from direct code analysis (major/minor units, route ordering, store registration)
- API changes: HIGH - Entity, DTO, repository, controller patterns all verified from existing code

**Research date:** 2026-03-16
**Valid until:** 2026-04-16 (stable -- no fast-moving dependencies)
