---
phase: 39-welcome-dashboard-and-final-cleanup
reviewed: 2026-04-07T00:00:00Z
depth: standard
files_reviewed: 21
files_reviewed_list:
  - trade-flow-ui/src/App.tsx
  - trade-flow-ui/src/components/PrerequisiteAlert.tsx
  - trade-flow-ui/src/components/layouts/DashboardLayout.tsx
  - trade-flow-ui/src/features/business/components/CreateBusinessForm.tsx
  - trade-flow-ui/src/features/dashboard/components/ChecklistItem.tsx
  - trade-flow-ui/src/features/dashboard/components/GettingStartedChecklist.tsx
  - trade-flow-ui/src/features/dashboard/components/WelcomeSection.tsx
  - trade-flow-ui/src/features/dashboard/components/index.ts
  - trade-flow-ui/src/features/dashboard/hooks/index.ts
  - trade-flow-ui/src/features/dashboard/hooks/useGettingStarted.ts
  - trade-flow-ui/src/features/dashboard/index.ts
  - trade-flow-ui/src/hooks/index.ts
  - trade-flow-ui/src/pages/CustomersPage.tsx
  - trade-flow-ui/src/pages/DashboardPage.tsx
  - trade-flow-ui/src/pages/ItemsPage.tsx
  - trade-flow-ui/src/pages/JobsPage.tsx
  - trade-flow-ui/src/pages/QuotesPage.tsx
  - trade-flow-ui/src/store/index.ts
  - trade-flow-ui/src/types/api.types.ts
  - trade-flow-api/src/quote/services/quote-email-sender.service.ts
  - trade-flow-api/src/user/enums/onboarding-step.enum.ts
findings:
  critical: 2
  warning: 3
  info: 3
  total: 8
status: issues_found
---

# Phase 39: Code Review Report

**Reviewed:** 2026-04-07
**Depth:** standard
**Files Reviewed:** 21
**Status:** issues_found

## Summary

This phase introduces the welcome dashboard, getting-started checklist, and associated cleanup work. The new dashboard feature code (`WelcomeSection`, `GettingStartedChecklist`, `ChecklistItem`, `useGettingStarted`) is clean and well-structured. The onboarding step enum and routing changes in `App.tsx` are straightforward.

Two critical issues were found in the backend `QuoteEmailSender` service: an XSS vulnerability in the HTML sanitizer and a hardcoded currency symbol. Three warnings cover a non-null assertion crash risk, a stale Redux serializable-check configuration, and a missing `as`-type guard in `CreateBusinessForm`. Three info items cover dead code.

## Critical Issues

### CR-01: XSS via unsanitized `href` on allowed `<a>` tags in `sanitizeHtml`

**File:** `trade-flow-api/src/quote/services/quote-email-sender.service.ts:108-119`

**Issue:** `sanitizeHtml` strips disallowed HTML tags but passes allowed tags through verbatim, including all their attributes. An attacker who controls the quote email message body (e.g., a future injection or direct API call) can supply `<a href="javascript:alert(document.cookie)">click</a>`. Because `a` is in the allow-list and the full tag match is returned unchanged, this `href` survives sanitization and renders as a live link in the recipient's email client.

The regex `<\/?([a-zA-Z][a-zA-Z0-9]*)\b[^>]*>` captures but does not strip attributes. Line 115 returns `match` — the full original tag including the `href`.

**Fix:** Strip all attributes from allowed tags (or explicitly allow only safe attributes like `href` after validating it does not begin with `javascript:` or `data:`):

```typescript
private sanitizeHtml(html: string): string {
  const allowedTags = ["p", "br", "strong", "em", "ul", "ol", "li"];
  // Allowed tags with their permitted attributes
  const allowedTagsWithAttrs: Record<string, string[]> = {
    a: ["href", "title"],
  };
  const safeHrefPattern = /^https?:\/\//i;

  const tagPattern = /<\/?([a-zA-Z][a-zA-Z0-9]*)\b([^>]*)>/g;

  return html.replace(tagPattern, (match, tagName: string, attrs: string) => {
    const lowerTag = tagName.toLowerCase();
    if (allowedTags.includes(lowerTag)) {
      return `<${tagName}>`;
    }
    if (lowerTag in allowedTagsWithAttrs) {
      // Strip attrs to only the permitted set with safe values
      const hrefMatch = /href="([^"]*)"/.exec(attrs);
      if (hrefMatch && safeHrefPattern.test(hrefMatch[1])) {
        return `<${tagName} href="${hrefMatch[1]}">`;
      }
      return `<${tagName}>`;
    }
    return "";
  });
}
```

