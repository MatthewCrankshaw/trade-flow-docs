# Technology Stack

**Analysis Date:** 2026-02-21

## Project Overview

**Trade Flow** is a full-stack business management system designed for sole traders. The system provides customer management, quote generation and tracking, inventory/item management, tax rate handling, and job/quote status tracking. It consists of:

- **Frontend (trade-flow-ui):** React TypeScript SPA for business owners to manage their operations
- **Backend (trade-flow-api):** NestJS REST API providing data persistence and business logic
- **Database:** MongoDB for persistent data storage
- **Authentication:** Firebase Authentication for user login/signup
- **Email:** SendGrid for transactional emails

The codebase is organized as a monorepo with two independent projects (`trade-flow-ui` and `trade-flow-api`), each with its own `package.json`, TypeScript config, and development setup.

## Languages

**Primary:**
- **TypeScript 5.9.3** - Primary language for both frontend and backend
  - Strict mode enabled across both projects
  - Decorators enabled for NestJS backend
  - JSX support in frontend via React 19

**Secondary:**
- **HTML5** - Markup in Vite-generated `index.html` and React component rendering
- **CSS3** - Utility-first styling via Tailwind CSS 4.x
- **JavaScript** - Build configuration files (Vite, ESLint), scripts, and test runners

## Runtime & Package Management

**Runtime:**
- **Node.js 22.x** - Backend runtime (confirmed in Dockerfile: `node:22-slim` and `node:22`)
  - ES2021 target for compiled JavaScript
  - CommonJS modules in NestJS (compiled to dist/)

**Package Manager:**
- **npm** (version 10+) - JavaScript package manager
  - Separate `package.json` and `package-lock.json` in each project
  - Both lockfiles committed for reproducible builds

## Frameworks & Libraries

### Frontend (trade-flow-ui)

#### Core Framework
- **React 19.2.0** - UI component framework
- **React DOM 19.2.0** - Web rendering for React

#### Build & Bundling
- **Vite 7.2.4** - Build tool and dev server
  - `@vitejs/plugin-react 5.1.1` - React Fast Refresh plugin
  - `@tailwindcss/vite 4.1.18` - Tailwind CSS v4 Vite plugin
  - TypeScript support via esbuild

#### Routing & Navigation
- **react-router-dom 7.13.0** - Client-side routing and navigation

#### State Management
- **@reduxjs/toolkit 2.11.2** - Redux state management
  - Redux Thunk middleware (bundled in RTK)
  - Redux DevTools support
- **react-redux 9.2.0** - React bindings for Redux store access

#### Form Management & Validation
- **react-hook-form 7.71.1** - Efficient form state and validation management
- **@hookform/resolvers 5.2.2** - Schema validation adapters
- **valibot 1.2.0** - Type-safe schema validation library

#### UI Components & Styling
- **Tailwind CSS 4.1.18** - Utility-first CSS framework
  - Custom theme with semantic color tokens
  - Dark mode support via custom variant
  - Responsive design (mobile-first, 768px breakpoint)
- **Radix UI** (15 packages, versions 1.1-2.2) - Headless accessible component primitives
  - `@radix-ui/react-dialog 1.1.15` - Modal dialog component
  - `@radix-ui/react-alert-dialog 1.1.15` - Alert confirmation dialogs
  - `@radix-ui/react-select 2.2.6` - Custom select dropdown
  - `@radix-ui/react-dropdown-menu 2.1.16` - Dropdown menus
  - `@radix-ui/react-tabs 1.1.13` - Tab navigation
  - `@radix-ui/react-label 2.1.8` - Form labels
  - `@radix-ui/react-progress 1.1.8` - Progress bars
  - `@radix-ui/react-avatar 1.1.11` - User avatar component
  - `@radix-ui/react-separator 1.1.8` - Visual separator/divider
  - `@radix-ui/react-collapsible 1.1.12` - Collapsible sections
  - `@radix-ui/react-slot 1.2.4` - Component composition utility
  - `radix-ui 1.4.3` - Meta package
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

