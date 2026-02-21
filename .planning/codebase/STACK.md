# Technology Stack

**Analysis Date:** 2026-02-21

## Overview

Trade Flow is a two-tier JavaScript/TypeScript full-stack application consisting of a React frontend and NestJS backend, both using TypeScript for type safety.

## Languages

**Primary:**
- TypeScript 5.9.3 - Both frontend and backend
- JavaScript - Configuration files and build scripts

**Secondary:**
- HTML - UI markup in React components
- CSS - Tailwind CSS for styling

## Runtime

**Environment:**
- Node.js 22.x - Backend runtime (inferred from package usage)
- Browser environments (Chrome, Firefox, Safari, Edge) - Frontend

**Package Manager:**
- npm (JavaScript package manager)
- Lockfile: `package-lock.json` present in both projects

## Frameworks

**Core Frontend:**
- React 19.2.0 - UI framework with TypeScript
- React Router DOM 7.13.0 - Client-side routing

**Core Backend:**
- NestJS 11.1.12 - Backend framework with dependency injection
- Express 5.0.1 - Underlying HTTP server (via NestJS)

**Frontend Styling:**
- Tailwind CSS 4.1.18 - Utility-first CSS framework
- shadcn/ui - React component library (Radix UI primitives)
- Radix UI (multiple components) - Accessible UI primitives
  - `@radix-ui/react-dialog`
  - `@radix-ui/react-select`
  - `@radix-ui/react-tabs`
  - `@radix-ui/react-alert-dialog`
  - `@radix-ui/react-avatar`
  - `@radix-ui/react-dropdown-menu`
  - `@radix-ui/react-label`
  - `@radix-ui/react-progress`
  - `@radix-ui/react-separator`
  - `@radix-ui/react-slot`
  - `@radix-ui/react-collapsible`

**Frontend State Management:**
- Redux Toolkit 2.11.2 - State management
- React Redux 9.2.0 - React bindings for Redux
- RTK Query - Data fetching via `createApi` and `fetchBaseQuery`

**Frontend Forms & Validation:**
- React Hook Form 7.71.1 - Form state management
- @hookform/resolvers 5.2.2 - Schema validation adapters
- Valibot 1.2.0 - Schema validation library

**Backend Authentication:**
- NestJS Passport 11.0.5 - Authentication middleware
- Passport JWT 4.0.1 - JWT strategy for Passport
- jsonwebtoken 9.0.2 - JWT creation and verification
- @nestjs/jwt 11.0.0 - JWT module for NestJS

**Backend Email:**
- @sendgrid/mail 8.1.6 - SendGrid email service client

**Testing:**
- Jest 30.2.0 - Test runner (Backend)
- ts-jest 29.3.1 - TypeScript support for Jest
- Supertest 7.2.2 - HTTP assertion library for testing
- @nestjs/testing 11.1.12 - NestJS testing utilities
- @types/jest 30.0.0 - Jest type definitions

**Development Tools:**
- Vite 7.2.4 - Frontend build tool and dev server
- @vitejs/plugin-react 5.1.1 - React plugin for Vite
- @tailwindcss/vite 4.1.18 - Tailwind CSS integration for Vite
- NestJS CLI 11.0.16 - NestJS scaffolding and compilation
- @nestjs/schematics 11.0.5 - NestJS code generation templates

**Code Quality:**
- ESLint 9.39.1+ - Linting for JavaScript/TypeScript
- @eslint/js 9.39.1+ - ESLint core rules
- typescript-eslint 8.53.1 - TypeScript support for ESLint
- eslint-plugin-react-hooks 7.0.1 - React Hooks linting rules
- eslint-plugin-react-refresh 0.4.24 - React Fast Refresh warnings
- Prettier 3.8.1 - Code formatter
- eslint-config-prettier 10.1.8 - ESLint config that disables formatting rules
- eslint-plugin-prettier 5.5.5 - Prettier integration for ESLint

**Git Hooks:**
- husky 9.1.7 - Git hook management
- lint-staged 16.1.0 - Run linters on staged files

**Logging & Monitoring:**
- Pino 10.3.0 - Structured logging for Node.js
- pino-http 11.0.0 - HTTP request logging middleware
- nestjs-pino 4.5.0 - NestJS integration for Pino
- pino-pretty 13.1.3 - Pretty-print formatter for Pino logs

**Date & Time:**
- Luxon 3.5.1 - Date and time manipulation (Backend)

**Data & Formatting:**
- Dinero.js 2.0.0-alpha.14 - Money/currency library (Frontend v2.0)
- Dinero.js 1.9.1 - Money/currency library (Backend v1.9)
- Class Validator 0.14.1 - DTO validation decorators (Backend)
- Class Transformer 0.5.1 - Object transformation (Backend)
- Zod - Schema validation (included in dependencies for validation)

**UI Utilities:**
- Lucide React 0.563.0 - Icon library
- Sonner 2.0.7 - Toast notification system
- Cmdk 1.1.1 - Command menu component
- class-variance-authority 0.7.1 - Component class variance
- clsx 2.1.1 - Conditional className utility
- tailwind-merge 3.4.0 - Merge Tailwind CSS classes

**Other:**
- RxJS 7.8.2 - Reactive programming (NestJS uses internally)
- reflect-metadata 0.2.2 - Metadata reflection (Required by NestJS/TypeScript)
- source-map-support 0.5.21 - Source map support for stack traces

## Configuration

**Frontend Configuration Files:**
- `trade-flow-ui/tsconfig.json` - TypeScript compiler options
- `trade-flow-ui/tsconfig.app.json` - App-specific TypeScript config
- `trade-flow-ui/tsconfig.node.json` - Node/Vite TypeScript config
- `trade-flow-ui/vite.config.ts` - Build and dev server configuration
- `trade-flow-ui/.eslintrc` - ESLint configuration
- `.prettierrc` (inherited from root or project-specific) - Prettier formatting rules

**Backend Configuration Files:**
- `trade-flow-api/tsconfig.json` - TypeScript compiler options
- `trade-flow-api/tsconfig.build.json` - Build-specific TypeScript config
- `trade-flow-api/tsconfig-check.json` - Strict type checking config
- `trade-flow-api/eslint.config.mjs` - ESLint configuration
- `trade-flow-api/.prettierrc` - Prettier formatting rules
- `.env` - Environment variables (sensitive, not committed)
- `.env.example` - Example environment variables template

**Environment Variables:**

Frontend (`trade-flow-ui/.env`):
```
VITE_FIREBASE_API_KEY
VITE_FIREBASE_AUTH_DOMAIN
VITE_FIREBASE_PROJECT_ID
VITE_FIREBASE_STORAGE_BUCKET
VITE_FIREBASE_MESSAGING_SENDER_ID
VITE_FIREBASE_APP_ID
VITE_FIREBASE_MEASUREMENT_ID
VITE_API_BASE_URL
```

Backend (`trade-flow-api/.env`):
```
CORS_ORIGIN
CORS_METHODS
SENDGRID_API_KEY
FIREBASE_PROJECT_ID
FIREBASE_API_KEY
INITIAL_SUPER_USER_EXTERNAL_ID
MONGO_URL
PORT
NODE_ENV
```

## Platform Requirements

**Development:**
- Node.js 22.x
- npm 9.0+
- TypeScript 5.9.3
- Git

**Production:**
- Node.js 22.x for backend
- Modern browser with ES2020+ support for frontend
- MongoDB 7.0+ database
- SendGrid account with API key
- Firebase project with authentication enabled

**Build Requirements:**
- Vite build system
- NestJS compiler

---

*Stack analysis: 2026-02-21*
