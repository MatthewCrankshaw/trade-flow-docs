# Architecture

**Analysis Date:** 2026-02-21

## Overview

This is a full-stack TypeScript mono-repo for a sole trader business management platform. It consists of two distinct architectural patterns:

- **Backend (Trade Flow API):** NestJS - Strict layered architecture with domain-driven modules
- **Frontend (Trade Flow UI):** React - Feature-based architecture with Redux Toolkit state management

## Backend Architecture (Trade Flow API)

### Pattern Overview

**Overall:** NestJS modular layered architecture with strict separation of concerns.

**Key Characteristics:**
- Controller → Service → Repository layer hierarchy (never bypass layers)
- Feature-based module organization (`@auth`, `@customer`, `@business`, etc.)
- Global CoreModule providing shared infrastructure (database, logging, access control)
- Dependency injection via NestJS modules
- Policy-based access control
- MongoDB as persistence layer

### Backend Layers

**Controller Layer:**
- Purpose: HTTP request handling, input validation, response mapping
- Location: `src/{feature}/controllers/`
- Contains: Request/response DTOs, HTTP method handlers
- Depends on: Services, error handling utilities
- Used by: Express/HTTP layer
- Pattern: Decorators (`@Get`, `@Post`, `@UseGuards`), request/response mapping, try-catch with error utilities

**Service Layer:**
- Purpose: Business logic orchestration, access control enforcement
- Location: `src/{feature}/services/`
- Contains: Creator/Retriever/Updater service classes implementing specific responsibilities
- Depends on: Repositories, policies, access controllers
- Used by: Controllers
- Pattern: Single responsibility principle (one service = one operation type), inject dependencies via constructor

**Repository Layer:**
- Purpose: Database operations, entity-to-DTO mapping
- Location: `src/{feature}/repositories/`
- Contains: MongoDB collection queries, document transformation
- Depends on: MongoDB client services, DTOs, entities
- Used by: Services
- Pattern: MongoDB ObjectId conversion, entity-to-DTO mapping on entry/exit

**Core Module (Global Infrastructure):**
- Location: `src/core/`
- Exports globally via `@Global()` decorator
- Provides:
  - `MongoConnectionService`: Database connection management
  - `MongoDbFetcher`: Query execution service
  - `MongoDbWriter`: Insert/update/delete service
  - `AccessControllerFactory`: Policy enforcement factory
  - `AuthorizedCreatorFactory`: Creation with ownership tracking
  - Shared DTOs, interfaces, value objects, utilities
  - Error handling and response formatting

### Backend Data Flow

**Create Resource Flow:**
1. Controller receives `CreateXRequest` (validated by NestJS pipes)
2. Controller creates new DTO with ID and default values
3. Controller calls `XCreator` service with authenticated user and DTO
4. Service applies business rules, calls access controller for permission check
5. Service delegates to repository
6. Repository converts DTO → Entity, inserts into MongoDB, converts back to DTO
7. Service returns DTO to controller
8. Controller maps DTO → Response object, returns via `createResponse()` utility
9. Response includes array of data items: `{ data: [item] | [], status: "success/error" }`

**Retrieve Resource Flow:**
1. Controller receives request with ID/filter
2. Controller calls `XRetriever` service with authenticated user and query params
3. Service calls repository to fetch
4. Service calls access controller to verify user can read the resource
5. Repository returns DTO (already mapped from entity)
6. Service returns DTO to controller
7. Controller maps to response format, returns via `createResponse()`

**Update Resource Flow:**
1. Controller receives `UpdateXRequest`
2. Controller fetches existing resource via retriever
3. Controller merges update fields with existing values (null = keep original)
4. Controller calls `XUpdater` service with user, existing entity, and update data
5. Service checks access control
6. Service calls repository update
7. Repository applies timestamps, saves, returns updated DTO
8. Controller returns mapped response

### Backend Key Abstractions

