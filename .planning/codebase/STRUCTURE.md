# Codebase Structure

**Analysis Date:** 2026-02-21

## Repository Root Structure

```
/PersonalProjects/
├── trade-flow-api/              # NestJS backend (Node.js + TypeScript)
├── trade-flow-ui/               # React frontend (Vite + TypeScript)
├── .planning/                   # Planning, analysis, and orchestration files
│   └── codebase/                # Generated codebase documentation (ARCHITECTURE.md, STRUCTURE.md, etc.)
├── trade-flow.code-workspace    # VS Code workspace configuration
└── .git/                         # Version control
```

## Backend Directory Structure (`trade-flow-api/`)

### Complete Directory Tree

```
trade-flow-api/
├── src/
│   ├── main.ts                          # Entry point: Creates NestJS app, sets up middleware
│   ├── app.module.ts                    # Root module: Imports all feature modules + core
│   │
│   ├── core/                            # Global shared infrastructure (exported via @Global())
│   │   ├── core.module.ts               # Global module declaration
│   │   ├── collections/
│   │   │   ├── dto.collection.ts        # DtoCollection<T> wrapper for arrays
│   │   │   └── entity.collection.ts     # EntityCollection<T> wrapper for arrays
│   │   ├── config/
│   │   │   └── logger.config.ts         # Pino logger configuration
│   │   ├── constants/
│   │   │   ├── pagination.constants.ts
│   │   │   └── currency.constants.ts
│   │   ├── data-transfer-objects/
│   │   │   ├── base-resource.dto.ts     # Base interface for all DTOs
│   │   │   └── query-results.dto.ts     # Pagination info
│   │   ├── entities/
│   │   │   └── base.entity.ts           # Base interface (id, timestamps)
│   │   ├── errors/                      # Custom error types and utilities
│   │   │   ├── error-codes.enum.ts      # Error code enumeration
│   │   │   ├── error-code.interface.ts
│   │   │   ├── forbidden-error.error.ts
│   │   │   ├── internal-server-error.error.ts
│   │   │   ├── invalid-request.error.ts
│   │   │   ├── resource-not-found.error.ts
│   │   │   └── handle-error.utility.ts  # Maps errors to HTTP responses
│   │   ├── factories/
│   │   │   ├── access-controller.factory.ts     # Wraps services with authorization
│   │   │   └── authorized-creator.factory.ts    # Wraps create with authorization
│   │   ├── interfaces/                  # Core service/repository interfaces
│   │   │   ├── creator-service.interface.ts
│   │   │   ├── creator-repository.interface.ts
│   │   │   ├── by-id-retriever-service.interface.ts
│   │   │   ├── by-id-retriever-repository.interface.ts
│   │   │   ├── updater-service.interface.ts
│   │   │   └── updater-repository.interface.ts
│   │   ├── policies/
│   │   │   └── policy.ts                # Base policy class for authorization
│   │   ├── response/                    # Response formatting utilities
│   │   │   ├── response.interface.ts    # Standard IResponse<T> format
│   │   │   ├── response-error.interface.ts
│   │   │   └── create-response.utility.ts    # Utility to format responses
│   │   ├── services/
│   │   │   ├── app-logger.service.ts    # Pino logger wrapper
│   │   │   └── mongo/
│   │   │       ├── mongo-connection.service.ts     # MongoDB connection management
│   │   │       ├── mongo-db-fetcher.service.ts     # Query execution
│   │   │       └── mongo-db-writer.service.ts      # Insert/update/delete
│   │   ├── test/
│   │   │   └── value-objects/           # Test utilities for value objects
│   │   ├── utilities/
│   │   │   ├── merge-objects.utility.ts
│   │   │   └── pagination.utility.ts
│   │   └── value-objects/
│   │       └── money.value-object.ts    # Immutable Money domain object
│   │
│   ├── auth/                            # Authentication module (Firebase JWT)
│   │   ├── auth.module.ts
│   │   ├── auth.guard.ts                # JwtAuthGuard: Validates JWT and extracts user
│   │   ├── jwt.strategy.ts              # Passport JWT strategy for validation
│   │   ├── interfaces/
│   │   │   └── decoded-token.interface.ts
│   │   └── services/
│   │       └── firebase-key-provider.service.ts    # Gets Firebase public keys
│   │
│   ├── ping/                            # Health check endpoint (example module structure)
│   │   ├── ping.module.ts
│   │   ├── controllers/
│   │   │   └── ping.controller.ts
│   │   ├── services/
│   │   │   └── ping.service.ts
│   │   ├── repositories/
│   │   │   └── ping.repository.ts
│   │   ├── entities/
│   │   │   └── ping.entity.ts
│   │   ├── data-transfer-objects/
│   │   │   └── ping.dto.ts
│   │   └── responses/
│   │       └── ping.response.ts
│   │
│   ├── user/                            # User domain module
│   │   ├── user.module.ts               # Declares controllers, services, repositories
│   │   ├── controllers/
│   │   │   └── user.controller.ts       # GET /v1/user, PATCH /v1/user/{id}
│   │   ├── services/
│   │   │   ├── user-creator.service.ts
│   │   │   ├── user-retriever.service.ts
│   │   │   └── user-updater.service.ts
│   │   ├── repositories/
│   │   │   └── user.repository.ts
│   │   ├── entities/
│   │   │   └── user.entity.ts
│   │   ├── enums/
│   │   │   ├── user-status.enum.ts
│   │   │   └── onboarding-step.enum.ts
│   │   ├── data-transfer-objects/
│   │   │   └── user.dto.ts
│   │   ├── requests/
│   │   │   └── update-user.request.ts
│   │   ├── responses/
│   │   │   └── user.response.ts
│   │   ├── policies/
│   │   │   └── user.policy.ts
│   │   ├── utilities/
│   │   │   ├── business-role-assigner.service.ts
│   │   │   └── onboarding-progress-updater.service.ts
│   │   └── test/
│   │       ├── controllers/
│   │       ├── services/
│   │       ├── repositories/
│   │       └── mocks/
│   │
│   ├── business/                        # Business domain (sole trader's business)
│   │   ├── business.module.ts
│   │   ├── controllers/
│   │   │   └── business.controller.ts   # GET /v1/business, POST /v1/business, PATCH /v1/business/:id
│   │   ├── services/
│   │   │   ├── business-creator.service.ts
│   │   │   ├── business-retriever.service.ts
│   │   │   ├── business-updater.service.ts
│   │   │   ├── default-business-items-creator.service.ts   # Creates default items
│   │   │   ├── default-tax-rates-creator.service.ts        # Creates default tax rates
│   │   │   └── default-job-types-creator.service.ts        # Creates default job types
│   │   ├── repositories/
│   │   │   ├── business.repository.ts
│   │   │   └── business-user.repository.ts  # Links users to businesses
│   │   ├── entities/
│   │   │   └── business.entity.ts
│   │   ├── enums/
│   │   │   ├── business-status.enum.ts
│   │   │   └── primary-trade.enum.ts
│   │   ├── data-transfer-objects/
│   │   │   └── business.dto.ts
│   │   ├── requests/
│   │   │   ├── create-business.request.ts
│   │   │   └── update-business.request.ts
│   │   ├── responses/
│   │   │   └── business.response.ts
│   │   ├── policies/
│   │   │   └── business.policy.ts       # Checks: user owns this business
│   │   └── test/
│   │
│   ├── customer/                        # Customer domain
│   │   ├── customer.module.ts
│   │   ├── controller/
│   │   │   └── customer.controller.ts   # GET /v1/customer, POST /v1/customer, PATCH /v1/customer/:id
│   │   ├── services/
│   │   │   ├── customer-creator.service.ts
│   │   │   ├── customer-retriever.service.ts
│   │   │   └── customer-updater.service.ts
│   │   ├── repositories/
│   │   │   └── customer.repository.ts
│   │   ├── entities/
│   │   │   └── customer.entity.ts
│   │   ├── enums/
│   │   │   └── customer-status.enum.ts
│   │   ├── data-transfer-objects/
│   │   │   └── customer.dto.ts
│   │   ├── requests/
│   │   │   ├── create-customer.request.ts
│   │   │   └── update-customer.request.ts
│   │   ├── responses/
│   │   │   └── customer.response.ts
│   │   └── policies/
│   │       └── customer.policy.ts       # Checks: user's business owns this customer
│   │
│   ├── item/                            # Item domain (products/services inventory)
│   │   ├── item.module.ts
│   │   ├── controllers/
│   │   │   ├── item.controller.ts
│   │   │   └── mappers/
│   │   │       └── item-controller.mapper.ts
│   │   ├── services/
│   │   │   ├── item-creator.service.ts
│   │   │   ├── item-retriever.service.ts
│   │   │   └── item-updater.service.ts
│   │   ├── repositories/
│   │   │   └── item.repository.ts
│   │   ├── mappers/                     # Request/response mapping
│   │   │   └── item-repository.mapper.ts
│   │   ├── entities/
│   │   │   └── item.entity.ts
│   │   ├── enums/
│   │   │   └── item-unit.enum.ts
│   │   ├── data-transfer-objects/
│   │   │   └── item.dto.ts
│   │   ├── requests/
│   │   │   ├── create-item.request.ts
│   │   │   └── update-item.request.ts
│   │   ├── responses/
│   │   │   └── item.response.ts
│   │   ├── policies/
│   │   │   └── item.policy.ts
│   │   └── test/
│   │
│   ├── job/                             # Job domain (contracts/projects)
│   │   ├── job.module.ts
│   │   ├── controllers/
│   │   │   ├── job.controller.ts
│   │   │   └── mappers/
│   │   │       └── job-controller.mapper.ts
│   │   ├── services/
│   │   │   ├── job-creator.service.ts
│   │   │   ├── job-retriever.service.ts
│   │   │   └── job-updater.service.ts
│   │   ├── repositories/
│   │   │   └── job.repository.ts
│   │   ├── mappers/
│   │   │   └── job-repository.mapper.ts
│   │   ├── entities/
│   │   │   └── job.entity.ts
│   │   ├── enum/                        # Note: singular 'enum'
│   │   │   ├── job-status.enum.ts
│   │   │   └── job-type.enum.ts
│   │   ├── data-transfer-objects/
│   │   │   └── job.dto.ts
│   │   ├── requests/
│   │   │   ├── create-job.request.ts
│   │   │   └── update-job.request.ts
│   │   ├── responses/
│   │   │   └── job.response.ts
│   │   ├── policies/
│   │   │   └── job.policy.ts
│   │   └── test/
│   │
│   ├── quote/                           # Quote domain (job quotes/estimates)
│   │   ├── quote.module.ts
│   │   ├── controllers/
│   │   │   └── quote.controller.ts
│   │   ├── services/
│   │   │   ├── quote-creator.service.ts
│   │   │   ├── quote-retriever.service.ts
│   │   │   └── quote-updater.service.ts
│   │   ├── repositories/
│   │   │   └── quote.repository.ts
│   │   ├── entities/
│   │   │   └── quote.entity.ts
│   │   ├── enums/
│   │   │   └── quote-status.enum.ts
│   │   ├── data-transfer-objects/
│   │   │   └── quote.dto.ts
│   │   ├── requests/
│   │   │   ├── create-quote.request.ts
│   │   │   └── update-quote.request.ts
│   │   ├── responses/
│   │   │   └── quote.response.ts
│   │   ├── policies/
│   │   │   └── quote.policy.ts
│   │   └── test/
│   │
│   ├── tax-rate/                        # Tax rate domain
│   │   ├── tax-rate.module.ts
│   │   ├── controllers/
│   │   │   └── tax-rate.controller.ts
│   │   ├── services/
│   │   │   ├── tax-rate-creator.service.ts
│   │   │   ├── tax-rate-retriever.service.ts
│   │   │   └── tax-rate-updater.service.ts
│   │   ├── repositories/
│   │   │   └── tax-rate.repository.ts
│   │   ├── entities/
│   │   │   └── tax-rate.entity.ts
│   │   ├── enums/
│   │   │   └── tax-rate-type.enum.ts
│   │   ├── data-transfer-objects/
│   │   │   └── tax-rate.dto.ts
│   │   ├── requests/
│   │   │   ├── create-tax-rate.request.ts
│   │   │   └── update-tax-rate.request.ts
│   │   ├── responses/
│   │   │   └── tax-rate.response.ts
│   │   └── policies/
│   │       └── tax-rate.policy.ts
│   │
│   ├── job-type/                        # Job type reference data
│   │   ├── job.module.ts                # Note: uses 'job.module.ts'
│   │   ├── controllers/
│   │   │   └── job.controller.ts
│   │   ├── services/
│   │   │   ├── job-creator.service.ts
│   │   │   ├── job-retriever.service.ts
│   │   │   └── job-updater.service.ts
│   │   ├── repositories/
│   │   │   └── job.repository.ts
│   │   ├── entities/
│   │   │   └── job.entity.ts
│   │   ├── enums/
│   │   │   └── job-category.enum.ts
│   │   ├── data-transfer-objects/
│   │   │   └── job.dto.ts
│   │   ├── requests/
│   │   │   ├── create-job.request.ts
│   │   │   └── update-job.request.ts
│   │   ├── responses/
│   │   │   └── job.response.ts
│   │   └── policies/
│   │       └── job.policy.ts
│   │
│   ├── email/                           # Email service (SendGrid integration)
│   │   ├── email.module.ts
│   │   ├── services/
│   │   │   └── email.service.ts         # SendGrid email sender
│   │   ├── interfaces/
│   │   │   └── email-payload.interface.ts
│   │   └── requests/
│   │       └── send-email.request.ts
│   │
│   ├── migration/                       # Data migration utilities
│   │   ├── migration.module.ts
│   │   ├── controllers/
│   │   │   └── migration.controller.ts
│   │   ├── services/
│   │   │   └── migration.service.ts
│   │   ├── repositories/
│   │   │   └── migration.repository.ts
│   │   ├── entities/
│   │   │   └── migration.entity.ts
│   │   ├── enums/
│   │   │   └── migration-status.enum.ts
│   │   ├── data-transfer-objects/
│   │   │   └── migration.dto.ts
│   │   ├── requests/
│   │   │   └── execute-migration.request.ts
│   │   ├── responses/
│   │   │   └── migration.response.ts
│   │   ├── policies/
│   │   │   └── migration.policy.ts
│   │   ├── migrations/
│   │   │   └── [migration-name].migration.ts   # Actual migration scripts
│   │   ├── interfaces/
│   │   │   └── migration.interface.ts
│   │   └── test/
│   │
│   └── (continuing below)
│
├── dist/                                # Compiled output (not committed, generated by npm run build)
├── node_modules/                        # Dependencies (not committed)
├── .env                                 # Environment variables (not committed)
├── .env.example                         # Example env file (committed)
├── tsconfig.json                        # TypeScript configuration
├── tsconfig.build.json                  # TypeScript build configuration
├── tsconfig-check.json                  # TypeScript strict check
├── nest-cli.json                        # NestJS CLI configuration
├── eslint.config.js                     # ESLint configuration
├── .eslintrc.json                       # ESLint rules
├── jest.config.js                       # Jest testing configuration
├── package.json                         # Dependencies, scripts
├── package-lock.json                    # Locked dependency versions
├── CLAUDE.md                            # Instructions for Claude AI
├── README.md                            # Project documentation
├── openapi.yaml                         # API specification
└── AGENT.md                             # Detailed agent reference
```

