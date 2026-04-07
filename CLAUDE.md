<!-- GSD:project-start source:PROJECT.md -->
## Project

**Trade Flow**

A business management application for sole tradespeople -- plumbers, electricians, builders, and other independent contractors -- that replaces scattered tools (paper notes, spreadsheets, generic invoicing apps, WhatsApp, calendar apps) with one streamlined system built around how trades actually work. Everything connects to the job: quotes, schedules, materials, labour, invoices, payments, and customer history. Tradespeople can now send quotes to customers via email, and customers can accept or reject quotes directly from a secure online link.

Two independent codebases: `trade-flow-api` (NestJS/MongoDB) and `trade-flow-ui` (React/Vite), each with their own git repo, managed and deployed independently. Feature branches are created in both repos for coordinated feature work.

**Core Value:** A job is the centre of the business -- Trade Flow helps tradespeople run their entire business from first call to final payment in one simple, structured system.

### Constraints

- **Tech stack:** Must follow existing NestJS (API) and React/Vite (UI) patterns -- see `.planning/codebase/CONVENTIONS.md`
- **Two repos:** Changes span `trade-flow-api` and `trade-flow-ui` with coordinated feature branches
- **Solo operator:** Schedule assignee is always the logged-in user for now, but data model supports team assignment later
- **No standalone calendar:** Schedules only appear within a job's detail page (v1.0 scope)
<!-- GSD:project-end -->

<!-- GSD:stack-start source:codebase/STACK.md -->
## Technology Stack

