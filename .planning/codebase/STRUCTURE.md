# Codebase Structure

**Analysis Date:** 2026-02-21

## Repository Root

```
/
├── trade-flow-api/              # NestJS backend (Node.js + TypeScript)
├── trade-flow-ui/               # React frontend (Vite + TypeScript)
├── .planning/                   # Planning and documentation
├── trade-flow.code-workspace    # VS Code workspace configuration
└── .git/                         # Version control
```

## Backend Directory Layout (trade-flow-api)

```
trade-flow-api/
├── src/
│   ├── main.ts                          # Application entry point
│   ├── app.module.ts                    # Root NestJS module
│   │
│   ├── core/                            # Global shared infrastructure (exported globally)
│   │   ├── collections/                 # DtoCollection, EntityCollection wrappers
│   │   ├── config/                      # Logger configuration
│   │   ├── constants/                   # Pagination, currency constants
│   │   ├── data-transfer-objects/       # Base DTOs (pagination, query options)
│   │   ├── entities/                    # Base entity interface
│   │   ├── errors/                      # Custom error classes, error mapping
│   │   ├── factories/                   # AccessControllerFactory, AuthorizedCreatorFactory
│   │   ├── interfaces/                  # Core interfaces (repository, service contracts)
│   │   ├── policies/                    # Base policy class for access control
│   │   ├── response/                    # Response formatting utilities
│   │   ├── services/                    # MongoDbFetcher, MongoDbWriter, Logging, Access control
│   │   ├── test/                        # Core tests (value objects)
│   │   ├── utilities/                   # Pagination parsing, object merging, etc.
│   │   └── core.module.ts               # Exports all above globally
│   │
│   ├── auth/                            # Authentication (Firebase JWT strategy)
│   │   ├── auth.guard.ts                # JwtAuthGuard for route protection
│   │   ├── interfaces/                  # Auth-related interfaces
│   │   ├── services/                    # JWT strategy service
│   │   └── auth.module.ts
│   │
│   ├── ping/                            # Health check endpoint (example module)
│   │   ├── controllers/
│   │   ├── services/
│   │   ├── repositories/
│   │   ├── entities/
│   │   ├── data-transfer-objects/
│   │   ├── responses/
│   │   └── ping.module.ts
│   │
│   ├── user/                            # User domain
│   │   ├── controllers/
│   │   ├── services/
│   │   │   ├── user-creator.service.ts
│   │   │   ├── user-retriever.service.ts
│   │   │   └── user-updater.service.ts
│   │   ├── repositories/
│   │   ├── entities/
│   │   ├── enums/
│   │   ├── data-transfer-objects/
│   │   ├── requests/
│   │   ├── responses/
│   │   ├── policies/
│   │   ├── utilities/
│   │   ├── test/                        # Unit tests (controllers, services, repositories, mocks)
│   │   └── user.module.ts
│   │
│   ├── business/                        # Business domain (sole trader's business)
│   │   ├── controllers/
│   │   ├── services/
│   │   ├── repositories/
│   │   ├── entities/
│   │   ├── enums/
│   │   ├── data-transfer-objects/
│   │   ├── requests/
│   │   ├── responses/
│   │   ├── policies/
│   │   ├── test/
│   │   └── business.module.ts
│   │
│   ├── customer/                        # Customer domain
│   │   ├── controller/
│   │   │   └── customer.controller.ts
│   │   ├── services/
│   │   │   ├── customer-creator.service.ts
│   │   │   ├── customer-retriever.service.ts
│   │   │   └── customer-updater.service.ts
│   │   ├── repositories/
│   │   ├── entities/
│   │   ├── enums/
│   │   ├── data-transfer-objects/
│   │   ├── requests/
│   │   ├── responses/
│   │   ├── policies/
│   │   └── customer.module.ts
│   │
│   ├── item/                            # Item domain (products/services)
│   │   ├── controllers/
│   │   │   └── mappers/                 # Request/response mappers
│   │   ├── services/
│   │   ├── repositories/
│   │   ├── entities/
│   │   ├── enums/
│   │   ├── data-transfer-objects/
│   │   ├── requests/
│   │   ├── responses/
│   │   ├── policies/
│   │   ├── test/
│   │   └── item.module.ts
│   │
│   ├── job/                             # Job domain (contracts/projects)
│   │   ├── controllers/
│   │   │   └── mappers/
│   │   ├── services/
│   │   ├── repositories/
│   │   ├── entities/
│   │   ├── enum/                        # Note: singular 'enum'
│   │   ├── data-transfer-objects/
│   │   ├── requests/
│   │   ├── responses/
│   │   ├── policies/
│   │   ├── test/
│   │   └── job.module.ts
│   │
│   ├── quote/                           # Quote domain (job quotes)
│   │   ├── controllers/
│   │   ├── services/
│   │   ├── repositories/
│   │   ├── entities/
│   │   ├── enums/
│   │   ├── data-transfer-objects/
│   │   ├── requests/
│   │   ├── responses/
│   │   ├── policies/
│   │   ├── test/
│   │   └── quote.module.ts
│   │
│   ├── tax-rate/                        # Tax rate domain
│   │   ├── controllers/
│   │   ├── services/
│   │   ├── repositories/
│   │   ├── entities/
│   │   ├── enums/
│   │   ├── data-transfer-objects/
│   │   ├── requests/
│   │   ├── responses/
│   │   ├── policies/
│   │   └── tax-rate.module.ts
│   │
│   ├── email/                           # Email service (notifications)
│   │   ├── services/
│   │   ├── interfaces/
│   │   ├── requests/
│   │   └── email.module.ts
│   │
│   ├── migration/                       # Data migration utilities
│   │   ├── controllers/
│   │   ├── services/
│   │   ├── repositories/
│   │   ├── entities/
│   │   ├── enums/
│   │   ├── data-transfer-objects/
│   │   ├── requests/
│   │   ├── responses/
│   │   ├── policies/
│   │   ├── migrations/                  # Migration scripts
│   │   ├── interfaces/
│   │   └── migration.module.ts
│   │
│   └── job-type/                        # Job type reference data
│       ├── controllers/
│       ├── services/
│       ├── repositories/
│       ├── entities/
│       ├── enums/
│       ├── data-transfer-objects/
│       ├── requests/
│       ├── responses/
│       └── job-type.module.ts
│
├── tsconfig.json                        # TypeScript configuration
├── tsconfig.build.json
├── nest-cli.json                        # NestJS CLI configuration
├── package.json                         # Dependencies, scripts
├── package-lock.json
├── eslint.config.js                     # ESLint configuration
├── .eslintrc.json
├── jest.config.js                       # Jest testing configuration
├── CLAUDE.md                            # Instructions for Claude AI
├── .env                                 # Environment variables (not committed)
└── dist/                                # Compiled output (not committed)
```