## Frontend Directory Structure (`trade-flow-ui/`)

```
trade-flow-ui/
├── src/
│   ├── main.tsx                         # React entry point: Creates root and renders App
│   ├── App.tsx                          # Root component: Providers, routes, error boundaries
│   ├── index.css                        # Global Tailwind CSS styles
│   │
│   ├── pages/                           # Route-level page components
│   │   ├── DashboardPage.tsx            # GET / → redirects to /dashboard
│   │   ├── CustomersPage.tsx            # GET /customers
│   │   ├── JobsPage.tsx                 # GET /jobs
│   │   ├── JobDetailPage.tsx            # GET /jobs/:jobId
│   │   ├── ItemsPage.tsx                # GET /items
│   │   ├── QuotesPage.tsx               # GET /quotes
│   │   ├── BusinessPage.tsx             # GET /business
│   │   ├── SettingsPage.tsx             # GET /settings
│   │   ├── LoginPage.tsx                # GET /login
│   │   ├── HomePage.tsx
│   │   └── support/                     # Support admin pages
│   │       ├── SupportDashboardPage.tsx # GET /support
│   │       ├── SupportBusinessesPage.tsx # GET /support/businesses
│   │       └── SupportUsersPage.tsx     # GET /support/users
│   │
│   ├── features/                        # Feature modules (self-contained domain areas)
│   │   │
│   │   ├── auth/                        # Authentication feature
│   │   │   ├── components/
│   │   │   │   ├── AuthForm.tsx         # Login form component
│   │   │   │   ├── ProtectedRoute.tsx   # Route guard: redirects if not authenticated
│   │   │   │   └── index.ts             # Barrel export
│   │   │   └── index.ts                 # Public exports (ProtectedRoute, AuthForm)
│   │   │
│   │   ├── business/                    # Business management feature
│   │   │   ├── components/
│   │   │   │   ├── BusinessForm.tsx
│   │   │   │   ├── BusinessInfoCard.tsx
│   │   │   │   └── index.ts
│   │   │   ├── api/
│   │   │   │   └── businessApi.ts       # RTK Query endpoints for business
│   │   │   ├── hooks/ (if any)
│   │   │   └── index.ts
│   │   │
│   │   ├── customers/                   # Customer management feature
│   │   │   ├── components/
│   │   │   │   ├── CustomersDataView.tsx     # Main view switcher (table/card)
│   │   │   │   ├── CustomersTable.tsx        # Table view
│   │   │   │   ├── CustomersCardList.tsx     # Card view
│   │   │   │   ├── CustomersTableSkeleton.tsx # Loading skeleton
│   │   │   │   ├── CustomersCardSkeleton.tsx  # Card skeleton
│   │   │   │   ├── CustomersFilters.tsx      # Filter UI
│   │   │   │   ├── CustomerFormDialog.tsx    # Create/edit dialog
│   │   │   │   ├── CustomerDetailsDialog.tsx # View details dialog
│   │   │   │   └── index.ts
│   │   │   ├── hooks/
│   │   │   │   ├── useCustomersList.ts   # Fetches and filters customers
│   │   │   │   ├── useCustomerActions.ts # Create/update/delete mutations
│   │   │   │   └── index.ts
│   │   │   ├── api/
│   │   │   │   ├── customerApi.ts        # RTK Query endpoints
│   │   │   │   └── index.ts
│   │   │   └── index.ts                  # Barrel export (re-exports all)
│   │   │
│   │   ├── items/                       # Item/inventory management
│   │   │   ├── components/
│   │   │   │   ├── ItemsDataView.tsx
│   │   │   │   ├── ItemsTable.tsx
│   │   │   │   ├── ItemsCardList.tsx
│   │   │   │   ├── ItemsFilters.tsx
│   │   │   │   ├── ItemFormDialog.tsx
│   │   │   │   ├── forms/
│   │   │   │   │   └── shared/           # Shared form components (e.g., PriceInput)
│   │   │   │   └── index.ts
│   │   │   ├── hooks/
│   │   │   │   ├── useItemsList.ts
│   │   │   │   ├── useItemActions.ts
│   │   │   │   └── index.ts
│   │   │   ├── api/
│   │   │   │   └── itemApi.ts
│   │   │   └── index.ts
│   │   │
│   │   ├── jobs/                        # Job/contract management
│   │   │   ├── components/
│   │   │   │   ├── JobsDataView.tsx
│   │   │   │   ├── JobsTable.tsx
│   │   │   │   ├── JobsCardList.tsx
│   │   │   │   ├── JobsFilters.tsx
│   │   │   │   ├── JobFormDialog.tsx
│   │   │   │   ├── JobDetailsPanel.tsx
│   │   │   │   └── index.ts
│   │   │   ├── hooks/
│   │   │   │   ├── useJobsList.ts
│   │   │   │   ├── useJobActions.ts
│   │   │   │   └── index.ts
│   │   │   ├── api/
│   │   │   │   └── jobApi.ts
│   │   │   └── index.ts
│   │   │
│   │   ├── quotes/                      # Quote/estimate management
│   │   │   ├── components/
│   │   │   │   ├── QuotesDataView.tsx
│   │   │   │   ├── QuotesTable.tsx
│   │   │   │   ├── QuoteFormDialog.tsx
│   │   │   │   ├── QuotePreview.tsx
│   │   │   │   └── index.ts
│   │   │   ├── api/
│   │   │   │   └── quoteApi.ts
│   │   │   └── index.ts
│   │   │
│   │   ├── tax-rates/                   # Tax rate management
│   │   │   ├── components/
│   │   │   │   ├── TaxRatesTable.tsx
│   │   │   │   ├── TaxRateFormDialog.tsx
│   │   │   │   └── index.ts
│   │   │   ├── hooks/
│   │   │   │   └── useTaxRates.ts
│   │   │   ├── api/
│   │   │   │   └── taxRateApi.ts
│   │   │   └── index.ts
│   │   │
│   │   ├── job-types/                   # Job type reference data
│   │   │   ├── components/
│   │   │   │   └── JobTypeSelector.tsx
│   │   │   ├── hooks/
│   │   │   │   └── useJobTypes.ts
│   │   │   ├── api/
│   │   │   │   └── jobTypeApi.ts
│   │   │   └── index.ts
│   │   │
│   │   └── [other features follow same pattern]
│   │
│   ├── components/                      # Shared reusable components
│   │   ├── ui/                          # shadcn/ui library components
│   │   │   ├── button.tsx
│   │   │   ├── card.tsx
│   │   │   ├── checkbox.tsx
│   │   │   ├── dialog.tsx
│   │   │   ├── dropdown-menu.tsx
│   │   │   ├── form.tsx
│   │   │   ├── input.tsx
│   │   │   ├── label.tsx
│   │   │   ├── progress.tsx
│   │   │   ├── select.tsx
│   │   │   ├── separator.tsx
│   │   │   ├── sonner.tsx               # Toast notifications
│   │   │   ├── table.tsx
│   │   │   ├── tabs.tsx
│   │   │   ├── alert-dialog.tsx
│   │   │   ├── avatar.tsx
│   │   │   ├── collapsible.tsx
│   │   │   ├── command.tsx
│   │   │   └── [20+ other components]
│   │   ├── layouts/
│   │   │   ├── DashboardLayout.tsx      # Main layout with header, sidebar
│   │   │   ├── index.ts
│   │   │   └── [other layout components]
│   │   ├── error-boundary/
│   │   │   ├── ErrorBoundary.tsx        # Top-level error boundary
│   │   │   ├── PageErrorBoundary.tsx    # Per-page error boundary
│   │   │   └── [error handling components]
│   │   └── onboarding/
│   │       ├── OnboardingDialogs.tsx    # Manages which onboarding dialog to show
│   │       ├── PrerequisiteAlert.tsx    # Warning: missing prerequisite (e.g., no business)
│   │       └── [other onboarding components]
│   │
│   ├── store/                           # Redux Toolkit state management
│   │   ├── index.ts                     # Store configuration combining slices + RTK Query
│   │   ├── hooks.ts                     # Typed hooks: useAppDispatch, useAppSelector
│   │   ├── slices/
│   │   │   └── onboardingSlice.ts       # Redux slice: onboarding dialog state
│   │   ├── middleware/
│   │   │   └── onboardingMiddleware.ts  # Custom middleware: watches auth/business changes
│   │   └── utils/
│   │       └── [store utilities]
│   │
│   ├── services/                        # API integration (RTK Query)
│   │   ├── api.ts                       # Base API slice with Firebase auth token injection
│   │   ├── index.ts                     # Re-exports
│   │   ├── userApi.ts                   # User-specific API endpoints
│   │   └── migrationApi.ts              # Data migration API endpoints
│   │
│   ├── providers/
│   │   └── auth-provider.tsx            # Firebase auth state provider (wraps entire app)
│   │
│   ├── contexts/
│   │   ├── AuthContext.tsx              # Auth context provider component
│   │   ├── AuthContext.ts               # Auth context type (useAuth hook)
│   │   ├── OnboardingProvider.tsx       # Onboarding context provider
│   │   ├── OnboardingContext.ts         # Onboarding context type
│   │   └── OnboardingContext.types.ts   # Onboarding type definitions
│   │
│   ├── hooks/                           # Shared custom React hooks
│   │   ├── useAuth.ts                   # Get current user from AuthContext
│   │   ├── useCurrentBusiness.ts        # Get selected business from Redux store
│   │   ├── useCurrency.ts               # Currency formatting utilities
│   │   ├── useOnboarding.ts             # Get/set onboarding dialog state
│   │   └── [other shared hooks]
│   │
│   ├── lib/                             # Utility functions and helpers
│   │   ├── utils.ts                     # Tailwind cn() helper, other utilities
│   │   └── forms/
│   │       └── schemas/
│   │           ├── customerSchema.ts    # Zod schema for customer form validation
│   │           ├── itemSchema.ts        # Zod schema for item form validation
│   │           ├── jobSchema.ts
│   │           ├── quoteSchema.ts
│   │           └── [other form schemas]
│   │
│   ├── config/
│   │   ├── firebase.ts                  # Firebase SDK initialization
│   │   └── navigation.ts                # Navigation configuration
│   │
│   ├── types/
│   │   ├── index.ts                     # Type exports
│   │   └── api.types.ts                 # API response/request types
│   │
│   ├── assets/
│   │   └── [images, icons, etc.]
│   │
│   └── vite-env.d.ts                    # Vite environment type definitions
│
├── public/
│   ├── index.html                       # HTML entry point
│   └── [static assets]
│
├── dist/                                # Build output (not committed)
├── node_modules/                        # Dependencies (not committed)
├── .env                                 # Environment variables (not committed)
├── .env.example                         # Example env file (committed)
├── tsconfig.json                        # TypeScript configuration
├── tsconfig.app.json
├── tsconfig.node.json
├── vite.config.ts                       # Vite build configuration
├── eslint.config.js                     # ESLint configuration
├── package.json                         # Dependencies, scripts
├── package-lock.json
├── CLAUDE.md                            # Instructions for Claude AI
└── README.md                            # Project documentation
```

