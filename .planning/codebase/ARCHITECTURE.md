# Architecture

**Analysis Date:** 2026-02-21

## Project Overview

**Trade Flow** is a full-stack sole trader business management platform enabling independent contractors to manage customers, create quotes, track jobs, manage inventory items, and handle tax rates. Two subsystems:

- **Frontend (React 19/TypeScript):** Customer-facing web application built with Vite
- **Backend (NestJS 11/TypeScript):** REST API with MongoDB persistence and Firebase authentication

The monorepo is organized as:
- `/trade-flow-api` - Backend application
- `/trade-flow-ui` - Frontend application

## Pattern Overview

**Overall:** Layered architecture with domain-driven module organization

**Key Characteristics:**
- **Backend:** Strict Controller → Service → Repository layering, never bypassed
- **Frontend:** Feature-based modules with Redux RTK Query for server state, Redux slices for client state
- **Authentication:** Firebase JWT (RS256) with server-side public key validation
- **Data Flow:** Unidirectional requests flow: UI → API → Database → Response → UI state update
- **Validation:** Request DTOs validated at controller boundary using class-validator
- **API Contract:** Standardized response format: `{ data: T[], pagination?, errors? }`

## Backend Architecture (NestJS)

### Layers

**Controllers (`src/{feature}/controllers/` or `src/{feature}/controller/`):**
- **Purpose:** HTTP request/response handling, input validation, response mapping
- **What it does:**
  - Receives HTTP requests decorated with `@Get()`, `@Post()`, `@Patch()`, `@Delete()`
  - Validates request body via decorators like `@Body()`, `@Param()`, `@Query()`
  - Extracts authenticated user from request via `@Req()`
  - Calls service layer methods
  - Maps DTOs to response objects
  - Returns standardized response via `createResponse()` utility
- **Contains:** Controller class + request/response DTOs
- **Depends on:** Services, DTOs, Guards (JwtAuthGuard), error utilities
- **Used by:** HTTP clients
- **Pattern:** One controller per domain, multiple route handlers per controller
- **Example:** `src/business/controllers/business.controller.ts`
  - Routes: `GET /v1/business`, `GET /v1/business/:id`, `POST /v1/business`, `PATCH /v1/business/:id`
  - Wraps service calls in try-catch
  - Maps business DTO to IBusinessResponse
  - Returns `createResponse([businessResponseData])`

**Services (`src/{feature}/services/`):**
- **Purpose:** Business logic orchestration, authorization checks, cross-module coordination
- **What it does:**
  - Implements business rules and workflows
  - Calls repositories to fetch/modify data
  - Enforces authorization policies
  - Coordinates with other services
  - Manages transactions and side effects (e.g., creating default items when business created)
  - NEVER accesses database directly
- **Contains:** Creator/Retriever/Updater service classes
- **Depends on:** Repositories, Policies, other Services, DTOs, Factories
- **Used by:** Controllers, other Services
- **Pattern:** Single-responsibility services (BusinessCreator, BusinessRetriever, BusinessUpdater)
- **Example:** `src/business/services/business-creator.service.ts`
  - Method: `create(authUser: IUserDto, business: IBusinessDto): Promise<IBusinessDto>`
  - Uses AuthorizedCreatorFactory to wrap repository with authorization
  - Calls businessRepository.create()
  - Creates BusinessUserRepository association
  - Assigns ADMIN role
  - Updates onboarding progress
  - Creates default items/tax-rates/job-types
  - Returns created business DTO

**Repositories (`src/{feature}/repositories/`):**
- **Purpose:** Data persistence abstraction, entity-DTO conversion, MongoDB operations
- **What it does:**
  - Executes MongoDB queries via MongoDbFetcher (read) and MongoDbWriter (write)
  - Converts MongoDB entities to DTOs on retrieval
  - Converts DTOs to entities on persistence
  - Handles ObjectId conversions
  - Logs database operations
- **Contains:** Repository class per entity type
- **Depends on:** MongoDbFetcher, MongoDbWriter, Entities, DTOs
- **Used by:** Services only
- **Pattern:** One repository per domain model
- **Example:** `src/business/repositories/business.repository.ts`
  - Methods: `create(dto)`, `findByIdOrFail(id)`, `findByIds(ids)`, `update(id, update)`
  - Maps BusinessEntity to IBusinessDto (and vice versa)
  - Throws ResourceNotFoundError if entity not found