**Policy (Access Control):**
- Purpose: Define read/write permissions per resource type
- Examples: `src/customer/policies/customer.policy.ts`, `src/business/policies/business.policy.ts`
- Pattern: Policy class passed to AccessControllerFactory, factory creates controller that enforces rules

**DTOs (Data Transfer Objects):**
- Purpose: Contract between layers, defines shape of data
- Location: `src/{feature}/data-transfer-objects/`
- Pattern: Interface naming convention `IX{Feature}Dto`, used throughout service/controller layers
- Not returned directly; mapped to response types before HTTP response

**Entities:**
- Purpose: MongoDB document structure
- Location: `src/{feature}/entities/`
- Pattern: Interface naming convention `IX{Feature}Entity`, includes MongoDB `_id` and timestamps
- Converted from/to DTO in repository layer only

**Request Objects:**
- Purpose: HTTP request body validation contracts
- Location: `src/{feature}/requests/`
- Validated by NestJS ValidationPipe before reaching controller

**Response Objects:**
- Purpose: HTTP response body structure
- Location: `src/{feature}/responses/`
- Pattern: Returned by controller after mapping from DTO

**Collections:**
- Purpose: Array-like wrapper for DTOs with pagination metadata
- Location: `src/core/collections/`
- Examples: `DtoCollection<T>`, `EntityCollection<T>`

**Value Objects:**
- Purpose: Immutable domain concepts (Money, etc.)
- Location: `src/core/value-objects/`
- Example: `src/core/value-objects/money.value-object.ts`

### Backend Entry Points

**Application Bootstrap:**
- Location: `src/main.ts`
- Triggers: npm run start:dev
- Responsibilities: NestFactory creation, middleware setup (CORS, validation), logging, graceful shutdown

**AppModule:**
- Location: `src/app.module.ts`
- Triggers: NestFactory.create()
- Responsibilities: ConfigModule initialization, feature module imports, global middleware

**Feature Modules:**
- Example: `src/customer/customer.module.ts`
- Pattern: Each feature has a module that imports CoreModule, declares controllers/services/repositories
- Imports CoreModule to access shared infrastructure

### Backend Error Handling

**Strategy:** Custom error classes extending NestJS exceptions, mapped to HTTP status codes

**Patterns:**
- `ResourceNotFoundError`: 404 + RESOURCE_NOT_FOUND code
- `ForbiddenError`: 403 + FORBIDDEN code
- `InvalidRequestError`: 400 + INVALID_REQUEST code
- `InternalServerError`: 500 + INTERNAL_SERVER_ERROR code
- `createHttpError()` utility converts to HTTP exceptions
- All caught in try-catch blocks, mapped via `createResponse()` utility

### Backend Cross-Cutting Concerns

**Logging:**
- Service: `AppLogger` (wraps Pino logger)
- Pattern: Injected as `private readonly logger = new AppLogger(ClassName.name)`
- Used for debug/info messages at key points (queries, business logic)

**Validation:**
- NestJS `ValidationPipe` applied globally in main.ts
- Options: `whitelist: true, forbidNonWhitelisted: true`
- Validates request DTOs before reaching controller

**Authentication:**
- Firebase JWT verification via `JwtAuthGuard`
- Guard extracts token from Authorization header
- Decoded JWT available as `request.user: IUserDto`
- Applied via `@UseGuards(JwtAuthGuard)` decorator on controller methods

**Authorization:**
- Access control via policies (CustomerPolicy, BusinessPolicy, etc.)
- Controller calls service, service calls accessController.canRead/canWrite
- Throws ForbiddenError if not allowed

---

## Frontend Architecture (Trade Flow UI)

### Pattern Overview

**Overall:** React feature-based architecture with Redux Toolkit state management and RTK Query for API caching.

**Key Characteristics:**
- Feature-based directory structure (customers, jobs, items, quotes, auth, etc.)
- Redux Toolkit with RTK Query for API state management
- Context API for authentication and onboarding
- React Router v7 for navigation
- shadcn/ui component library
- Firebase Authentication (JWT tokens)