## Key Directory Purposes

### Backend

**`src/core/`** - Global Infrastructure
- **Purpose:** Shared services, utilities, and infrastructure used by all modules
- **What it exports:** Database access, logging, error handling, authorization, response formatting
- **Key files:** core.module.ts (declares it as @Global()), services/mongo/ (database), errors/ (custom errors)
- **Never commit:** Dynamic data from migrations

**`src/{feature}/`** - Feature Modules
- **Purpose:** Self-contained domain area with complete CRUD responsibility
- **Structure:** controllers/, services/, repositories/, entities/, DTOs, requests, responses, policies, enums, test/
- **Pattern:** Each feature independently implements the three-layer architecture
- **Examples:** customer/, job/, business/, item/, quote/, tax-rate/, job-type/, user/

**`src/{feature}/controllers/`** - HTTP Handlers
- **What lives:** Controller class with @Get/@Post/@Patch/@Delete decorators
- **Responsibility:** Validate request, call service, map response
- **File pattern:** {feature}.controller.ts

**`src/{feature}/services/`** - Business Logic
- **What lives:** Creator/Retriever/Updater service classes
- **Responsibility:** Implement business rules, call repositories, enforce authorization
- **File pattern:** {feature}-{operation}.service.ts (e.g., customer-creator.service.ts)