## Frontend Directory Layout (trade-flow-ui)

```
trade-flow-ui/
├── src/
│   ├── main.tsx                         # React application entry point
│   ├── App.tsx                          # Root application component + routes
│   ├── index.css                        # Global styles
│   │
│   ├── pages/                           # Route pages (one per route)
│   │   ├── DashboardPage.tsx
│   │   ├── CustomersPage.tsx
│   │   ├── JobsPage.tsx
│   │   ├── JobDetailPage.tsx
│   │   ├── ItemsPage.tsx
│   │   ├── QuotesPage.tsx
│   │   ├── BusinessPage.tsx
│   │   ├── SettingsPage.tsx
│   │   ├── LoginPage.tsx
│   │   ├── support/
│   │   │   ├── SupportDashboardPage.tsx
│   │   │   ├── SupportBusinessesPage.tsx
│   │   │   └── SupportUsersPage.tsx
│   │   └── ...
│   │
│   ├── features/                        # Feature modules (domain areas)
│   │   ├── auth/                        # Authentication
│   │   │   ├── components/
│   │   │   │   ├── AuthForm.tsx
│   │   │   │   ├── ProtectedRoute.tsx   # Auth guard wrapper
│   │   │   │   └── index.ts
│   │   │   └── index.ts                 # Barrel export
│   │   │
│   │   ├── customers/
│   │   │   ├── components/
│   │   │   │   ├── CustomersDataView.tsx
│   │   │   │   ├── CustomersTable.tsx
│   │   │   │   ├── CustomersCardList.tsx
│   │   │   │   ├── CustomersTableSkeleton.tsx
│   │   │   │   ├── CustomersCardSkeleton.tsx
│   │   │   │   ├── CustomersFilters.tsx
│   │   │   │   ├── CustomerFormDialog.tsx
│   │   │   │   ├── CustomerDetailsDialog.tsx
│   │   │   │   └── index.tsx            # Re-exports all components
│   │   │   ├── hooks/
│   │   │   │   ├── useCustomersList.ts
│   │   │   │   ├── useCustomerActions.ts
│   │   │   │   └── index.ts
│   │   │   ├── api/
│   │   │   │   ├── customerApi.ts       # RTK Query endpoints
│   │   │   │   └── index.ts
│   │   │   └── index.ts                 # Barrel export
│   │   │
│   │   ├── jobs/
│   │   │   ├── components/
│   │   │   ├── hooks/
│   │   │   ├── api/
│   │   │   └── index.ts
│   │   │
│   │   ├── items/
│   │   │   ├── components/
│   │   │   │   └── forms/
│   │   │   │       └── shared/          # Shared form components
│   │   │   ├── hooks/
│   │   │   ├── api/
│   │   │   └── index.ts
│   │   │
│   │   ├── quotes/
│   │   │   ├── components/
│   │   │   ├── api/
│   │   │   └── index.ts
│   │   │
│   │   ├── business/
│   │   │   ├── components/
│   │   │   ├── api/
│   │   │   └── index.ts
│   │   │
│   │   ├── tax-rates/
│   │   │   ├── components/
│   │   │   ├── api/
│   │   │   └── index.ts
│   │   │
│   │   ├── job-types/
│   │   │   ├── components/
│   │   │   ├── api/
│   │   │   └── index.ts
│   │   │
│   │   └── ...
│   │
│   ├── components/                      # Shared components
│   │   ├── ui/                          # shadcn/ui library components
│   │   │   ├── button.tsx
│   │   │   ├── card.tsx
│   │   │   ├── dialog.tsx
│   │   │   ├── table.tsx
│   │   │   ├── tabs.tsx
│   │   │   ├── toast.tsx
│   │   │   └── ... (20+ shadcn components)
│   │   ├── layouts/
│   │   │   ├── DashboardLayout.tsx      # Main page layout wrapper
│   │   │   └── index.ts
│   │   ├── error-boundary/
│   │   │   ├── ErrorBoundary.tsx        # Top-level error boundary
│   │   │   ├── PageErrorBoundary.tsx    # Per-page error boundary
│   │   │   └── ...
│   │   ├── onboarding/
│   │   │   ├── OnboardingDialogs.tsx    # Onboarding dialog manager
│   │   │   ├── PrerequisiteAlert.tsx    # Missing prerequisite warnings
│   │   │   └── ...
│   │   └── sonner/                      # Toast notifications
│   │
│   ├── store/                           # Redux Toolkit state management
│   │   ├── index.ts                     # Store configuration
│   │   ├── hooks.ts                     # useAppDispatch, useAppSelector
│   │   ├── slices/
│   │   │   └── onboardingSlice.ts       # Onboarding dialog visibility state
│   │   ├── middleware/
│   │   │   └── onboardingMiddleware.ts  # Listens to auth/business changes
│   │   └── utils/
│   │
│   ├── services/                        # API service layer
│   │   ├── api.ts                       # RTK Query base configuration + setup
│   │   ├── index.ts                     # Re-exports
│   │   ├── userApi.ts                   # User-specific API endpoints
│   │   └── migrationApi.ts              # Migration API endpoints
│   │
│   ├── providers/
│   │   └── auth-provider.tsx            # Firebase auth state provider
│   │
│   ├── contexts/                        # React Context
│   │   ├── AuthContext.tsx              # Auth state context (user, loading)
│   │   ├── AuthContext.ts               # Auth context type definition
│   │   ├── OnboardingContext.tsx        # Onboarding state provider
│   │   ├── OnboardingContext.ts         # Onboarding context definition
│   │   └── OnboardingContext.types.ts   # Onboarding type definitions
│   │
│   ├── hooks/                           # Shared custom hooks
│   │   ├── useAuth.ts                   # Get auth context
│   │   ├── useCurrentBusiness.ts        # Get current business from store
│   │   ├── useCurrency.ts               # Currency formatting utilities
│   │   └── ...
│   │
│   ├── lib/                             # Utility functions
│   │   ├── utils.ts                     # cn() for tailwind class merging
│   │   └── forms/
│   │       └── schemas/                 # Zod form validation schemas
│   │           ├── customerSchema.ts
│   │           ├── itemSchema.ts
│   │           └── ...
│   │
│   ├── config/                          # Configuration files
│   │   ├── firebase.ts                  # Firebase SDK initialization
│   │   └── navigation.ts                # Navigation configuration
│   │
│   ├── types/                           # Shared TypeScript types
│   │   ├── index.ts                     # Type exports
│   │   └── api.types.ts                 # API response types
│   │
│   ├── assets/                          # Static assets
│   │   └── ...
│   │
│   └── vite-env.d.ts                    # Vite environment type definitions
│
├── public/                              # Static files (index.html, favicon, etc.)
├── tsconfig.json                        # TypeScript configuration
├── tsconfig.app.json
├── tsconfig.node.json
├── vite.config.ts                       # Vite build configuration
├── eslint.config.js                     # ESLint configuration
├── package.json                         # Dependencies, scripts
├── package-lock.json
├── index.html                           # HTML entry point
├── CLAUDE.md                            # Instructions for Claude AI
├── .env                                 # Environment variables (not committed)
├── .env.example                         # Example environment variables
└── dist/                                # Built output (not committed)
```