#### Development Tools
- **TypeScript 5.9.3** - TypeScript compiler and language
- **ESLint 9.39.1** - JavaScript/TypeScript linter
  - `@eslint/js 9.39.1` - ESLint core rules
  - `typescript-eslint 8.53.1` - TypeScript-aware linting
  - `eslint-plugin-react-hooks 7.0.1` - React Hooks rules
  - `eslint-plugin-react-refresh 0.4.24` - React Fast Refresh rules
- **Type Definitions:**
  - `@types/react 19.2.9`
  - `@types/react-dom 19.2.3`
  - `@types/node 25.0.10`
- **Utilities:**
  - `globals 17.1.0` - ESLint globals for browser environment

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
  - `@eslint/js 9.39.2`
  - `@typescript-eslint/eslint-plugin 8.53.1`
  - `@typescript-eslint/parser 8.53.1`
  - `typescript-eslint 8.53.1`
- **Prettier 3.8.1** - Code formatter
  - `eslint-plugin-prettier 5.5.5` - ESLint Prettier integration
  - `eslint-config-prettier 10.1.8` - Disable conflicting ESLint rules

#### Testing
- **jest 30.2.0** - Test runner with assertion library
- **@nestjs/testing 11.1.12** - NestJS testing module
- **supertest 7.2.2** - HTTP assertion library for route testing
- **Type Definitions:**
  - `@types/jest 30.0.0`
  - `@types/supertest 6.0.3`
  - `@types/express 5.0.1`
  - `@types/luxon 3.6.2`
  - `@types/dinero.js 1.9.4`
  - `@types/passport-jwt 4.0.1`
  - `@types/node 25.0.10`

#### Git & Development Workflow
- **husky 9.1.7** - Git hooks manager
- **lint-staged 16.1.0** - Run linters on staged files
- **nodemon 3.1.9** - File watcher for development auto-restart

## Configuration

### Frontend (trade-flow-ui)

**TypeScript Configuration:**
- `tsconfig.json` - Root project references (points to app and node configs)
- `tsconfig.app.json` - App compiler options
  - Target: `ES2022`
  - Module: `ESNext`
  - JSX: `react-jsx`
  - Path alias: `@/*` → `./src/*`
  - Strict mode: enabled (`strict: true`, `noUnusedLocals`, `noUnusedParameters`)
- `tsconfig.node.json` - Build tools (Vite, ESLint) TypeScript config

**Build Configuration:**
- `vite.config.ts` - Vite build configuration
  - Plugins: `react()`, `tailwindcss()`, `@vitejs/plugin-react`
  - Path alias resolution to `@/` for src

**Code Quality:**
- `eslint.config.js` - ESLint flat config (ESLint 9.x)
  - JS recommended rules
  - TypeScript ESLint rules
  - React Hooks rules
  - React Fast Refresh rules

**HTML Entry:**
- `index.html` - Root HTML file
  - Preconnect to Google Fonts (Inter, IBM Plex Mono)
  - Root div for React mounting

**Environment:**
- `.env` - Local configuration (not committed, contains secrets)
- `.env.example` - Environment template:
  - Firebase configuration (API key, auth domain, project ID, etc.)
  - `VITE_API_BASE_URL` - Backend API URL
  - `CORS_ORIGIN` - API CORS origin

### Backend (trade-flow-api)

**TypeScript Configuration:**
- `tsconfig.json` - Main compiler options
  - Target: `ES2021`
  - Module: `commonjs` (compiled output)
  - Decorator metadata: enabled (`emitDecoratorMetadata`, `experimentalDecorators`)
  - Strict mode: enabled
  - Path aliases (14 domain modules):
    - `@auth/*`, `@business/*`, `@business-test/*`, `@core/*`, `@core-test/*`
    - `@customer/*`, `@email/*`, `@migration/*`, `@ping/*`, `@user/*`, `@item/*`, `@quote/*`, `@tax-rate/*`, `@job/*`
- `tsconfig.build.json` - Build-specific config (references tsconfig.json)
- `tsconfig-check.json` - Strict type checking config (stricter than main)

**Code Quality:**
- `eslint.config.mjs` - ESLint flat config (ES modules)
  - ESLint recommended rules
  - TypeScript ESLint rules
  - Prettier integration
  - Ignores: `dist/`, `node_modules/`, `coverage/`, `scripts/`