**`src/{feature}/repositories/`** - Database Access
- **What lives:** MongoDB queries via MongoDbFetcher/MongoDbWriter
- **Responsibility:** Entity-DTO mapping, database operations
- **File pattern:** {feature}.repository.ts

### Frontend

**`src/pages/`** - Route Pages
- **Purpose:** One component per route
- **What it does:** Composes feature components into a page layout
- **File pattern:** {PascalCase}Page.tsx (e.g., CustomersPage.tsx)

**`src/features/{feature}/`** - Feature Modules
- **Purpose:** Self-contained domain with components + hooks + API
- **Structure:** components/, hooks/, api/, index.ts (barrel export)
- **Examples:** customers/, jobs/, items/, quotes/, auth/, business/, tax-rates/, job-types/

**`src/features/{feature}/components/`** - Feature UI
- **What lives:** React components specific to this feature
- **Responsibility:** Render UI, call hooks for data
- **File pattern:** {PascalCase}[Component].tsx (e.g., CustomerFormDialog.tsx)

**`src/features/{feature}/hooks/`** - Feature Logic
- **What lives:** Custom React hooks for data fetching and state
- **Responsibility:** Call RTK Query hooks, manage local state
- **File pattern:** use{Feature}{Operation}.ts (e.g., useCustomersList.ts)