### Frontend Layers

**Entry Point:**
- Location: `src/main.tsx`
- Creates React root and renders `<App />`

**App Component:**
- Location: `src/App.tsx`
- Purpose: Root route definitions and provider setup
- Sets up: Redux Provider, AuthProvider, BrowserRouter, global error boundary
- Routes:
  - `/login` - Login page (unauthenticated)
  - `/dashboard`, `/customers`, `/jobs`, `/items`, `/quotes`, `/business`, `/settings` - Protected authenticated routes
  - `/support/*` - Support admin routes
  - Fallback redirects to `/dashboard`
- Authenticated routes wrapped in `AuthenticatedLayout` which provides OnboardingProvider

**Providers:**
- Location: `src/providers/auth-provider.tsx`
- Purpose: Firebase authentication state management
- Responsibilities: Monitor auth state changes, reset Redux cache on logout, provide AuthContext
- Wraps: Entire app via `<AuthProvider>`

**Contexts:**
- **AuthContext** (`src/contexts/AuthContext.tsx`): Current user and loading state from Firebase
- **OnboardingContext** (`src/contexts/OnboardingProvider.tsx`): Onboarding dialog state and triggers
- Pattern: Provider wraps authenticated routes to manage onboarding dialogs

### Frontend Data Flow

**Page-to-API Flow:**
1. Page component (e.g., CustomersPage) renders
2. Calls custom hook (e.g., `useCustomersList`) to fetch data
3. Hook uses RTK Query hook (e.g., `useGetCustomersQuery(businessId)`)
4. RTK Query `apiSlice` configured with Firebase token injection in `prepareHeaders`
5. API service (`src/services/api.ts`) defines endpoints and caching strategy
6. Feature-specific API file (e.g., `customerApi.ts`) injects endpoints into apiSlice
7. Hook returns data + loading + error state
8. Component renders with data, loading skeleton, or error state

**State Management Flow:**
- Redux store in `src/store/index.ts` combines:
  - `apiSlice.reducer` - RTK Query cached responses
  - `onboarding` - Onboarding dialog visibility state
  - `onboardingMiddleware` - Listens to auth/business changes to trigger dialogs
- Components use `useAppDispatch` and `useAppSelector` to access Redux state
- Components use RTK Query hooks (auto-generated) to fetch data

**Component Composition:**
- Feature pages import from feature index barrel file
- Example: `import { useCustomersList, useCustomerActions, CustomerFormDialog } from "@/features/customers"`
- Components are co-located with hooks and API definitions
- Shared UI components from `src/components/ui/` (shadcn)
- Layout components from `src/components/layouts/`

### Frontend Key Abstractions

**Features:**
- Purpose: Self-contained domain areas (customers, items, jobs, quotes, auth, business, tax-rates)
- Location: `src/features/{feature}/`
- Structure:
  ```
  src/features/{feature}/
  ├── components/     # React components
  ├── hooks/         # Custom React hooks
  ├── api/           # RTK Query endpoints
  └── index.ts       # Barrel export (re-exports all)
  ```
- Pattern: Each feature is independent, imports shared components and types
- Example: `src/features/customers/` contains CustomerFormDialog, useCustomersList, customerApi

**Hooks:**
- Purpose: Reusable logic encapsulation
- Examples:
  - `useCustomersList()` - fetch, filter, search logic
  - `useCustomerActions()` - create/update/delete mutations
  - `useCurrentBusiness()` - get current business from store/context
- Pattern: Return objects with state + setters and status flags
- Used by: Page components and other hooks

**API Definitions (RTK Query):**
- Location: `src/features/{feature}/api/`
- Pattern: Call `apiSlice.injectEndpoints()` to add endpoints
- Each endpoint includes:
  - `query`: URL builder
  - `transformResponse`: API response → component data shape
  - `invalidatesTags`: Caches to invalidate on mutation
  - `providesTags`: Caches to provide for this query