- `.prettierrc` - Prettier formatting rules
  - `semi: true`, `singleQuote: false`
  - `printWidth: 125`, `tabWidth: 2`
  - `trailingComma: "all"`, `arrowParens: "always"`
  - `endOfLine: "lf"`, `bracketSpacing: true`

**NestJS Configuration:**
- `nest-cli.json` - NestJS CLI configuration
  - Collection: `@nestjs/schematics`
  - Source root: `src/`
  - Compiler options: `deleteOutDir: true`

**Development:**
- `nodemon.json` - File watcher for development auto-restart

**Docker:**
- `Dockerfile` - Multi-stage build
  - Builder: `node:22-slim` (installs build tools: python3, make, g++)
  - Development: `node:22`
- `docker-compose.yaml` - Local development stack
  - MongoDB 7.0 service with health checks
  - API service on port 3000
  - UI service commented out (can be enabled)
  - Shared network: `app-network`

**Deployment:**
- `railway.json` - Railway.app deployment config

**Environment:**
- `.env` - Runtime configuration (not committed)
- `.env.example` - Environment template:
  - `CORS_ORIGIN` - Allowed CORS origins
  - `CORS_METHODS` - Allowed HTTP methods
  - `MONGO_URL` - MongoDB connection string
  - `PORT` - Server port
  - `FIREBASE_PROJECT_ID` - Firebase project ID
  - `FIREBASE_API_KEY` - Firebase API key
  - `SENDGRID_API_KEY` - SendGrid email API key
  - `INITIAL_SUPER_USER_EXTERNAL_ID` - Superuser Firebase UID

### Shared Configuration

- `trade-flow.code-workspace` - VS Code workspace configuration for monorepo

## Platform Requirements

**Development:**
- **Node.js 22.x** (confirmed in Dockerfile)
- **npm 10.x** (bundled with Node 22)
- **TypeScript 5.9.3** (installed locally per project)
- **Docker & Docker Compose** (for MongoDB and containerized development)
- **Git** (for version control and Husky hooks)

**Production:**
- **Node.js 22.x** for backend API
- **MongoDB 7.0+** for data persistence
- **Firebase Project** with authentication enabled
- **SendGrid Account** with API key for email delivery
- **Web Browser** with ES2020+ support for frontend

## Key Commands

### Frontend Development
```bash
npm run dev              # Start Vite dev server (http://localhost:5173)
npm run build           # Production build (tsc + vite build)
npm run preview         # Preview production build locally
npm run typecheck       # TypeScript type checking (no emit)
npm run lint            # ESLint linting
```

### Backend Development
```bash
npm run start           # Start production server
npm run start:dev       # Start with watch mode and hot reload
npm run start:debug     # Start with debugger and watch mode
npm run build           # NestJS build to dist/
npm run typecheck       # TypeScript type checking
npm run typecheck:strict # Strict type checking (tsconfig-check.json)
npm run test            # Jest unit tests with coverage
npm run test:watch      # Jest watch mode
npm run lint            # ESLint with auto-fix
npm run lint:check      # ESLint without fixing
npm run validate        # Full validation (typecheck + lint)
npm run format          # Prettier code formatting
```

### Docker
```bash
docker-compose up       # Start MongoDB + API (hot reload)
docker-compose down     # Stop and remove containers
```

## Key Technical Characteristics

**Type Safety:**
- Strict TypeScript compilation across both projects
- Runtime DTO validation via class-validator in backend
- Schema validation via valibot in frontend

**Logging:**
- Backend: Pino structured logging with HTTP middleware
- Frontend: Browser console (no explicit logger configured)

**Code Quality:**
- ESLint with TypeScript support (ESLint 9.x flat config)
- Prettier code formatting (125 char line width)
- Husky pre-commit hooks with lint-staged for backend

**Testing:**
- Backend: Jest unit tests with supertest for HTTP assertions
- Frontend: Static analysis only (ESLint, TypeScript)

**Styling:**
- Tailwind CSS v4 (utility-first approach)
- Radix UI primitives (unstyled, accessible components)
- Custom theme with semantic color tokens

**API Standards:**
- REST API with OpenAPI/Swagger documentation (`openapi.yaml`)
- JSON request/response bodies
- Standard HTTP status codes
- CORS enabled with configurable origins

---

*Stack analysis: 2026-02-21*