**`src/features/{feature}/api/`** - API Endpoints
- **What lives:** RTK Query endpoint definitions
- **Responsibility:** Define queries/mutations, cache invalidation strategy
- **File pattern:** {feature}Api.ts (e.g., customerApi.ts)

**`src/components/ui/`** - shadcn/ui Components
- **What lives:** UI library components (Button, Dialog, Table, Form, Select, etc.)
- **Responsibility:** Provide reusable styled UI elements
- **Generated by:** `npx shadcn-ui add {component}`
- **Committed:** Yes

**`src/store/`** - State Management
- **What lives:** Redux store configuration, slices, middleware, hooks
- **Responsibility:** Central state for RTK Query cache + Redux slices for client state
- **Files:** index.ts (store), slices/, middleware/, hooks.ts

**`src/services/`** - API Integration
- **What lives:** RTK Query base configuration + feature API definitions
- **Responsibility:** API setup, authentication token injection
- **Files:** api.ts (base configuration), {feature}Api.ts (feature endpoints)

## Naming Conventions

### Backend Files

- **Controllers:** `{feature}.controller.ts` (e.g., `customer.controller.ts`)
- **Services:** `{feature}-{operation}.service.ts` (e.g., `customer-creator.service.ts`, `customer-retriever.service.ts`, `customer-updater.service.ts`)
- **Repositories:** `{feature}.repository.ts` (e.g., `customer.repository.ts`)
- **Entities:** `{feature}.entity.ts` (e.g., `customer.entity.ts`)
- **DTOs:** `{feature}.dto.ts` (e.g., `customer.dto.ts`)
- **Requests:** `{operation}-{feature}.request.ts` (e.g., `create-customer.request.ts`)
- **Responses:** `{feature}.response.ts` (e.g., `customer.response.ts`)
- **Policies:** `{feature}.policy.ts` (e.g., `customer.policy.ts`)
- **Modules:** `{feature}.module.ts` (e.g., `customer.module.ts`)