## Directory Purposes - Backend

### Core Module (`src/core/`)
- **Purpose:** Global shared infrastructure exported via `@Global()` decorator
- **What lives here:** Database access, logging, error handling, access control, response formatting
- **Key files:**
  - `core.module.ts` - Declares all providers and exports globally
  - `services/mongo/` - MongoDB connection and query services
  - `errors/` - Custom error classes and utilities
  - `factories/` - Dependency injection factories

### Feature Modules (e.g., `src/customer/`, `src/job/`)
- **Purpose:** Self-contained domain areas with clear layering
- **What lives here:**
  - Controllers for HTTP handling
  - Services for business logic (creator/retriever/updater pattern)
  - Repositories for database access
  - Entities and DTOs for type safety
  - Policies for access control
  - Tests for unit testing
- **Pattern:** Imports CoreModule, declares own components, exports specific services

### Test Directories (`src/{feature}/test/`)
- **Purpose:** Unit tests for the feature
- **Contains:** Test files with .spec.ts extension, mock factories
- **Pattern:** Mirror structure of domain (controllers, services, repositories directories)

## Directory Purposes - Frontend

### Pages (`src/pages/`)
- **Purpose:** Route-level components (one per route)
- **What lives here:** High-level page layout and composition
- **Key files:** One .tsx file per route (CustomersPage, JobsPage, etc.)