**Core Infrastructure (`src/core/`):**
- **Purpose:** Cross-cutting services and abstractions shared globally
- **Contents:**
  - `services/mongo/` - MongoDB connection, fetching, writing
  - `factories/` - AccessControllerFactory, AuthorizedCreatorFactory
  - `interfaces/` - Base interfaces (ICreatorService, ICreatorRepository, etc.)
  - `errors/` - Custom error types (ForbiddenError, ResourceNotFoundError, etc.)
  - `response/` - Response formatting utilities (createResponse, createErrorResponse)
  - `value-objects/` - Domain concepts (Money)
  - `collections/` - Collection wrappers (DtoCollection, EntityCollection)
  - `policies/` - Base policy class
- **Module:** `src/core/core.module.ts` exports globally via `@Global()` decorator
- **Used by:** All feature modules

### Data Flow

**Create Business Flow:**
```
Frontend:
1. User submits CreateBusinessRequest form
2. Form dispatches createBusiness RTK Query mutation
3. POST /v1/business with {name, country, currency, trade}

Backend:
1. BusinessController.create() receives request + authenticated user
2. Validates request DTO via ValidationPipe
3. Creates new BusinessDto with generated ID
4. Calls BusinessCreator.create(authUser, businessDto)
5. Creator calls AuthorizedCreatorFactory.createFor(repository, policy)
   - Wraps repository.create() with authorization
   - Policy checks: user can create businesses (always true for first business)
6. Repository.create(dto):
   - Converts DTO to Entity
   - Inserts into MongoDB "businesses" collection
   - Converts entity back to DTO
   - Returns created DTO
7. Creator does orchestration:
   - Calls BusinessUserRepository to create ownership link
   - Calls BusinessRoleAssigner to assign ADMIN role
   - Calls OnboardingProgressUpdater to mark step complete
   - Calls DefaultBusinessItemsCreator to create default items
   - Calls DefaultTaxRatesCreator to create default tax rates
   - Calls DefaultJobTypesCreator to create default job types
8. Controller maps BusinessDto to IBusinessResponse
9. Returns createResponse([businessResponse])

Frontend:
1. RTK Query receives response
2. Caches data by "Business" tag
3. Invalidates "Business" tag triggers refetch of dependent queries
4. Component receives updated data via useGetBusinessesQuery()
5. UI re-renders with new business
```

**Get Customers Flow:**
```
Frontend:
1. CustomersPage mounts or calls useGetCustomersQuery(businessId)
2. RTK Query checks cache:
   - If valid, returns cached data
   - If stale/missing, makes GET /v1/customer request

Backend:
1. CustomerController.getByBusinessId() receives request
2. Calls CustomerRetriever.getByBusinessId(authUser, businessId)
3. Retriever calls AccessControllerFactory.createAccessController(repository, policy)
4. Access controller calls policy.canRead(authUser, businessId)
   - Policy checks: user is owner of this business
   - Throws ForbiddenError if not authorized
5. Repository.findByBusinessId(businessId):
   - Queries MongoDB with filter {businessId}
   - Converts entities to DTOs
   - Returns DtoCollection<ICustomerDto>
6. Retriever returns customer DTOs
7. Controller maps to responses
8. Returns createResponse(customerResponses)

Frontend:
1. RTK Query caches response with "Customer" tag
2. Component selector reads from cache
3. UI renders customer list
```

### Key Abstractions

**Service Patterns (Interfaces in `src/core/interfaces/`):**
- `ICreatorService` - `create(authUser: IUserDto, dto: IBaseResourceDto): Promise<IBaseResourceDto>`
- `IByIdRetrieverService` - `findByIdOrFail(id: string): Promise<IBaseResourceDto>`
- `IUpdaterService` - `update(authUser: IUserDto, existing, updates): Promise<IBaseResourceDto>`

**Repository Patterns:**
- `ICreatorRepository` - `create(dto): Promise<DTO>`
- `IByIdRetrieverRepository` - `findByIdOrFail(id): Promise<DTO>`
- `IUpdaterRepository` - `update(id, dto): Promise<DTO>`

