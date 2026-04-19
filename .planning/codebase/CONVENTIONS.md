# Coding Conventions

**Analysis Date:** 2026-02-21

This codebase consists of two separate projects with distinct conventions: a NestJS backend API and a React frontend.

## Backend (API) Conventions

### File Naming Patterns

**Backend uses kebab-case with type suffixes:**
- Modules: `[name].module.ts` → `business.module.ts` in `src/business/`
- Controllers: `[name].controller.ts` → `business.controller.ts` in `src/business/controllers/`
- Services: `[name]-[action].service.ts` → `business-creator.service.ts`, `business-retriever.service.ts`, `business-updater.service.ts`, `business-deleter.service.ts` in `src/business/services/`
- Repositories: `[name].repository.ts` → `business.repository.ts` in `src/business/repositories/`
- DTOs: `[name].dto.ts` → `business.dto.ts` in `src/business/data-transfer-objects/`
- Responses: `[name].response.ts` → `business.response.ts` in `src/business/responses/`
- Entities: `[name].entity.ts` → `business.entity.ts` in `src/business/entities/`
- Enums: `[name].enum.ts` → `business-status.enum.ts` in `src/business/enums/`
- Errors: `[name].error.ts` → `invalid-request.error.ts` in `src/core/errors/`
- Utilities: `[name].utility.ts` → `handle-error.utility.ts` in `src/core/errors/`
- Constants: `[name].constant.ts` → `errors-map.constant.ts` in `src/core/errors/`
- Interfaces: `[name].interface.ts` → `jwt-user.interface.ts` in `src/auth/interfaces/`
- Test files: `[name].spec.ts` → `business-creator.service.spec.ts` in `src/business/test/services/`

**File structure from `src/ping/` example:**
```
src/ping/
├── controllers/
│   └── ping.controller.ts
├── services/
│   └── ping.service.ts
├── repositories/
│   └── ping.repository.ts
├── data-transfer-objects/
│   └── ping.dto.ts
├── responses/
│   └── ping.response.ts
├── entities/
│   └── ping.entity.ts
├── ping.module.ts
└── test/
    ├── services/
    │   └── ping.service.spec.ts
    └── mocks/
```

### Class and Interface Naming

**Backend naming patterns:**
- DTO Interfaces: `I[Name]Dto` → `IBusinessDto`, `IUserDto` in `src/business/data-transfer-objects/business.dto.ts`
- Response Interfaces: `I[Name]Response` → `IPingResponse` in `src/ping/responses/ping.response.ts`
- Generic Interfaces: `I[Name]` → `IResponse`, `IJwtUser` in `src/core/response/response.interface.ts`
- Service Classes: `[Name][Action]` → `BusinessCreator`, `BusinessRetriever`, `BusinessUpdater`, `BusinessDeleter` in `src/business/services/`
- Repository Classes: `[Name]Repository` → `BusinessRepository` in `src/business/repositories/business.repository.ts`
- Controller Classes: `[Name]Controller` → `BusinessController` in `src/business/controllers/business.controller.ts`
- Module Classes: `[Name]Module` → `BusinessModule` in `src/business/business.module.ts`
- Enum Names: Singular → `BusinessStatus`, `ErrorCodes` in `src/business/enums/business-status.enum.ts`

**Enum value patterns:**
- Keys: UPPER_SNAKE_CASE → `ACTIVE`, `INACTIVE`
- Values: lowercase strings → `"active"`, `"inactive"`
```typescript
export enum BusinessStatus {
  ACTIVE = "active",
  INACTIVE = "inactive",
}

export enum ErrorCodes {
  RESOURCE_NOT_FOUND = "CORE_0",
  ACTION_NOT_ALLOWED = "CORE_1",
}
```

### Functions and Variables

- **Function names**: camelCase → `createPing`, `findBusinessById` in `src/ping/services/ping.service.ts`
- **Variables**: camelCase and descriptive → `businessId`, `authUser`, `createdBusiness` in controllers/services
- **Unused parameters**: Prefix with underscore → `_context`, `_options` in `src/core/services/app-logger.service.ts`
- **Private methods**: Prefix with no underscore, use verb-noun → `mapToResponse()`, `validateInput()` in controllers/services
- **Constants**: UPPER_SNAKE_CASE in files or CONSTANT_CASE for inline → `COLLECTION = "businesses"` static readonly in repository