### Backend Directories

- **Feature modules:** kebab-case (e.g., `customer`, `job-type`, `tax-rate`)
- **Internal directories:** kebab-case (e.g., `data-transfer-objects`, `value-objects`)

### Backend Classes/Interfaces

- **Classes:** PascalCase (e.g., `CustomerController`, `CustomerCreator`, `CustomerRepository`)
- **Interfaces:** `I` + PascalCase (e.g., `ICustomerDto`, `ICustomerEntity`, `IResponse<T>`)
- **Enums:** PascalCase (e.g., `CustomerStatus`, `JobType`)

### Frontend Files

- **Components:** PascalCase.tsx (e.g., `CustomerFormDialog.tsx`, `CustomersTable.tsx`)
- **Hooks:** camelCase starting with `use` (e.g., `useCustomersList.ts`, `useAuth.ts`)
- **API files:** camelCase with domain (e.g., `customerApi.ts`, `jobApi.ts`)
- **Type files:** camelCase.ts (e.g., `api.types.ts`)
- **Pages:** PascalCase + "Page.tsx" (e.g., `CustomersPage.tsx`, `DashboardPage.tsx`)

### Frontend Directories

- **Feature modules:** kebab-case (e.g., `customers`, `job-types`, `tax-rates`)
- **Internal directories:** kebab-case (e.g., `error-boundary`, `data-transfer-objects`)