## Project Overview
- **Frontend (trade-flow-ui):** React TypeScript SPA for business owners to manage their operations
- **Backend (trade-flow-api):** NestJS REST API providing data persistence and business logic
- **Database:** MongoDB for persistent data storage
- **Authentication:** Firebase Authentication for user login/signup
- **Email:** SendGrid for transactional emails
## Languages
- **TypeScript 5.9.3** - Primary language for both frontend and backend
- **HTML5** - Markup in Vite-generated `index.html` and React component rendering
- **CSS3** - Utility-first styling via Tailwind CSS 4.x
- **JavaScript** - Build configuration files (Vite, ESLint), scripts, and test runners
## Runtime & Package Management
- **Node.js 22.x** - Backend runtime (confirmed in Dockerfile: `node:22-slim` and `node:22`)
- **npm** (version 10+) - JavaScript package manager
## Frameworks & Libraries
### Frontend (trade-flow-ui)
#### Core Framework
- **React 19.2.0** - UI component framework
- **React DOM 19.2.0** - Web rendering for React
#### Build & Bundling
- **Vite 7.2.4** - Build tool and dev server
#### Routing & Navigation
- **react-router-dom 7.13.0** - Client-side routing and navigation
#### State Management
- **@reduxjs/toolkit 2.11.2** - Redux state management
- **react-redux 9.2.0** - React bindings for Redux store access
#### Form Management & Validation
- **react-hook-form 7.71.1** - Efficient form state and validation management
- **@hookform/resolvers 5.2.2** - Schema validation adapters
- **valibot 1.2.0** - Type-safe schema validation library
#### UI Components & Styling
- **Tailwind CSS 4.1.18** - Utility-first CSS framework
- **Radix UI** (15 packages, versions 1.1-2.2) - Headless accessible component primitives
- **class-variance-authority 0.7.1** - Typed CSS class composition (component variants)
- **tailwind-merge 3.4.0** - Merge Tailwind classes without conflicts
- **clsx 2.1.1** - Conditional CSS class utility
- **lucide-react 0.563.0** - SVG icon library (~560+ icons)
- **sonner 2.0.7** - Toast notification system
- **cmdk 1.1.1** - Command menu/palette component
- **tw-animate-css 1.4.0** - Tailwind CSS animation extensions
#### Financial Handling
- **dinero.js 2.0.0-alpha.14** - Immutable money/currency object library
- **@dinero.js/currencies 2.0.0-alpha.14** - Currency data for dinero.js
#### Authentication
- **firebase 12.8.0** - Firebase SDK (auth, Firestore, real-time features)
#### Testing
- **vitest 4.1.3** - Fast unit test runner (Vite-native)
- **@testing-library/react 16.3.2** - React component testing utilities
- **@testing-library/jest-dom 6.9.1** - Custom DOM matchers for assertions
- **@testing-library/user-event 14.6.1** - Simulates realistic user interactions
#### Development Tools
- **TypeScript 5.9.3** - TypeScript compiler and language
- **ESLint 9.39.1** - JavaScript/TypeScript linter
- **Type Definitions:**
- **Utilities:**
### Backend (trade-flow-api)
#### Core Framework
- **@nestjs/core 11.1.12** - NestJS framework core
- **@nestjs/common 11.1.12** - Common utilities and decorators
- **@nestjs/platform-express 11.1.12** - Express HTTP adapter
#### Configuration & Modules
- **@nestjs/config 4.0.2** - Configuration management for environment variables
#### Authentication & Security
- **@nestjs/jwt 11.0.0** - JWT handling for NestJS
- **@nestjs/passport 11.0.5** - Passport authentication integration
- **jsonwebtoken 9.0.2** - JWT encode/decode (RS256 support)
- **passport 0.7.0** - Authentication middleware framework
- **passport-jwt 4.0.1** - JWT strategy for Passport
#### Database & ORM
- **mongodb 7.0.0** - MongoDB native driver
- **mongoose 9.1.5** - MongoDB object modeling (ODM)
#### Logging & Observability
- **nestjs-pino 4.5.0** - Pino logging integration for NestJS
- **pino 10.3.0** - Fast structured JSON logger
- **pino-http 11.0.0** - HTTP request/response logging middleware
- **pino-pretty 13.1.3** (devDep) - Pretty-print log formatter for development
#### Data Validation & Transformation
- **class-validator 0.14.1** - DTO validation using decorators
- **class-transformer 0.5.1** - Object to DTO transformation
#### External Services
- **@sendgrid/mail 8.1.6** - SendGrid email service SDK
#### Date & Time
- **luxon 3.5.1** - Date/time manipulation library
#### Financial Data
- **dinero.js 1.9.1** - Money/currency objects (v1.x for API)
#### Reactive Programming
- **rxjs 7.8.2** - Reactive extensions (used internally by NestJS)
#### Language & Metadata
- **reflect-metadata 0.2.2** - Metadata reflection (required by TypeScript decorators)
#### Build & Compilation
- **@nestjs/cli 11.0.16** - NestJS CLI for scaffolding and building
- **@nestjs/schematics 11.0.5** - Code generation schematics
- **ts-jest 29.3.1** - Jest TypeScript transformer
- **ts-loader 9.5.2** - Webpack TypeScript loader
- **ts-node 10.9.2** - Execute TypeScript directly in Node
- **tsconfig-paths 4.2.0** - TypeScript path alias resolution
- **source-map-support 0.5.21** - Source map support in stack traces
#### Code Quality & Formatting
- **ESLint 9.39.2** - JavaScript/TypeScript linter (flat config)
- **Prettier 3.8.1** - Code formatter
#### Testing
- **jest 30.2.0** - Test runner with assertion library
- **@nestjs/testing 11.1.12** - NestJS testing module
- **supertest 7.2.2** - HTTP assertion library for route testing
- **Type Definitions:**
#### Git & Development Workflow
- **husky 9.1.7** - Git hooks manager
- **lint-staged 16.1.0** - Run linters on staged files
- **nodemon 3.1.9** - File watcher for development auto-restart
## Configuration
### Frontend (trade-flow-ui)
- `tsconfig.json` - Root project references (points to app and node configs)
- `tsconfig.app.json` - App compiler options
- `tsconfig.node.json` - Build tools (Vite, ESLint) TypeScript config
- `vite.config.ts` - Vite build configuration
- `eslint.config.js` - ESLint flat config (ESLint 9.x)
- `index.html` - Root HTML file
- `.env` - Local configuration (not committed, contains secrets)
- `.env.example` - Environment template:
### Backend (trade-flow-api)
- `tsconfig.json` - Main compiler options
- `tsconfig.build.json` - Build-specific config (references tsconfig.json)
- `tsconfig-check.json` - Strict type checking config (stricter than main)
- `eslint.config.mjs` - ESLint flat config (ES modules)
- `.prettierrc` - Prettier formatting rules
- `nest-cli.json` - NestJS CLI configuration
- `nodemon.json` - File watcher for development auto-restart
- `Dockerfile` - Multi-stage build
- `docker-compose.yaml` - Local development stack
- `railway.json` - Railway.app deployment config
- `.env` - Runtime configuration (not committed)
- `.env.example` - Environment template:
### Shared Configuration
- `trade-flow.code-workspace` - VS Code workspace configuration for monorepo
## Platform Requirements
- **Node.js 22.x** (confirmed in Dockerfile)
- **npm 10.x** (bundled with Node 22)
- **TypeScript 5.9.3** (installed locally per project)
- **Docker & Docker Compose** (for MongoDB and containerized development)
- **Git** (for version control and Husky hooks)
- **Node.js 22.x** for backend API
- **MongoDB 7.0+** for data persistence
- **Firebase Project** with authentication enabled
- **SendGrid Account** with API key for email delivery
- **Web Browser** with ES2020+ support for frontend
## Key Commands
### Frontend Development
### Backend Development
### Docker
## Key Technical Characteristics
- Strict TypeScript compilation across both projects
- Runtime DTO validation via class-validator in backend
- Schema validation via valibot in frontend
- Backend: Pino structured logging with HTTP middleware
- Frontend: Browser console (no explicit logger configured)
- ESLint with TypeScript support (ESLint 9.x flat config)
- Prettier code formatting (125 char line width)
- Husky pre-commit hooks with lint-staged for backend
- Backend: Jest unit tests with supertest for HTTP assertions
- Frontend: Vitest unit tests with Testing Library for component testing
- Tailwind CSS v4 (utility-first approach)
- Radix UI primitives (unstyled, accessible components)
- Custom theme with semantic color tokens
- REST API with OpenAPI/Swagger documentation (`openapi.yaml`)
- JSON request/response bodies
- Standard HTTP status codes
- CORS enabled with configurable origins
## Testing
### Backend (trade-flow-api) -- Jest
- **Runner:** Jest 30.2.0 with ts-jest transformer
- **Environment:** Node
- **Libraries:** @nestjs/testing (test module), supertest (HTTP assertions)
- **File pattern:** `*.spec.ts`
- **Location:** `src/{feature}/test/` subdirectories mirroring source structure
  - `test/controllers/` -- controller specs
  - `test/services/` -- service specs (creator, retriever, updater, deleter)
  - `test/repositories/` -- repository specs
  - `test/policies/` -- policy specs
  - `test/mocks/` -- shared mock factories