### Import Organization

**Backend import order:**
1. External dependencies → `import { Injectable } from "@nestjs/common"`
2. Path aliases (mandatory) → `import { IUserDto } from "@user/data-transfer-objects/user.dto"`
3. Type imports → `import type { ObjectId } from "mongodb"`

**Path aliases available:**
- `@auth/*` → `src/auth/`
- `@business/*` → `src/business/`
- `@business-test/*` → `src/business/test/`
- `@core/*` → `src/core/`
- `@core-test/*` → `src/core/test/`
- `@customer/*` → `src/customer/`
- `@email/*` → `src/email/`
- `@item/*` → `src/item/`
- `@job/*` → `src/job/`
- `@migration/*` → `src/migration/`
- `@ping/*` → `src/ping/`
- `@quote/*` → `src/quote/`
- `@tax-rate/*` → `src/tax-rate/`
- `@user/*` → `src/user/`

Example from `src/ping/controllers/ping.controller.ts`:
```typescript
import { JwtAuthGuard } from "@auth/auth.guard";
import { Controller, Get, Inject, UseGuards } from "@nestjs/common";
import { PingService } from "@ping/services/ping.service";
import { IPingResponse } from "../responses/ping.response";
```

### Error Handling

**Pattern: Custom error classes extending Error**

Error classes in `src/core/errors/`:
- `InvalidRequestError` → HTTP 422 (Unprocessable Entity)
- `ResourceNotFoundError` → HTTP 404 (Not Found)
- `ForbiddenError` → HTTP 403 (Forbidden)
- `InternalServerError` → HTTP 500 (Internal Server Error)

Each error class provides:
- `getCode()` → returns `ErrorCodes` enum value
- `getMessage()` → returns human-readable message
- `getDetails()` → returns additional context

Error handling utility `src/core/errors/handle-error.utility.ts` converts custom errors to HTTP exceptions:
```typescript
export const createHttpError = (error: unknown): Error => {
  if (error instanceof InvalidRequestError) {
    const response: IResponse = {
      errors: [
        {
          code: error.getCode(),
          message: error.getMessage(),
          details: error.getDetails(),
        },
      ],
    };
    return new HttpException(response, HttpStatus.UNPROCESSABLE_ENTITY);
  }
  // ... handle other error types
};
```

### Logging

**Framework:** Pino logger wrapped by NestJS

Logger service in `src/core/services/app-logger.service.ts`:
- `log(message, context?)` → info level
- `error(message, trace?, context?)` → error level
- `warn(message, context?)` → warn level
- `debug(message, context?)` → debug level
- `verbose(message, context?)` → trace level

Pattern for instantiation:
```typescript
private readonly logger = new AppLogger(ClassName.name);
this.logger.log("Operation completed", { userId, result });
```

### Code Style

**Prettier Configuration** (`src/core/errors/handle-error.utility.ts` style):
- Print Width: 125 characters
- Tab Width: 2 spaces
- Tabs: false (spaces)
- Semicolons: true
- Trailing Commas: all
- Quote Style: double quotes
- Arrow Function Parens: always
- Bracket Spacing: true
- End of Line: LF

**ESLint Rules:**
- `@typescript-eslint/interface-name-prefix` → off (allows I prefix)
- `@typescript-eslint/explicit-function-return-type` → off
- `@typescript-eslint/explicit-module-boundary-types` → off
- `@typescript-eslint/no-explicit-any` → warn
- `@typescript-eslint/no-unused-vars` → warn with `argsIgnorePattern: "^_"`
- `prettier/prettier` → error (enforces prettier formatting)

**TypeScript Configuration:**
- Target: ES2021
- Module: CommonJS
- Strict Mode: Enabled (`strictNullChecks`, `noImplicitAny`, `strictBindCallApply`)
- Emit Decorator Metadata: true (for NestJS)
- Experimental Decorators: true (for NestJS)
- Source Maps: true

**Variable Declarations:**
- Use `const` by default
- Use `let` only when accumulating state
- Never use `var`

### Comments