### Features (`src/features/{feature}/`)
- **Purpose:** Self-contained domain areas
- **What lives here:**
  - `components/` - React UI components specific to this feature
  - `hooks/` - Custom hooks for data fetching and state
  - `api/` - RTK Query endpoint definitions
- **Pattern:** Exported via barrel `index.ts` file for easy importing

### Components (`src/components/`)
- **Purpose:** Shared UI components used across features
- **What lives here:**
  - `ui/` - shadcn/ui library components
  - `layouts/` - Page layout wrappers
  - `error-boundary/` - Error boundary components
  - `onboarding/` - Onboarding-specific components

### Services (`src/services/`)
- **Purpose:** API and external service integration
- **What lives here:**
  - `api.ts` - RTK Query base configuration
  - Feature-specific API files (userApi.ts, migrationApi.ts)
- **Pattern:** RTK Query endpoints that generate hooks

### Store (`src/store/`)
- **Purpose:** Redux Toolkit state management
- **What lives here:**
  - `index.ts` - Store configuration
  - `slices/` - Redux slices (onboarding, etc.)
  - `middleware/` - Custom middleware
  - `hooks.ts` - Typed dispatch/selector hooks

## Key File Locations

### Backend Entry Points
- **Application bootstrap:** `src/main.ts`
- **Module configuration:** `src/app.module.ts`
- **Feature bootstrap:** `src/{feature}/{feature}.module.ts`

### Backend Configuration
- **Database:** `src/core/services/mongo/mongo-connection.service.ts`
- **Logging:** `src/core/config/logger.config.ts`
- **CORS/Security:** `src/main.ts` (middleware setup)

### Backend Core Logic
- **Base interfaces:** `src/core/interfaces/`
- **Error handling:** `src/core/errors/` + `src/core/response/`
- **Database access:** `src/core/services/mongo/`

### Frontend Entry Points
- **Application bootstrap:** `src/main.tsx`
- **Root component:** `src/App.tsx`
- **Pages:** `src/pages/*.tsx`

### Frontend Configuration
- **Firebase setup:** `src/config/firebase.ts`
- **API base URL:** `src/services/api.ts`
- **Redux store:** `src/store/index.ts`

### Frontend Core Logic
- **Authentication:** `src/providers/auth-provider.tsx`, `src/contexts/AuthContext.tsx`
- **API integration:** `src/services/api.ts` + feature `api/` files
- **State management:** `src/store/`

## Naming Conventions

### Backend