### Frontend Variables/Functions

- **camelCase:** Variables, functions, hooks, const objects (e.g., `customersList`, `handleCreateCustomer`, `useCustomers`)
- **UPPER_SNAKE_CASE:** Constants (e.g., `MAX_RETRIES`, `DEFAULT_PAGE_SIZE`)
- **PascalCase:** Components, types (e.g., `CustomersPage`, `ICustomerDto`)

## Where to Add New Code

### Adding a New Backend Feature

1. **Create feature directory:** `src/{feature}/`
2. **Create subdirectories:**
   ```
   mkdir -p src/{feature}/{controllers,services,repositories,entities,data-transfer-objects,requests,responses,policies,enums,test}
   ```
3. **Create entity:** `src/{feature}/entities/{feature}.entity.ts`
   - Defines MongoDB document structure
4. **Create DTO:** `src/{feature}/data-transfer-objects/{feature}.dto.ts`
   - Interface `I{Feature}Dto` without MongoDB fields
5. **Create request:** `src/{feature}/requests/{operation}-{feature}.request.ts`
   - Class with validation decorators for HTTP body
6. **Create response:** `src/{feature}/responses/{feature}.response.ts`
   - Interface `I{Feature}Response` for HTTP response
7. **Create repository:** `src/{feature}/repositories/{feature}.repository.ts`
   - Methods: `create()`, `findByIdOrFail()`, `findByIds()`, `update()`
   - Handles entity ↔ DTO mapping