**Authorization Patterns:**
- **Policies:** Classes implementing `canRead()`, `canWrite()` methods checking if user can perform action
  - Location: `src/{feature}/policies/{feature}.policy.ts`
  - Example: `BusinessPolicy` checks if user owns the business
- **AccessControllerFactory:** Wraps repository methods with policy enforcement
- **AuthorizedCreatorFactory:** Specifically for create operations; ensures ownership is tracked

**Domain Objects:**
- **Entities:** MongoDB document structure (internal only, never exposed outside repository)
  - Location: `src/{feature}/entities/`
  - Pattern: `I{Feature}Entity extends { _id: ObjectId, createdAt, updatedAt, ... }`
- **DTOs:** Layer boundary contracts, used throughout service/controller layers
  - Location: `src/{feature}/data-transfer-objects/`
  - Pattern: `I{Feature}Dto` (no MongoDB fields)
- **Requests:** HTTP request body validation schemas
  - Location: `src/{feature}/requests/` or `src/{feature}/request/`
  - Pattern: Class with `@IsString()`, `@IsEmail()` etc. decorators
- **Responses:** HTTP response body structure
  - Location: `src/{feature}/responses/`
  - Pattern: `I{Feature}Response` - clean data contract

**Error Handling (in `src/core/errors/`):**
- **Custom Error Classes:**
  - `ForbiddenError` - Access denied (403)
  - `ResourceNotFoundError` - Entity not found (404)
  - `InvalidRequestError` - Bad request (422)
  - `InternalServerError` - Server error (500)
- **All inherit:** `getCode()`, `getMessage()`, `getDetails()` methods
- **Utility:** `createHttpError()` maps to HTTP exceptions with status codes
- **Pattern:** Controllers catch all errors via try-catch, pass to createHttpError()

**Standard Response Format:**
```typescript
interface IResponse<T> {
  data?: T[];           // Always array
  pagination?: {
    total: number;
    limit: number;
    offset: number;
  };
  errors?: [{ code: string, message: string, details?: any }];
}
```

### Entry Points

**`src/main.ts` - Application Bootstrap:**
- Triggers: `npm run start:dev` → Node.js process startup
- Responsibilities:
  1. Creates NestJS application: `NestFactory.create(AppModule)`
  2. Sets up Pino logger: `app.useLogger(pinoLogger)`
  3. Validates requests globally: `ValidationPipe({ whitelist: true, forbidNonWhitelisted: true })`
  4. Configures CORS: Checks `CORS_ORIGIN` env var against request origin
  5. Starts server: `app.listen(PORT)`
  6. Graceful shutdown: Listens for SIGTERM/SIGINT, closes app cleanly

**`src/app.module.ts` - Root Module:**
- Imports:
  - `ConfigModule.forRoot()` - Loads .env variables
  - `LoggerModule.forRoot()` - Configures Pino logging
  - `CoreModule` - Global services
  - Feature modules: AuthModule, UserModule, BusinessModule, CustomerModule, ItemModule, QuoteModule, TaxRateModule, JobModule, EmailModule, MigrationModule, PingModule
- Exports: Nothing (AppModule is root)

**Feature Modules (e.g., `src/business/business.module.ts`):**
- Imports: CoreModule (access to database, logging, etc.)
- Declares: Controllers, Services, Repositories
- Exports: Services that other modules need
- Pattern: NestJS module decorator `@Module()` configures DI

### Cross-Cutting Concerns

**Logging:**
- Implementation: Pino logger wrapped in `AppLogger` in `src/core/services/app-logger.service.ts`
- Usage: `private readonly logger = new AppLogger(ClassName.name)`
- Pattern: Log at key points (query execution, business logic, errors)

**Validation:**
- Global ValidationPipe applied in `main.ts`
- Decorators on request classes: `@IsString()`, `@IsEmail()`, `@IsNumber()` etc.
- Rejects invalid payloads with 400 BadRequest

**Authentication:**
- Firebase JWT verification in `JwtAuthGuard` from `src/auth/auth.guard.ts`
- Guard extracts token from `Authorization: Bearer {token}` header
- JwtStrategy validates against Firebase public keys
- Decoded JWT available as `request.user: IUserDto` in controllers
- Applied via `@UseGuards(JwtAuthGuard)` on protected routes