Alternatively, replace this hand-rolled sanitizer with the `sanitize-html` npm package which handles these edge cases correctly.

---

### CR-02: Hardcoded `$` currency symbol in quote email regardless of business currency

**File:** `trade-flow-api/src/quote/services/quote-email-sender.service.ts:52`

**Issue:** The total amount displayed in the quote email always uses a US dollar sign:

```typescript
total: `$${quote.totals.total.toMajorUnits().toFixed(2)}`,
```

Trade Flow supports multiple currencies (GBP, EUR, AUD, etc.). A UK plumber quoting in GBP will have the email show `$1,200.00` instead of `£1,200.00`. The business currency is available via `business.currency` (ISO 4217 code). The correct symbol or formatting must be derived from that value.

**Fix:** Use the business currency to format the total. The `business` object is already fetched on line 37:

```typescript
// Option 1: derive symbol from ISO code
const currencySymbols: Record<string, string> = {
  GBP: "£",
  EUR: "€",
  AUD: "A$",
  USD: "$",
  // extend as needed
};
const symbol = currencySymbols[business.currency] ?? business.currency;
total: `${symbol}${quote.totals.total.toMajorUnits().toFixed(2)}`,

// Option 2: use Intl.NumberFormat for locale-aware formatting
total: new Intl.NumberFormat("en", {
  style: "currency",
  currency: business.currency,
}).format(quote.totals.total.toMajorUnits()),
```

`Intl.NumberFormat` is the most correct approach as it handles locale-specific formatting rules.

---

## Warnings

### WR-01: Non-null assertion `selectedCustomer!` can crash if dialog callback fires with null customer

**File:** `trade-flow-ui/src/pages/CustomersPage.tsx:139`

**Issue:** `onEdit` is passed as `() => handleEditCustomer(selectedCustomer!)`. The `selectedCustomer` state is initialized to `null` and is only set to a customer object inside `handleViewCustomer`. If `onEdit` is ever called before `handleViewCustomer` runs — for example if `CustomerDetailsDialog` exposes the edit button when `customer` is `null`, or if component internals change — this will crash with "Cannot read properties of null".

The `!` assertion suppresses the TypeScript error without adding any runtime safety. The project's memory explicitly flags "Avoid as type assertions — prefer type guards/validation over `as` casts", and the same applies to non-null assertions.

**Fix:** Add a runtime guard:

```typescript
onEdit={() => {
  if (selectedCustomer) {
    handleEditCustomer(selectedCustomer);
  }
}}
```

---

### WR-02: Redux serializable-check config references a non-existent `auth` slice

**File:** `trade-flow-ui/src/store/index.ts:16-17`

**Issue:**

```typescript
ignoredActions: ["auth/setUser"],
ignoredPaths: ["auth.user"],
```

There is no `auth` reducer registered in the store. The auth state is managed entirely by React Context (`AuthProvider`), not by Redux. These entries suppress serialization warnings for a Redux path that does not exist, silently masking the config's intent. If a future developer adds an `auth` Redux slice with a non-serializable Firebase `User` object, they will not see the warning they would expect to see.

**Fix:** Remove both stale entries since the `auth` slice is managed via React Context, not Redux:

```typescript
serializableCheck: {
  // No non-serializable Redux state — Firebase user is in React Context, not Redux
},
```

---

### WR-03: `as` type assertion used to access error message in `CreateBusinessForm`

**File:** `trade-flow-ui/src/features/business/components/CreateBusinessForm.tsx:89-91`

**Issue:**