8. **Create services:** `src/{feature}/services/`
   - `{feature}-creator.service.ts` - implements `ICreatorService`
   - `{feature}-retriever.service.ts` - implements `IByIdRetrieverService`
   - `{feature}-updater.service.ts` - implements `IUpdaterService`
9. **Create controller:** `src/{feature}/controllers/{feature}.controller.ts`
   - Decorate class with `@Controller('v1')`
   - Routes: `@Get()`, `@Post()`, `@Patch()`, etc.
10. **Create policy:** `src/{feature}/policies/{feature}.policy.ts`
    - Extend `Policy` base class
    - Implement `canRead()`, `canWrite()` methods
11. **Create module:** `src/{feature}/{feature}.module.ts`
    - Import CoreModule
    - Declare controllers, services, repositories
    - Export specific services
12. **Import in AppModule:** `src/app.module.ts`
    - Add `{Feature}Module` to imports array

### Adding a New Frontend Feature

1. **Create feature directory:** `src/features/{feature}/`
2. **Create subdirectories:**
   ```
   mkdir -p src/features/{feature}/{components,hooks,api}
   ```
3. **Create components:** `src/features/{feature}/components/`
   - Each component file: `{ComponentName}.tsx` with TypeScript interfaces for props
   - Create `index.ts` to re-export all components
4. **Create hooks:** `src/features/{feature}/hooks/`
   - `use{Feature}List.ts` - fetch and filter data
   - `use{Feature}Actions.ts` - create/update/delete mutations
   - Create `index.ts` to re-export
5. **Create API:** `src/features/{feature}/api/{feature}Api.ts`
   ```typescript
   export const {feature}Api = apiSlice.injectEndpoints({
     endpoints: (builder) => ({
       get{Feature}s: builder.query({
         query: (businessId: string) => `/v1/{feature}?businessId=${businessId}`,
         transformResponse: (response: IResponse<{Feature}Response>) => response.data,
         providesTags: (result) => result ? [...result.map(i => ({ type: "{Feature}", id: i.id })), "{Feature}"] : ["{Feature}"],
       }),
       create{Feature}: builder.mutation({
         query: (data) => ({ url: `/v1/{feature}`, method: "POST", body: data }),
         invalidatesTags: ["{Feature}"],
       }),
     }),
   })
   ```
6. **Create barrel export:** `src/features/{feature}/index.ts`
   ```typescript
   export * from "./components"
   export * from "./hooks"
   export * from "./api"
   ```
7. **Reference in page:** `src/pages/{Feature}Page.tsx`
   - Import components/hooks from feature
   - Compose into page layout

### Adding a New Component

**Backend:**
- Location: `src/{feature}/controllers/{component}.controller.ts`
- Create controller class with routes

**Frontend - Feature Specific:**
- Location: `src/features/{feature}/components/{Component}.tsx`
- Create React component with typed props interface
- Add to feature's `index.ts` barrel export

**Frontend - Shared:**
- Location: `src/components/` (either in `ui/`, `layouts/`, `error-boundary/`, or top-level)
- Create React component
- Use in multiple features/pages

### Adding Tests

**Backend:**
- Location: `src/{feature}/test/{what}.spec.ts`
- Structure: Mirror the domain (controllers/, services/, repositories/ subdirectories)
- Pattern: Use Jest with mocks from `mocks/` subdirectory

**Frontend:**
- Location: Co-locate with component as `{Component}.test.tsx` OR in `__tests__/` directory
- Pattern: Use Vitest with React Testing Library

### Adding Utilities

**Backend:**
- Global: `src/core/utilities/{utility-name}.utility.ts`
- Feature-specific: `src/{feature}/utilities/{utility-name}.utility.ts`

**Frontend:**
- Global: `src/lib/{utility-name}.ts`
- Feature-specific: `src/features/{feature}/{utility-name}.ts` (if not exported from index.ts)

## Special Directories

### Backend

**`src/core/` (Global Module):**
- **Generated:** No
- **Committed:** Yes
- **Purpose:** Shared infrastructure

**`src/{feature}/test/`:**
- **Generated:** No
- **Committed:** Yes
- **Purpose:** Unit tests with mocks

**`dist/` (Build Output):**
- **Generated:** Yes (by `npm run build`)
- **Committed:** No (in .gitignore)

**`node_modules/`:**
- **Generated:** Yes (by `npm install`)
- **Committed:** No (in .gitignore)

### Frontend

**`src/components/ui/` (shadcn/ui):**
- **Generated:** Partially (via `npx shadcn-ui add {component}`)
- **Committed:** Yes
- **Purpose:** UI component library

**`dist/` (Build Output):**
- **Generated:** Yes (by `npm run build`)
- **Committed:** No (in .gitignore)

**`node_modules/`:**
- **Generated:** Yes (by `npm install`)
- **Committed:** No (in .gitignore)

**`public/` (Static Assets):**
- **Generated:** No (hand-managed)
- **Committed:** Yes
- **Purpose:** Static files served directly

---

*Structure analysis: 2026-02-21*