**Authorization:**
- Per-resource policies (e.g., BusinessPolicy in `src/business/policies/`)
- Controllers call service → Service calls AccessControllerFactory → Factory wraps repo with policy
- Policy checks: User owns business, user is admin, etc.
- Throws ForbiddenError if not allowed

---

## Frontend Architecture (React)

### Layers

**Pages (`src/pages/`):**
- **Purpose:** Route-level components representing screen destinations
- **What it does:** Composes feature components and layout, handles page-level concerns
- **Examples:** CustomersPage, JobsPage, DashboardPage, SettingsPage
- **Depends on:** Feature components, hooks, Redux selectors, React Router
- **Used by:** React Router in App.tsx
- **Pattern:** One component per route; imports from feature barrel exports

**Features (`src/features/{feature}/`):**
- **Purpose:** Self-contained domain modules (auth, customers, jobs, items, quotes, business, tax-rates, job-types)
- **Structure:**
  ```
  src/features/{feature}/
  ├── components/    # React components specific to this feature
  ├── hooks/         # Custom hooks for data/state
  ├── api/           # RTK Query endpoint definitions
  └── index.ts       # Barrel export re-exporting all
  ```
- **What it does:**
  - Components: Render UI for feature (forms, tables, dialogs, lists)
  - Hooks: Encapsulate data fetching and state logic
  - API: Define RTK Query endpoints extended from base apiSlice
- **Depends on:** Services, shared components, Redux, React
- **Used by:** Pages, other features
- **Example:** `src/features/customers/`
  - Components: CustomersDataView, CustomersTable, CustomerFormDialog, CustomersFilters
  - Hooks: useCustomersList, useCustomerActions
  - API: customerApi.ts defining endpoints

**Components (`src/components/`):**
- **Purpose:** Reusable UI building blocks
- **Subdirectories:**
  - `ui/` - shadcn/ui primitive components (Button, Card, Dialog, Table, Form, Select, etc.)
  - `layouts/` - Page layout wrappers (DashboardLayout, etc.)
  - `error-boundary/` - Error boundary components (ErrorBoundary, PageErrorBoundary)
  - `onboarding/` - Onboarding dialog components
- **Depends on:** React, Tailwind CSS, shadcn/ui
- **Used by:** Pages and features
- **Pattern:** Composable, prop-driven, no internal state (or minimal local state)

**Store (`src/store/`):**
- **Purpose:** Redux state management configuration
- **Contents:**
  - `index.ts` - Store configuration combining RTK Query apiSlice + custom reducers
  - `slices/` - Redux slices (onboarding dialog state)
  - `middleware/` - Custom middleware (onboarding middleware watches auth/business changes)
  - `hooks.ts` - Typed `useAppDispatch`, `useAppSelector`
  - `utils/` - Store utilities
- **Pattern:**
  - RTK Query (apiSlice) for async API state with automatic caching
  - Redux slices for client state (UI state like dialog visibility)
  - Custom middleware for cross-store subscriptions
- **Dependencies:** Redux Toolkit, RTK Query
- **Used by:** All components via hooks

**Services (`src/services/`):**
- **Purpose:** API integration via RTK Query
- **Contents:**
  - `api.ts` - Base RTK Query apiSlice with:
    - `baseQuery`: fetchBaseQuery with API base URL
    - Firebase auth token injection in prepareHeaders
    - Tag types for cache invalidation
  - Feature API files: `customerApi.ts`, `userApi.ts`, `migrationApi.ts`
  - Each feature file calls `apiSlice.injectEndpoints()` to add endpoints
- **Pattern:** Each endpoint defines query/mutation + cache invalidation strategy
- **Used by:** Features via generated hooks

**Providers (`src/providers/`):**
- **Purpose:** Context providers wrapping app sections
- **Contents:** AuthProvider - Manages Firebase auth state
- **Used by:** App.tsx wraps entire app

**Contexts (`src/contexts/`):**
- **Purpose:** React Context API for shared state
- **Contents:**
  - AuthContext - Current user, loading state (provided by AuthProvider)
  - OnboardingContext - Onboarding dialog state (provided by OnboardingProvider)