**JSDoc pattern** (from `src/core/services/app-logger.service.ts`):
```typescript
/**
 * Application logger that wraps PinoLogger with the NestJS Logger API.
 *
 * Usage:
 *   private readonly logger = new AppLogger(ClassName.name);
 *   this.logger.log("Operation completed", { userId, result });
 */
@Injectable({ scope: Scope.TRANSIENT })
export class AppLogger implements LoggerService {
```

**Method documentation** style:
```typescript
/**
 * Set the logging context (typically the class name)
 */
public setContext(context: string): void {
```

---

## Query Filtering & Sorting Convention

Established in Phase 54 (v1.9). Applies to all future list endpoints.

### URL Parameter Format

| Concern | Format | Example |
|---------|--------|---------|
| Filter | `?filter:<field>:<operator>=<value>` | `?filter:subscription.status:eq=active` |
| Sort ascending | `?sort=<field>` | `?sort=email` |
| Sort descending | `?sort=-<field>` | `?sort=-createdAt` |
| Search | `?search=<term>` | `?search=john` |
| Pagination | `?page=<n>&limit=<n>` | `?page=1&limit=20` |

### Filter Operators

| Operator | Meaning | Multi-value? |
|----------|---------|-------------|
| `eq` | equals | No |
| `nq` | not equals | No |
| `lt` | less than | No |
| `gt` | greater than | No |
| `le` | less or equal | No |
| `ge` | greater or equal | No |
| `in` | in list | Yes (comma-separated) |
| `bt` | between | Yes (comma-separated, exactly 2 values) |

### Implementation Files

- `@core/interfaces/query-filter.interface` — `IQueryFilter` interface and `FilterOperator` type
- `@core/utilities/query-filter-parser.utility` — `parseQueryFilters()` and `escapeRegex()`
- `@core/utilities/query-sort-parser.utility` — `parseQuerySort()`
- `@core/data-transfer-objects/base-query-options.dto` — `IBaseQueryOptionsDto` with `filters`, `sort`, `search`, `pagination`

### Usage Pattern

Controllers parse URL params and build the options DTO:

```typescript
const filters = parseQueryFilters(query as Record<string, string>);
const sort = parseQuerySort(query.sort as string | undefined);
const options: IBaseQueryOptionsDto = { filters, sort, search: query.search, pagination: { page, limit } };
```

Each endpoint validates allowed filter field names against its own allowlist — the generic parser accepts any field name.

---

## Frontend (UI) Conventions

### File Naming Patterns

**Frontend uses PascalCase for components, camelCase for hooks/utilities:**
- Components: PascalCase → `CustomersCardList.tsx`, `BusinessFormDialog.tsx` in `src/features/customers/components/`
- Hooks: camelCase with `use` prefix → `useAuth.ts`, `useOnboarding.ts` in `src/hooks/` or feature-specific `src/features/customers/hooks/`
- Utilities: kebab-case → `date-helpers.ts`, `format-currency.ts` in `src/lib/`
- Redux slices: camelCase with `Slice` suffix → `authSlice.ts`, `customersSlice.ts` in `src/store/slices/`
- Types/Interfaces: PascalCase → `Customer`, `AuthContextType` in `src/types/` or feature-specific

### Component Structure

**Functional components only - no class components**

Pattern from `src/features/customers/components/CustomersCardList.tsx`:
```typescript
import type { Customer } from "@/types";

interface CustomersCardListProps {
  customers: Customer[];
  onViewCustomer: (customer: Customer) => void;
  onEditCustomer: (customer: Customer) => void;
  onToggleStatus: (customer: Customer) => void;
}

export function CustomersCardList({
  customers,
  onViewCustomer,
  onEditCustomer,
  onToggleStatus,
}: CustomersCardListProps) {
  return (
    // JSX
  );
}
```

**Props naming conventions:**
- Handler callbacks: `on[Action]` → `onViewCustomer`, `onEditCustomer`, `onToggleStatus`
- Data props: descriptive names → `customers`, `selectedItem`, `isLoading`
- Boolean props: `is[State]` or `can[Action]` → `isOpen`, `isLoading`, `canDelete`

### Import Organization

**Frontend import order:**
1. External dependencies → `import { Provider } from "react-redux"`
2. Type imports → `import type { Customer } from "@/types"`
3. Internal imports from `@/` alias → `import { Button } from "@/components/ui/button"`
4. Relative imports → `import { localHelper } from "./helpers"`