```typescript
(error as { data?: { errors?: { message: string }[] } }).data
  ?.errors?.[0]?.message
```

The project memory explicitly states: "Avoid `as` type assertions — prefer type guards/validation over `as` casts." The `error` here is the RTK Query error type (`FetchBaseQueryError | SerializedError`). The `as` cast bypasses TypeScript's type narrowing and could silently return `undefined` in unexpected ways.

**Fix:** Use a type guard:

```typescript
function isFetchBaseQueryError(error: unknown): error is { data: { errors?: { message: string }[] } } {
  return (
    typeof error === "object" &&
    error !== null &&
    "data" in error &&
    typeof (error as Record<string, unknown>).data === "object"
  );
}

// In JSX:
{error && (
  <Alert variant="destructive">
    <AlertDescription>
      {isFetchBaseQueryError(error)
        ? error.data.errors?.[0]?.message ?? "Failed to create business"
        : "Failed to create business"}
    </AlertDescription>
  </Alert>
)}
```

---

## Info

### IN-01: Dead `description` field in `prerequisiteMessages.business` config object

**File:** `trade-flow-ui/src/components/PrerequisiteAlert.tsx:21`

**Issue:** The `prerequisiteMessages.business` object defines a `description` field (`"You need to create a business before you can "`), but the business-prerequisite branch (lines 47-68) never uses it. The description is constructed inline as JSX using `contextActionText[context]`. The `description` property on the `business` entry is unreferenced dead code.

**Fix:** Remove the unused `description` field from `prerequisiteMessages.business` to keep the config object consistent with how it is actually consumed:

```typescript
const prerequisiteMessages: Record<
  PrerequisiteType,
  { title: string; description: string; link: string; linkText: string }
> = {
  business: {
    title: "Business required",
    // description removed -- not used; business branch uses contextActionText inline
    link: "/business",
    linkText: "Create Business",
  },
  customers: {
    title: "Customers required",
    description:
      "You need at least one customer before you can create jobs. Customers are who you perform work for.",
    link: "/customers",
    linkText: "Add Customer",
  },
};
```

Update the `Record` type accordingly to make `description` optional or remove it from the `business` variant.

---

### IN-02: `saveCustomerEmail` silently swallows exceptions without logging

**File:** `trade-flow-api/src/quote/services/quote-email-sender.service.ts:87-89`

**Issue:**

```typescript
} catch {
  // Non-critical: log but don't fail the send operation
}
```

The comment says "log but don't fail" but no logging occurs. If `customerRetriever.findByIdOrFail` or `customerUpdater.update` throws (e.g., database timeout, authorization failure), the failure is silently discarded with no observability. This makes diagnosing "why did the customer email not get saved?" impossible in production.

**Fix:** Add a logger call. The class already has access to NestJS DI — inject `AppLogger`:

```typescript
private async saveCustomerEmail(authUser: IUserDto, customerId: string, email: string): Promise<void> {
  try {
    const customer = await this.customerRetriever.findByIdOrFail(authUser, customerId);
    await this.customerUpdater.update(authUser, customer, { email });
  } catch (error) {
    this.logger.warn("Failed to save customer email after quote send", { customerId, error });
  }
}
```

---

### IN-03: `console.error` used directly in `DashboardLayout` sign-out error handler

**File:** `trade-flow-ui/src/components/layouts/DashboardLayout.tsx:95`

**Issue:**

```typescript
} catch (error) {
  console.error("Error signing out:", error);
}
```

Project conventions use `console.log` / `console.error` in the frontend (no dedicated logger), so this is not a violation of convention. However the sign-out error is silently swallowed without any user feedback. If `signOut(auth)` fails (network error, Firebase misconfiguration), the user sees nothing — no toast, no error message — and may believe they have logged out when they have not.

**Fix:** Surface the error to the user with a toast:

```typescript
const handleSignOut = async () => {
  try {
    await signOut(auth);
  } catch (error) {
    console.error("Error signing out:", error);
    toast.error("Sign out failed", {
      description: "Please try again or refresh the page.",
    });
  }
};
```

---

_Reviewed: 2026-04-07_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: standard_