- **Used by:** Components via useAuth, useOnboarding hooks

**Hooks (`src/hooks/`):**
- **Purpose:** Shared custom hooks
- **Contents:** useAuth, useCurrentBusiness, useCurrency, etc.
- **Pattern:** Encapsulate data fetching and state logic
- **Used by:** Pages and features

### Data Flow

**Initial Load (Authenticated User):**
```
1. Browser loads index.html → renders src/main.tsx
2. React mounts App.tsx
3. ErrorBoundary wraps everything
4. Provider (Redux) wraps app
5. AuthProvider wraps app:
   - Firebase onAuthStateChanged() fires
   - If user logged in, sets authContext.user
6. BrowserRouter enables navigation
7. Routes evaluate; if at /dashboard:
   - ProtectedRoute checks authContext.user
   - If not authenticated, redirects to /login
   - If authenticated, renders authenticated routes + OnboardingProvider + PageErrorBoundary
8. Page component mounts (e.g., DashboardPage)
```

**Fetching Data (Get Customers):**
```
1. CustomersPage mounts
2. Calls useCustomersList() hook
3. Hook calls useGetCustomersQuery(businessId) RTK Query hook
4. RTK Query checks cache:
   - If fresh in cache, returns cached data
   - If missing/stale, executes:
5. apiSlice.baseQuery (fetchBaseQuery):
   - Gets Firebase user: auth.currentUser
   - Gets ID token: user.getIdToken()
   - Sets Authorization header: Bearer {token}
   - Fetches GET /v1/customer?businessId={id}
6. Backend processes request (see Backend Data Flow)
7. Response received: { data: [...], pagination: {...} }
8. RTK Query caches response by "Customer" tag
9. Hook returns: { data: [], isLoading: false, error: null }
10. Component renders customer list
```

**Creating Entity (Create Customer):**
```
1. User fills CustomerFormDialog
2. Form submission dispatches useCreateCustomerMutation() mutation
3. RTK Query POST /v1/customer with {name, email, address, businessId}
   - Includes Firebase auth token in header
4. Backend creates customer (see Backend Create Flow)
5. Response: { data: [{id, name, email, ...}] }
6. RTK Query:
   - Caches response
   - Invalidates "Customer" tag
   - Triggers refetch of useGetCustomersQuery() automatically
7. CustomersDataView receives updated list
8. UI re-renders with new customer
9. CustomersPage shows success toast via Sonner
```

**State Management:**
```
Redux Store Structure:
{
  [apiSlice.reducerPath]: {
    queries: {
      "getCustomers(...)": { data: [...], status: "fulfilled" },
      "getBusiness(...)": { data: {...}, status: "fulfilled" },
      ...
    },
    mutations: { ... }
  },
  onboarding: {
    isOpen: false,
    currentStep: "BUSINESS_CREATION",
    ...
  }
}

Component reads via:
- useAppSelector() for onboarding state
- useGetCustomersQuery() for API state
- useAuth() for AuthContext user
- useCurrentBusiness() custom hook for business from store
```

### Key Abstractions

**RTK Query Endpoints Pattern:**
```typescript
// In src/features/customers/api/customerApi.ts
apiSlice.injectEndpoints({
  endpoints: (builder) => ({
    getCustomers: builder.query({
      query: (businessId: string) => `/v1/customer?businessId=${businessId}`,
      transformResponse: (response: IResponse<CustomerResponse>) => response.data,
      providesTags: (result) =>
        result ? [...result.map(c => ({ type: "Customer", id: c.id })), "Customer"] : ["Customer"],
    }),
    createCustomer: builder.mutation({
      query: (data) => ({
        url: `/v1/customer`,
        method: "POST",
        body: data,
      }),
      invalidatesTags: ["Customer"],
    }),
    // Generated hooks from these endpoints:
    // useGetCustomersQuery(businessId)
    // useCreateCustomerMutation()
  }),
})
```

**Feature Organization:**
- Each feature is independent and self-contained
- Imports shared components from `src/components/`
- Imports shared hooks from `src/hooks/`
- Exports all components/hooks via barrel `index.ts`
- Pages import from feature barrel: `import { CustomerFormDialog, useCustomersList } from "@/features/customers"`

