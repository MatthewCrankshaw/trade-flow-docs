# External Integrations

**Analysis Date:** 2026-02-21

## APIs & External Services

**Firebase Authentication:**
- Service: Google Firebase Authentication
- What it's used for: User authentication (sign-up, sign-in, password reset)
- SDK/Client: `firebase` npm package v12.8.0 (Frontend), Firebase keys endpoint (Backend)
- Auth: RS256 JWT tokens verified against Firebase public keys
- Key files:
  - Frontend: `src/config/firebase.ts` - Firebase app initialization
  - Frontend: `src/features/auth/components/AuthForm.tsx` - Login/signup UI
  - Frontend: `src/providers/auth-provider.tsx` - Auth state management
  - Backend: `src/auth/jwt.strategy.ts` - JWT validation strategy
  - Backend: `src/auth/services/firebase-key-provider.service.ts` - Public key fetching

**SendGrid Email Service:**
- Service: SendGrid for transactional email
- What it's used for: Sending emails to users
- SDK/Client: `@sendgrid/mail` npm package v8.1.6
- Auth: `SENDGRID_API_KEY` environment variable
- Key files:
  - Backend: `src/email/services/email-sender.service.ts` - Email sending implementation
  - Backend: `src/email/email.module.ts` - Email module configuration

**Firebase Public Keys API:**
- Service: Google's Firebase public key endpoint
- What it's used for: Fetching Firebase's RS256 public keys for JWT validation
- URL: `https://www.googleapis.com/robot/v1/metadata/x509/securetoken@system.gserviceaccount.com`
- Key files:
  - Backend: `src/auth/services/firebase-key-provider.service.ts` - Key fetching with 1-hour caching

## Data Storage

**Databases:**
- Type/Provider: MongoDB 7.0
- Connection: `MONGO_URL` environment variable (defaults to `mongodb://localhost:27017`)
- Client: `mongodb` npm package v7.0.0 (native Node.js driver)
- ORM: Custom repository pattern (no ORM used)
- Database name: `trade-flow-db`
- Key files:
  - Backend: `src/core/services/mongo/mongo-connection.service.ts` - Connection management
  - Backend: `src/core/services/mongo/mongo-db-writer.service.ts` - Write operations
  - Backend: `src/core/services/mongo/mongo-db-fetcher.service.ts` - Read operations

**Collections (inferred from repositories):**
- `ping` - Health check data
- `business` - Business entity records
- `business_users` - Business user associations
- `customers` - Customer records
- `items` - Product/service items
- `quotes` - Quote documents
- `quote_line_items` - Line items within quotes
- `users` - User profiles
- `tax_rates` - Tax configuration
- `jobs` - Job/work records
- `job_types` - Job type definitions
- `migrations` - Schema migration tracking

**File Storage:**
- Local filesystem only for now (no cloud storage integration detected)

**Caching:**
- Firebase public keys cached for 1 hour in memory
- No Redis or external caching service detected

## Authentication & Identity

**Auth Provider:**
- Service: Firebase Authentication
- Implementation: JWT with RS256 signature verification
- Token flow:
  1. Frontend: User signs in via Firebase SDK
  2. Frontend: Firebase returns ID token
  3. Frontend: Token sent in `Authorization: Bearer {token}` header
  4. Backend: JWT strategy validates token against Firebase public keys
  5. Backend: User ID extracted from token `sub` or `uid` claim
- Key files:
  - Frontend: `src/config/firebase.ts` - Firebase configuration
  - Frontend: `src/services/api.ts` - RTK Query base query that adds auth header
  - Backend: `src/auth/jwt.strategy.ts` - JWT validation and user extraction
  - Backend: `src/auth/auth.guard.ts` - Route protection decorator

**Initial Setup:**
- `INITIAL_SUPER_USER_EXTERNAL_ID` environment variable for bootstrapping first admin user

## Monitoring & Observability

**Error Tracking:**
- None detected (no Sentry, Rollbar, or similar integration)

**Logs:**
- Implementation: Pino structured logging
- Transport: Console output (formatted with pino-pretty in development)
- Key files:
  - Backend: `src/core/services/app-logger.service.ts` - Application logger wrapper
  - Backend: `src/core/config/logger.config.ts` - Logger configuration

**Metrics:**
- Firebase provides analytics (measurement ID configured but not directly used in code)
- No custom metrics or performance monitoring detected

## CI/CD & Deployment

**Hosting:**
- Not specified in codebase (assumed: self-hosted Node.js or containerized deployment)

**CI Pipeline:**
- Not detected (no GitHub Actions, GitLab CI, Jenkins configs found)

**Build Artifacts:**
- Frontend: Built by Vite to `/dist` directory
- Backend: Compiled by NestJS to `/dist` directory

## Environment Configuration

**Required env vars - Frontend:**
```
VITE_FIREBASE_API_KEY          # Firebase API key
VITE_FIREBASE_AUTH_DOMAIN      # Firebase auth domain
VITE_FIREBASE_PROJECT_ID       # Firebase project ID
VITE_FIREBASE_STORAGE_BUCKET   # Firebase storage bucket
VITE_FIREBASE_MESSAGING_SENDER_ID
VITE_FIREBASE_APP_ID
VITE_FIREBASE_MEASUREMENT_ID   # Firebase Analytics
VITE_API_BASE_URL              # Backend API URL (default: http://localhost:3000)
```

**Required env vars - Backend:**
```
CORS_ORIGIN                    # Comma-separated allowed origins
CORS_METHODS                   # HTTP methods (GET,HEAD,PUT,PATCH,POST,DELETE)
SENDGRID_API_KEY              # SendGrid API key for email
FIREBASE_PROJECT_ID           # Firebase project ID for JWT validation
FIREBASE_API_KEY              # Firebase API key (for key fetching)
INITIAL_SUPER_USER_EXTERNAL_ID # Bootstrap admin user Firebase UID
MONGO_URL                     # MongoDB connection string (default: mongodb://localhost:27017)
PORT                          # Server port (default: 3000)
NODE_ENV                      # Environment (development/production)
```

**Secrets location:**
- `.env` files in each project root (not committed to git)
- Frontend: `trade-flow-ui/.env`
- Backend: `trade-flow-api/.env`
- Example templates: `.env.example` in each project

## Webhooks & Callbacks

**Incoming:**
- None detected

**Outgoing:**
- SendGrid email events: Application sends email through SendGrid API but doesn't configure webhooks
- No other outbound webhook integrations detected

## CORS Configuration

**Allowed Origins:**
- Configured via `CORS_ORIGIN` environment variable (comma-separated list)
- Methods: `GET, HEAD, PUT, PATCH, POST, DELETE`
- Credentials: Enabled

**Key files:**
- Backend: `src/main.ts` - CORS middleware configuration

## Network Communication

**Frontend to Backend:**
- Base URL: `VITE_API_BASE_URL` (default: `http://localhost:3000`)
- Authentication: Firebase JWT in `Authorization: Bearer` header
- Content-Type: JSON (inferred from RTK Query and API patterns)

---

*Integration audit: 2026-02-21*