- **Path aliases:** Configured in jest moduleNameMapper (matches tsconfig paths)
- **Commands:**
  - `npm run test` -- run all tests with coverage
  - `npm run test:watch` -- watch mode
  - `npm run test:debug` -- debug mode
- **Coverage output:** `../coverage/`

### Frontend (trade-flow-ui) -- Vitest
- **Runner:** Vitest 4.1.3 (Vite-native, extends vite.config via mergeConfig)
- **Environment:** jsdom
- **Libraries:** @testing-library/react, @testing-library/jest-dom, @testing-library/user-event
- **Setup file:** `vitest.setup.ts` imports `@testing-library/jest-dom/vitest` for matchers
- **Globals:** Enabled (describe, it, expect available without imports)
- **File pattern:** `src/**/*.{test,spec}.{ts,tsx}`
- **Location:** `__tests__/` directories alongside source files
- **CSS:** Disabled in tests
- **Path aliases:** `@` alias inherited from vite config
- **Commands:**
  - `npm run test` -- run all tests (vitest run)
  - `npm run test:watch` -- watch mode
  - `npm run test:coverage` -- run with coverage
<!-- GSD:stack-end -->

<!-- GSD:conventions-start source:CONVENTIONS.md -->
## Conventions

## Backend (API) Conventions
### File Naming Patterns
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
### Class and Interface Naming
- DTO Interfaces: `I[Name]Dto` → `IBusinessDto`, `IUserDto` in `src/business/data-transfer-objects/business.dto.ts`
- Response Interfaces: `I[Name]Response` → `IPingResponse` in `src/ping/responses/ping.response.ts`
- Generic Interfaces: `I[Name]` → `IResponse`, `IJwtUser` in `src/core/response/response.interface.ts`
- Service Classes: `[Name][Action]` → `BusinessCreator`, `BusinessRetriever`, `BusinessUpdater`, `BusinessDeleter` in `src/business/services/`
- Repository Classes: `[Name]Repository` → `BusinessRepository` in `src/business/repositories/business.repository.ts`
- Controller Classes: `[Name]Controller` → `BusinessController` in `src/business/controllers/business.controller.ts`
- Module Classes: `[Name]Module` → `BusinessModule` in `src/business/business.module.ts`
- Enum Names: Singular → `BusinessStatus`, `ErrorCodes` in `src/business/enums/business-status.enum.ts`
- Keys: UPPER_SNAKE_CASE → `ACTIVE`, `INACTIVE`
- Values: lowercase strings → `"active"`, `"inactive"`
### Functions and Variables
- **Function names**: camelCase → `createPing`, `findBusinessById` in `src/ping/services/ping.service.ts`
- **Variables**: camelCase and descriptive → `businessId`, `authUser`, `createdBusiness` in controllers/services
- **Unused parameters**: Prefix with underscore → `_context`, `_options` in `src/core/services/app-logger.service.ts`
- **Private methods**: Prefix with no underscore, use verb-noun → `mapToResponse()`, `validateInput()` in controllers/services
- **Constants**: UPPER_SNAKE_CASE in files or CONSTANT_CASE for inline → `COLLECTION = "businesses"` static readonly in repository
### Import Organization
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
### Error Handling
- `InvalidRequestError` → HTTP 422 (Unprocessable Entity)
- `ResourceNotFoundError` → HTTP 404 (Not Found)
- `ForbiddenError` → HTTP 403 (Forbidden)
- `InternalServerError` → HTTP 500 (Internal Server Error)
- `getCode()` → returns `ErrorCodes` enum value
- `getMessage()` → returns human-readable message
- `getDetails()` → returns additional context
### Logging
- `log(message, context?)` → info level
- `error(message, trace?, context?)` → error level
- `warn(message, context?)` → warn level
- `debug(message, context?)` → debug level
- `verbose(message, context?)` → trace level
### Code Style
- Print Width: 125 characters
- Tab Width: 2 spaces
- Tabs: false (spaces)
- Semicolons: true
- Trailing Commas: all
- Quote Style: double quotes
- Arrow Function Parens: always
- Bracket Spacing: true
- End of Line: LF
- `@typescript-eslint/interface-name-prefix` → off (allows I prefix)
- `@typescript-eslint/explicit-function-return-type` → off
- `@typescript-eslint/explicit-module-boundary-types` → off
- `@typescript-eslint/no-explicit-any` → warn
- `@typescript-eslint/no-unused-vars` → warn with `argsIgnorePattern: "^_"`
- `prettier/prettier` → error (enforces prettier formatting)
- Target: ES2021
- Module: CommonJS
- Strict Mode: Enabled (`strictNullChecks`, `noImplicitAny`, `strictBindCallApply`)
- Emit Decorator Metadata: true (for NestJS)
- Experimental Decorators: true (for NestJS)
- Source Maps: true
- Use `const` by default
- Use `let` only when accumulating state
- Never use `var`
### Comments
## Frontend (UI) Conventions
### File Naming Patterns
- Components: PascalCase → `CustomersCardList.tsx`, `BusinessFormDialog.tsx` in `src/features/customers/components/`
- Hooks: camelCase with `use` prefix → `useAuth.ts`, `useOnboarding.ts` in `src/hooks/` or feature-specific `src/features/customers/hooks/`
- Utilities: kebab-case → `date-helpers.ts`, `format-currency.ts` in `src/lib/`
- Redux slices: camelCase with `Slice` suffix → `authSlice.ts`, `customersSlice.ts` in `src/store/slices/`
- Types/Interfaces: PascalCase → `Customer`, `AuthContextType` in `src/types/` or feature-specific
### Component Structure
- Handler callbacks: `on[Action]` → `onViewCustomer`, `onEditCustomer`, `onToggleStatus`
- Data props: descriptive names → `customers`, `selectedItem`, `isLoading`
- Boolean props: `is[State]` or `can[Action]` → `isOpen`, `isLoading`, `canDelete`
### Import Organization
### Type Declarations
- Always use `import type` for type imports → `import type { Customer } from "@/types"`
- Use interfaces for component props → `interface CustomersCardListProps { ... }`
- Use types for unions/complex types → `type OnboardingStep = "step1" | "step2"`
### State Management
- Use `useAppDispatch` and `useAppSelector` from `@/store/hooks` (not raw React Redux hooks)
- RTK Query for API data fetching
- Local React state (`useState`) for UI-only concerns
### Styling
- Color tokens: `bg-background`, `text-foreground`, `bg-primary`, `text-muted-foreground`
- Layout utilities: `flex`, `gap-`, `space-y-`, `mx-auto`
- Responsive classes: `md:` prefix for desktop (768px breakpoint)
- Mobile-first design approach
- Use `cn()` from `@/lib/utils` for conditional Tailwind classes
- Always add `min-w-0` to flex children containing truncatable content
- Example from `src/features/customers/components/CustomersCardList.tsx`:
- Use `[Feature]DataView` components that switch between Table (desktop) and Cards (mobile)
- Example: `CustomersDataView` component
### Functions and Variables
- **Function names**: camelCase → `formatCurrency`, `parseDate`, `calculateTax`
- **Variables**: camelCase and descriptive → `isLoading`, `selectedCustomer`, `filteredItems`
- **Constants**: camelCase or UPPER_SNAKE_CASE → `API_BASE_URL`, `defaultTimeout`
- **Boolean variables**: `is`, `has`, `can` prefix → `isOpen`, `hasError`, `canSubmit`
### Comments
### Validation
- Use Valibot for schema validation (from `package.json`)
- React Hook Form with Valibot resolvers for form validation (from `@hookform/resolvers`)
- Field-level validation with clear error messages
### Error Boundaries
## Shared Patterns
### Module Organization
- One feature per directory → `src/business/`, `src/customer/`, `src/quote/`
- Each module contains: controllers, services, repositories, DTOs, responses, entities
- Test files in `test/` subdirectory
- One feature per directory → `src/features/customers/`, `src/features/quotes/`
- Each feature contains: components, hooks, api (RTK Query), types
- Shared utilities in `src/lib/`, `src/hooks/`, `src/types/`
### TypeScript Configuration
- Strict null checks: enabled
- No implicit any: enabled
- Explicit function return types: required
- Path aliases configured in `tsconfig.json`
<!-- GSD:conventions-end -->