- Example: `useGetCustomersQuery(businessId)` auto-generated from endpoint

**Pages:**
- Location: `src/pages/`
- Purpose: Route-level components (one per route)
- Pattern: Import feature components/hooks, arrange into layout
- Example: CustomersPage imports CustomersDataView, CustomersFilters, CustomerFormDialog

**Shared Components:**
- Location: `src/components/`
- Subdirectories:
  - `ui/` - shadcn/ui components (Button, Card, Dialog, Table, etc.)
  - `layouts/` - Page layout wrappers (DashboardLayout)
  - `onboarding/` - Onboarding-related components
  - `error-boundary/` - Error boundary components
- Pattern: Composed into feature and page components

**Store Slices:**
- Location: `src/store/slices/`
- Example: `onboardingSlice.ts` - manages dialog visibility state
- Pattern: Redux Toolkit createSlice, minimal state (mainly for UI state like dialogs)
- Accessed via `useAppSelector` hooks

**Services:**
- Location: `src/services/`
- Current files:
  - `api.ts` - Base RTK Query apiSlice configuration
  - `api.ts` - Re-exports generated hooks
  - `userApi.ts` - User-specific endpoints
  - `migrationApi.ts` - Data migration endpoints
- Pattern: Feature-specific API definitions inject into base apiSlice

### Frontend Entry Points

**Application Bootstrap:**
- Location: `src/main.tsx`
- Triggers: npm run dev
- Responsibilities: Create React root, render App component

**App Component:**
- Location: `src/App.tsx`
- Triggers: Root render
- Responsibilities: Provider setup (Redux, Auth, Router), route configuration

**Pages:**
- Examples: `src/pages/CustomersPage.tsx`, `src/pages/JobsPage.tsx`
- Triggers: Route match in React Router
- Responsibilities: Layout composition, feature hook calls, dialog state management

### Frontend Error Handling

**Strategy:** Error boundaries + React Query error states

**Patterns:**
- `ErrorBoundary` wrapper in App for uncaught errors
- `PageErrorBoundary` wrapper per page
- RTK Query hooks return `{ error, status }` - component conditional rendering
- `<PrerequisiteAlert>` component shows missing dependencies (e.g., must create business first)

### Frontend Cross-Cutting Concerns

**Logging:**
- console.log/console.error (no logger library)
- Could be enhanced with error tracking service

**Validation:**
- Form validation via `react-hook-form` + `zod` schemas
- Location: `src/lib/forms/schemas/`
- Server-side validation errors shown in UI toasts

**Authentication:**
- Firebase Authentication (JWT tokens)
- `AuthProvider` monitors auth state via `onAuthStateChanged`
- Tokens injected in RTK Query `prepareHeaders`
- `ProtectedRoute` guards routes - redirects to /login if not authenticated

**Authorization:**
- Backend enforces via policies
- Frontend shows/hides UI based on business ownership
- Example: Can't create customers without a business
- Backend returns 403 if user lacks permission

---

## Synchronization Between Frontend and Backend

**API Contract:**
- Backend: Response objects in `src/{feature}/responses/`
- Frontend: Types in `src/types/` and API response mapping in feature API files
- All API endpoints prefixed with `/v1/` for versioning

**Authentication Flow:**
1. User logs in via Firebase (frontend)
2. Firebase issues JWT token
3. Frontend injects token in all API requests via RTK Query prepareHeaders
4. Backend JwtAuthGuard verifies and decodes token
5. User ID from token used for access control checks

**Feature Implementation Pattern:**
- Backend creates Controller → Service → Repository → Database
- Frontend creates Page → Hooks → Components → API definitions
- Components call hooks, hooks call RTK Query endpoints, endpoints call backend

---

*Architecture analysis: 2026-02-21*