**Files:**
- Controllers: `{feature}.controller.ts` (e.g., `customer.controller.ts`)
- Services: `{feature}-{operation}.service.ts` (e.g., `customer-creator.service.ts`, `customer-retriever.service.ts`)
- Repositories: `{feature}.repository.ts` (e.g., `customer.repository.ts`)
- Entities: `{feature}.entity.ts` (e.g., `customer.entity.ts`)
- DTOs: `{feature}.dto.ts` (e.g., `customer.dto.ts`)
- Modules: `{feature}.module.ts` (e.g., `customer.module.ts`)

**Directories:**
- Feature modules: kebab-case (e.g., `customer`, `job-type`, `tax-rate`)
- Internal directories: kebab-case (e.g., `data-transfer-objects`, `value-objects`)

**Classes/Interfaces:**
- Classes: PascalCase (e.g., `CustomerController`, `CustomerCreator`)
- Interfaces: `I` + PascalCase (e.g., `ICustomerDto`, `ICustomerEntity`)
- Enums: PascalCase (e.g., `CustomerStatus`)

### Frontend

**Files:**
- Components: PascalCase.tsx (e.g., `CustomersDataView.tsx`)
- Hooks: camelCase starting with `use` (e.g., `useCustomersList.ts`)
- Services: camelCase with domain (e.g., `customerApi.ts`)
- Types: camelCase.ts (e.g., `api.types.ts`)

**Directories:**
- Feature modules: kebab-case (e.g., `customers`, `job-types`)
- Internal directories: kebab-case (e.g., `data-transfer-objects`, `form-schemas`)

**Variables/Functions:**
- camelCase (e.g., `const customersList = ...`, `function handleCreateCustomer() {}`)
- Constants: UPPER_SNAKE_CASE (e.g., `const MAX_RETRIES = 3`)

## Where to Add New Code

### New Backend Feature
1. **Create module directory:** `src/{feature}/`
2. **Create subdirectories:**
   - `controllers/` - HTTP endpoints
   - `services/` - Business logic (creator, retriever, updater services)
   - `repositories/` - Database access
   - `entities/` - MongoDB document structure
   - `data-transfer-objects/` - DTOs
   - `requests/` - Request validation objects
   - `responses/` - Response format objects
   - `policies/` - Access control
   - `enums/` - Feature-specific enums
   - `test/` - Unit tests
3. **Create module file:** `src/{feature}/{feature}.module.ts`
   - Import CoreModule
   - Declare controllers, services, repositories
   - Export specific services
4. **Import in AppModule:** `src/app.module.ts`

### New Frontend Feature
1. **Create feature directory:** `src/features/{feature}/`
2. **Create subdirectories:**
   - `components/` - React components
   - `hooks/` - Custom hooks
   - `api/` - RTK Query endpoints
3. **Create barrel export:** `src/features/{feature}/index.ts`
4. **Reference in pages:** Import components and hooks from feature barrel
5. **Add API endpoints:** Create `src/features/{feature}/api/{feature}Api.ts`
   - Use `apiSlice.injectEndpoints()`
   - Export generated hooks

### New Component (Backend)
- Location: `src/{feature}/components/` (if feature-specific)
- Create: Controller class with `@Controller()` and route decorators

### New Component (Frontend)
- Location:
  - Feature-specific: `src/features/{feature}/components/`
  - Shared: `src/components/`
- Create: React component file (.tsx) with TypeScript interfaces for props

### New Utility Function
- **Backend:** `src/core/utilities/` (global) or `src/{feature}/utilities/` (feature-specific)
- **Frontend:** `src/lib/` (global) or `src/{feature}/` (feature-specific)

### New Test
- **Backend:** `src/{feature}/test/{what}.spec.ts`
- **Frontend:** Co-locate with component as `{component}.test.tsx` or in dedicated `__tests__/` directory

## Special Directories

### Backend Specifics

**`src/core/` (Global Module):**
- Generated: No (hand-written)
- Committed: Yes
- Purpose: Shared infrastructure available to all modules

**`src/{feature}/test/`:**
- Generated: No (hand-written)
- Committed: Yes
- Purpose: Unit tests with mocks and fixtures

**`dist/` (Build Output):**
- Generated: Yes (by `npm run build`)
- Committed: No (in `.gitignore`)

### Frontend Specifics

**`src/components/ui/` (shadcn/ui Library):**
- Generated: Partially (via `npx shadcn-ui add {component}`)
- Committed: Yes
- Purpose: UI component library components

**`dist/` (Build Output):**
- Generated: Yes (by `npm run build`)
- Committed: No (in `.gitignore`)

**`node_modules/`:**
- Generated: Yes (by `npm install`)
- Committed: No (in `.gitignore`)

---

*Structure analysis: 2026-02-21*