<!-- GSD:architecture-start source:ARCHITECTURE.md -->
## Architecture

## Project Overview
- **Frontend (React 19/TypeScript):** Customer-facing web application built with Vite
- **Backend (NestJS 11/TypeScript):** REST API with MongoDB persistence and Firebase authentication
- `/trade-flow-api` - Backend application
- `/trade-flow-ui` - Frontend application
## Pattern Overview
- **Backend:** Strict Controller → Service → Repository layering, never bypassed
- **Frontend:** Feature-based modules with Redux RTK Query for server state, Redux slices for client state
- **Authentication:** Firebase JWT (RS256) with server-side public key validation
- **Data Flow:** Unidirectional requests flow: UI → API → Database → Response → UI state update
- **Validation:** Request DTOs validated at controller boundary using class-validator
- **API Contract:** Standardized response format: `{ data: T[], pagination?, errors? }`
## Backend Architecture (NestJS)
### Layers
- **Purpose:** HTTP request/response handling, input validation, response mapping
- **What it does:**
- **Contains:** Controller class + request/response DTOs
- **Depends on:** Services, DTOs, Guards (JwtAuthGuard), error utilities
- **Used by:** HTTP clients
- **Pattern:** One controller per domain, multiple route handlers per controller
- **Example:** `src/business/controllers/business.controller.ts`
- **Purpose:** Business logic orchestration, authorization checks, cross-module coordination
- **What it does:**
- **Contains:** Creator/Retriever/Updater service classes
- **Depends on:** Repositories, Policies, other Services, DTOs, Factories
- **Used by:** Controllers, other Services
- **Pattern:** Single-responsibility services (BusinessCreator, BusinessRetriever, BusinessUpdater)
- **Example:** `src/business/services/business-creator.service.ts`
- **Purpose:** Data persistence abstraction, entity-DTO conversion, MongoDB operations
- **What it does:**
- **Contains:** Repository class per entity type
- **Depends on:** MongoDbFetcher, MongoDbWriter, Entities, DTOs
- **Used by:** Services only
- **Pattern:** One repository per domain model
- **Example:** `src/business/repositories/business.repository.ts`
- **Purpose:** Cross-cutting services and abstractions shared globally
- **Contents:**
- **Module:** `src/core/core.module.ts` exports globally via `@Global()` decorator
- **Used by:** All feature modules
### Data Flow
```
```
```
```
### Key Abstractions
- `ICreatorService` - `create(authUser: IUserDto, dto: IBaseResourceDto): Promise<IBaseResourceDto>`
- `IByIdRetrieverService` - `findByIdOrFail(id: string): Promise<IBaseResourceDto>`
- `IUpdaterService` - `update(authUser: IUserDto, existing, updates): Promise<IBaseResourceDto>`
- `ICreatorRepository` - `create(dto): Promise<DTO>`
- `IByIdRetrieverRepository` - `findByIdOrFail(id): Promise<DTO>`
- `IUpdaterRepository` - `update(id, dto): Promise<DTO>`
- **Policies:** Classes implementing `canRead()`, `canWrite()` methods checking if user can perform action
- **AccessControllerFactory:** Wraps repository methods with policy enforcement
- **AuthorizedCreatorFactory:** Specifically for create operations; ensures ownership is tracked
- **Entities:** MongoDB document structure (internal only, never exposed outside repository)
- **DTOs:** Layer boundary contracts, used throughout service/controller layers
- **Requests:** HTTP request body validation schemas
- **Responses:** HTTP response body structure
- **Custom Error Classes:**
- **All inherit:** `getCode()`, `getMessage()`, `getDetails()` methods
- **Utility:** `createHttpError()` maps to HTTP exceptions with status codes
- **Pattern:** Controllers catch all errors via try-catch, pass to createHttpError()
```typescript
```
### Entry Points
- Triggers: `npm run start:dev` → Node.js process startup
- Responsibilities:
- Imports:
- Exports: Nothing (AppModule is root)
- Imports: CoreModule (access to database, logging, etc.)
- Declares: Controllers, Services, Repositories
- Exports: Services that other modules need
- Pattern: NestJS module decorator `@Module()` configures DI
### Cross-Cutting Concerns
- Implementation: Pino logger wrapped in `AppLogger` in `src/core/services/app-logger.service.ts`
- Usage: `private readonly logger = new AppLogger(ClassName.name)`
- Pattern: Log at key points (query execution, business logic, errors)
- Global ValidationPipe applied in `main.ts`
- Decorators on request classes: `@IsString()`, `@IsEmail()`, `@IsNumber()` etc.
- Rejects invalid payloads with 400 BadRequest
- Firebase JWT verification in `JwtAuthGuard` from `src/auth/auth.guard.ts`
- Guard extracts token from `Authorization: Bearer {token}` header
- JwtStrategy validates against Firebase public keys
- Decoded JWT available as `request.user: IUserDto` in controllers
- Applied via `@UseGuards(JwtAuthGuard)` on protected routes
- Per-resource policies (e.g., BusinessPolicy in `src/business/policies/`)
- Controllers call service → Service calls AccessControllerFactory → Factory wraps repo with policy
- Policy checks: User owns business, user is admin, etc.
- Throws ForbiddenError if not allowed
## Frontend Architecture (React)
### Layers
- **Purpose:** Route-level components representing screen destinations
- **What it does:** Composes feature components and layout, handles page-level concerns
- **Examples:** CustomersPage, JobsPage, DashboardPage, SettingsPage
- **Depends on:** Feature components, hooks, Redux selectors, React Router
- **Used by:** React Router in App.tsx
- **Pattern:** One component per route; imports from feature barrel exports
- **Purpose:** Self-contained domain modules (auth, customers, jobs, items, quotes, business, tax-rates, job-types)
- **Structure:**
- **What it does:**
- **Depends on:** Services, shared components, Redux, React
- **Used by:** Pages, other features
- **Example:** `src/features/customers/`
- **Purpose:** Reusable UI building blocks
- **Subdirectories:**
- **Depends on:** React, Tailwind CSS, shadcn/ui
- **Used by:** Pages and features
- **Pattern:** Composable, prop-driven, no internal state (or minimal local state)
- **Purpose:** Redux state management configuration
- **Contents:**
- **Pattern:**
- **Dependencies:** Redux Toolkit, RTK Query
- **Used by:** All components via hooks
- **Purpose:** API integration via RTK Query
- **Contents:**
- **Pattern:** Each endpoint defines query/mutation + cache invalidation strategy
- **Used by:** Features via generated hooks
- **Purpose:** Context providers wrapping app sections
- **Contents:** AuthProvider - Manages Firebase auth state
- **Used by:** App.tsx wraps entire app
- **Purpose:** React Context API for shared state
- **Contents:**
- **Used by:** Components via useAuth, useOnboarding hooks
- **Purpose:** Shared custom hooks
- **Contents:** useAuth, useCurrentBusiness, useCurrency, etc.
- **Pattern:** Encapsulate data fetching and state logic
- **Used by:** Pages and features
### Data Flow
```
```
```
```
```
```
```
- useAppSelector() for onboarding state
- useGetCustomersQuery() for API state
- useAuth() for AuthContext user
- useCurrentBusiness() custom hook for business from store
```
### Key Abstractions
```typescript
```
- Each feature is independent and self-contained
- Imports shared components from `src/components/`
- Imports shared hooks from `src/hooks/`
- Exports all components/hooks via barrel `index.ts`
- Pages import from feature barrel: `import { CustomerFormDialog, useCustomersList } from "@/features/customers"`
```typescript
```
```typescript
```
### Entry Points
- Triggers: `npm run dev` → Vite dev server
- Creates React root and renders `<App />`
- Sets up provider hierarchy:
### Cross-Cutting Concerns
- Browser console.log/console.error
- Could be enhanced with error tracking service (Sentry, etc.)
- React Hook Form + Valibot schemas in `src/lib/forms/schemas/`
- Example: `customerSchema.ts` validates customer create form
- Server errors displayed as toasts via Sonner
- Firebase Authentication (JavaScript SDK)
- AuthProvider monitors `onAuthStateChanged()`
- Tokens injected in RTK Query `prepareHeaders` callback
- ProtectedRoute guards routes - redirects to /login if not authenticated
- Backend enforces via policies (403 on unauthorized)
- Frontend shows/hides UI based on business ownership
- Example: Can't access Customers page without a business
- Backend returns error if user lacks permission
## Synchronization Between Frontend and Backend
- Backend response format: `src/{feature}/responses/{feature}.response.ts`
- Frontend types: `src/types/` and mapping in feature API files
- All endpoints prefixed with `/v1/` for versioning
- Example: Backend returns IBusinessResponse, RTK Query transforms to frontend shape
```
```
```
- Controller (HTTP handler)
- Services (creator/retriever/updater)
- Repositories (MongoDB access)
- Entities (MongoDB structure)
- DTOs (layer contracts)
- Responses (HTTP shape)
- Page (route component)
- Components (UI pieces)
- Hooks (data/state logic)
- API (RTK Query endpoints)
- Types (TypeScript contracts)
```
<!-- GSD:architecture-end -->