**Path alias:** `@/` → `src/` (configured in `tsconfig.json`)

Example from `src/App.tsx`:
```typescript
import { Provider } from "react-redux";
import {
  BrowserRouter,
  Navigate,
  Routes,
  Route,
  Outlet,
} from "react-router-dom";

import { ErrorBoundary, PageErrorBoundary } from "@/components/error-boundary";
import { OnboardingDialogs } from "@/components/onboarding";
import type { Customer } from "@/types";
```

### Type Declarations

**Frontend type patterns:**
- Always use `import type` for type imports → `import type { Customer } from "@/types"`
- Use interfaces for component props → `interface CustomersCardListProps { ... }`
- Use types for unions/complex types → `type OnboardingStep = "step1" | "step2"`

### State Management

**Redux patterns:**
- Use `useAppDispatch` and `useAppSelector` from `@/store/hooks` (not raw React Redux hooks)
- RTK Query for API data fetching
- Local React state (`useState`) for UI-only concerns

From `src/store/hooks.ts`:
```typescript
export const useAppDispatch = useDispatch.withTypes<AppDispatch>();
export const useAppSelector = useSelector.withTypes<RootState>();
```

### Styling

**Tailwind CSS utility classes with semantic tokens:**
- Color tokens: `bg-background`, `text-foreground`, `bg-primary`, `text-muted-foreground`
- Layout utilities: `flex`, `gap-`, `space-y-`, `mx-auto`
- Responsive classes: `md:` prefix for desktop (768px breakpoint)
- Mobile-first design approach

**Conditional classes utility:**
- Use `cn()` from `@/lib/utils` for conditional Tailwind classes

**Flex overflow pattern:**
- Always add `min-w-0` to flex children containing truncatable content
- Example from `src/features/customers/components/CustomersCardList.tsx`:
```typescript
<div className="min-w-0 flex-1 space-y-1">
  <span className="truncate font-medium">{customer.name}</span>
</div>
```

**DataView pattern for responsive tables/cards:**
- Use `[Feature]DataView` components that switch between Table (desktop) and Cards (mobile)
- Example: `CustomersDataView` component

### Functions and Variables

- **Function names**: camelCase → `formatCurrency`, `parseDate`, `calculateTax`
- **Variables**: camelCase and descriptive → `isLoading`, `selectedCustomer`, `filteredItems`
- **Constants**: camelCase or UPPER_SNAKE_CASE → `API_BASE_URL`, `defaultTimeout`
- **Boolean variables**: `is`, `has`, `can` prefix → `isOpen`, `hasError`, `canSubmit`

### Comments

**JSDoc pattern for hooks:**
```typescript
/**
 * Retrieve authentication context.
 * @throws Error if used outside AuthProvider
 */
export const useAuth = () => {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error("useAuth must be used within an AuthProvider");
  }
  return context;
};
```

### Validation

**Frontend validation:**
- Use Valibot for schema validation (from `package.json`)
- React Hook Form with Valibot resolvers for form validation (from `@hookform/resolvers`)
- Field-level validation with clear error messages

### Error Boundaries

**Pattern from `src/App.tsx`:**
```typescript
<ErrorBoundary>
  <Provider store={store}>
    <AuthProvider>
      <BrowserRouter>
        {/* Routes with PageErrorBoundary */}
        <PageErrorBoundary>
          <Outlet />
        </PageErrorBoundary>
      </BrowserRouter>
    </AuthProvider>
  </Provider>
</ErrorBoundary>
```

---

## Shared Patterns

### Module Organization

**Backend modules** follow feature-based structure:
- One feature per directory → `src/business/`, `src/customer/`, `src/quote/`
- Each module contains: controllers, services, repositories, DTOs, responses, entities
- Test files in `test/` subdirectory

**Frontend features** follow similar structure:
- One feature per directory → `src/features/customers/`, `src/features/quotes/`
- Each feature contains: components, hooks, api (RTK Query), types
- Shared utilities in `src/lib/`, `src/hooks/`, `src/types/`

### TypeScript Configuration

**Both projects use strict TypeScript:**
- Strict null checks: enabled
- No implicit any: enabled
- Explicit function return types: required
- Path aliases configured in `tsconfig.json`

---

*Convention analysis: 2026-02-21*