**Redux Slice Pattern (Onboarding):**
```typescript
// src/store/slices/onboardingSlice.ts
const onboardingSlice = createSlice({
  name: "onboarding",
  initialState: { isOpen: false, currentStep: null },
  reducers: {
    openOnboarding: (state) => { state.isOpen = true; },
    closeOnboarding: (state) => { state.isOpen = false; },
  },
})

// Used in components:
const isOpen = useAppSelector(state => state.onboarding.isOpen);
const dispatch = useAppDispatch();
dispatch(openOnboarding());
```

**Custom Hook Pattern:**
```typescript
// src/features/customers/hooks/useCustomersList.ts
export function useCustomersList(businessId: string) {
  const { data, isLoading, error } = useGetCustomersQuery(businessId);
  const [filtered, setFiltered] = useState(data || []);
  const [searchTerm, setSearchTerm] = useState("");

  useEffect(() => {
    setFiltered(
      data?.filter(c => c.name.includes(searchTerm)) || []
    );
  }, [data, searchTerm]);

  return { customers: filtered, isLoading, error, searchTerm, setSearchTerm };
}

// Used in page:
const { customers, isLoading } = useCustomersList(businessId);
```

### Entry Points

**`src/main.tsx` - Application Bootstrap:**
- Triggers: `npm run dev` → Vite dev server
- Creates React root and renders `<App />`

**`src/App.tsx` - Root Component:**
- Sets up provider hierarchy:
  1. ErrorBoundary (catches render errors)
  2. Provider (Redux) - access store via hooks
  3. AuthProvider (Firebase auth state)
  4. BrowserRouter (routing)
5. Defines routes:
     - `/login` - LoginPage (unauthenticated)
     - `/dashboard`, `/customers`, `/jobs`, etc. - Protected routes in AuthenticatedLayout
     - AuthenticatedLayout wraps with OnboardingProvider and PageErrorBoundary
6. Includes Toaster for notifications

### Cross-Cutting Concerns

**Logging:**
- Browser console.log/console.error
- Could be enhanced with error tracking service (Sentry, etc.)

**Validation:**
- React Hook Form + Valibot schemas in `src/lib/forms/schemas/`
- Example: `customerSchema.ts` validates customer create form
- Server errors displayed as toasts via Sonner

**Authentication:**
- Firebase Authentication (JavaScript SDK)
- AuthProvider monitors `onAuthStateChanged()`
- Tokens injected in RTK Query `prepareHeaders` callback
- ProtectedRoute guards routes - redirects to /login if not authenticated

**Authorization:**
- Backend enforces via policies (403 on unauthorized)
- Frontend shows/hides UI based on business ownership
- Example: Can't access Customers page without a business
- Backend returns error if user lacks permission

---

## Synchronization Between Frontend and Backend

**API Contract:**
- Backend response format: `src/{feature}/responses/{feature}.response.ts`
- Frontend types: `src/types/` and mapping in feature API files
- All endpoints prefixed with `/v1/` for versioning
- Example: Backend returns IBusinessResponse, RTK Query transforms to frontend shape

**Authentication Flow:**
```
1. User submits login form with email/password
2. Firebase.auth().signInWithEmailAndPassword() authenticates user
3. Firebase issues JWT token (stored in browser memory)
4. AuthProvider calls user.getIdToken() to retrieve token
5. RTK Query prepareHeaders() adds: Authorization: Bearer {token}
6. Backend JwtAuthGuard:
   - Extracts token from Authorization header
   - Validates signature against Firebase public keys
   - Decodes JWT payload → user ID
   - Injects user into request.user
7. Controllers and services access authenticated user
```

**Feature Implementation Pattern:**
```
Backend creates:
- Controller (HTTP handler)
- Services (creator/retriever/updater)
- Repositories (MongoDB access)
- Entities (MongoDB structure)
- DTOs (layer contracts)
- Responses (HTTP shape)

Frontend creates:
- Page (route component)
- Components (UI pieces)
- Hooks (data/state logic)
- API (RTK Query endpoints)
- Types (TypeScript contracts)

Flow: User → Page → Components call Hooks → Hooks call RTK Query → Backend API
```

---

*Architecture analysis: 2026-02-21*