<!-- GSD:workflow-start source:GSD defaults -->
## GSD Workflow Enforcement

Before using Edit, Write, or other file-changing tools, start work through a GSD command so planning artifacts and execution context stay in sync.

Use these entry points:
- `/gsd:quick` for small fixes, doc updates, and ad-hoc tasks
- `/gsd:debug` for investigation and bug fixing
- `/gsd:execute-phase` for planned phase work

Do not make direct repo edits outside a GSD workflow unless the user explicitly asks to bypass it.
<!-- GSD:workflow-end -->

## CI Gate Policy

All phases of development and milestones MUST have all quality checks passing before they can be deployed. This is enforced via the `npm run ci` script in both repositories.

### Quality Gates (both repos)

Every deployment must pass these checks in order:

1. **Unit tests** -- all tests must pass with zero failures
2. **Linting** -- ESLint must report zero errors
3. **Formatting** -- Prettier must report zero formatting issues
4. **Type checking** -- TypeScript compiler must report zero errors

### Commands

| Repository | CI Gate Command | Description |
|------------|----------------|-------------|
| trade-flow-api | `npm run ci` | Runs tests, lint:check, format:check, typecheck |
| trade-flow-ui | `npm run ci` | Runs tests, lint, format:check, typecheck |

### Enforcement Rules

- **Before merging any feature branch:** `npm run ci` must pass in both affected repos
- **Before deploying to Railway:** The build process should run `npm run ci` before `npm run build`
- **During phase execution:** Each plan's verification step must confirm `npm run ci` passes
- **Quick tasks:** Must not leave any repo with failing CI checks
- **No suppression:** Do not use `eslint-disable`, `@ts-ignore`, `@ts-expect-error`, or `@ts-nocheck` to bypass checks -- fix the root cause

<!-- GSD:profile-start -->
## Developer Profile

> Profile not yet configured. Run `/gsd:profile-user` to generate your developer profile.
> This section is managed by `generate-claude-profile` -- do not edit manually.
<!-- GSD:profile-end -->
