# Phase 16: Token Infrastructure and Public API - Research

**Researched:** 2026-03-15
**Domain:** Cryptographic token generation, public API endpoints, NestJS rate limiting
**Confidence:** HIGH

## Summary

This phase creates the backend infrastructure for customers to view quotes without logging in. The system needs three core pieces: (1) a `quote_tokens` MongoDB collection with a repository/service layer for generating and validating cryptographically secure tokens, (2) a public API endpoint (`GET /v1/public/quote/:token`) that bypasses the existing `JwtAuthGuard` and returns a customer-safe subset of quote data, and (3) rate limiting on the public endpoint to prevent abuse.

The existing codebase provides strong patterns to follow. The Controller -> Service -> Repository layering, `MongoDbFetcher`/`MongoDbWriter` for database access, `createResponse()` for standard responses, and `createHttpError()` for error mapping all apply directly. The main novelty is the first unauthenticated endpoint -- every existing route uses `@UseGuards(JwtAuthGuard)`. The public controller simply omits this decorator.

**Primary recommendation:** Create a new `QuoteTokenModule` (separate from `QuoteModule`) with its own repository, services, entity, DTO, and a `PublicQuoteController`. Use Node.js built-in `crypto.randomBytes(32)` for token generation and `@nestjs/throttler` v6.5.0 for rate limiting the public endpoint only.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- Crypto-random string (32-byte hex or base64url), 30-day expiry from generation
- Stored in a separate `quote_tokens` collection (not on the quote document)
- Token used as path segment in URL: `/quote/view/{token}` (frontend route, Phase 17) and `/v1/public/quote/:token` (API endpoint)
- Quote-specific tokens -- no scoping or resource-type abstraction
- Token record includes: token, quoteId, expiresAt, revokedAt, sentAt, recipientEmail
- sentAt + recipientEmail link each token to a specific send event (for attribution)
- View tracking (viewedAt, viewCount) deferred to Phase 17
- No limit on active tokens per quote -- each re-send creates a new token, all valid until expiry or revocation
- Path: `GET /v1/public/quote/:token` -- new `public` namespace for unauthenticated endpoints
- No `@UseGuards(JwtAuthGuard)` -- first unguarded endpoint in the application
- Public response shape: includes business name, customer name, job title, quote number, quote title, quote date, validUntil, current status, full line items (quantity, unit, unitPrice, lineTotal, taxRate, type, bundle components), totals (subtotal, tax, total)
- Hidden from public response: all internal IDs (businessId, customerId, jobId, itemId), status timestamps (sentAt, acceptedAt, rejectedAt, deletedAt, updatedAt), notes field, original prices / discount amounts (originalUnitPrice, originalLineTotal, discountAmount)
- Basic rate limiting on public endpoint (~60 requests/minute per IP)
- Expired token: HTTP 410 Gone with message "This quote link has expired. Please contact the tradesperson for an updated link."
- Deleted quote: HTTP 410 Gone with message "This quote is no longer available."
- Invalid/unknown token: HTTP 404
- Log failed token lookups via Pino logger -- no database audit trail
- Tokens generated on send (Phase 18 calls the token service) -- no token for draft quotes
- Token generation service is internal only (no HTTP endpoint to create tokens)
- On quote deletion: revoke all tokens for that quote (set revokedAt)
- New token generated on each re-send -- old tokens remain valid until expiry or revocation

### Claude's Discretion
- Exact token length and encoding (hex vs base64url)
- Rate limiting implementation approach (NestJS throttler, custom guard, etc.)
- Token service class structure and naming
- Public controller placement (new module vs within quote module)
- Index strategy on quote_tokens collection

### Deferred Ideas (OUT OF SCOPE)
- General-purpose scoped token system (resourceType + scope fields) -- future if tokens needed for invoices or other resources
- Customer-facing notes field (separate from internal notes) -- future consideration
- Token-based view tracking (viewedAt, viewCount on token record) -- Phase 17
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| RESP-01 (partial -- backend only) | Customer can view the full quote online via a secure link without logging in | Token generation service, quote_tokens collection, public API endpoint returning customer-safe quote data, rate limiting, error handling for expired/invalid tokens |
</phase_requirements>

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| Node.js `crypto` | built-in | Token generation via `randomBytes(32)` | No external dependency needed; cryptographically secure; built into Node.js |
| `@nestjs/throttler` | ^6.5.0 | Rate limiting on public endpoint | Official NestJS rate limiting module; v6.5.0 supports NestJS 11; well-maintained |
| `mongodb` | ^7.0.0 (existing) | quote_tokens collection access via MongoDbFetcher/MongoDbWriter | Already in use; no new database driver needed |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `luxon` | ^3.5.1 (existing) | DateTime handling for expiresAt calculations | Already used for all date handling in the project |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| `crypto.randomBytes` | `uuid` v4 | randomBytes gives more control over length/encoding; uuid is 128-bit which is sufficient but less flexible |
| `@nestjs/throttler` | Custom IP-tracking guard | Throttler is 3 lines of config vs building your own; handles edge cases (proxies, storage) |

**Installation:**
```bash
cd trade-flow-api && npm install @nestjs/throttler@^6.5.0
```

## Architecture Patterns

### Recommended Project Structure
```
src/
├── quote-token/                          # New module (separate from quote/)
│   ├── controllers/
│   │   └── public-quote.controller.ts    # GET /v1/public/quote/:token (no auth guard)
│   ├── services/
│   │   ├── quote-token-creator.service.ts    # Generates tokens (internal, no HTTP endpoint)
│   │   ├── quote-token-retriever.service.ts  # Validates and retrieves tokens
│   │   └── quote-token-revoker.service.ts    # Revokes tokens on quote deletion
│   ├── repositories/
│   │   └── quote-token.repository.ts     # CRUD for quote_tokens collection
│   ├── data-transfer-objects/
│   │   └── quote-token.dto.ts            # IQuoteTokenDto interface
│   ├── entities/
│   │   └── quote-token.entity.ts         # IQuoteTokenEntity (MongoDB shape)
│   ├── responses/
│   │   └── public-quote.response.ts      # IPublicQuoteResponse (customer-safe subset)
│   ├── enums/
│   │   └── quote-token-error-codes.enum.ts  # (or add to existing ErrorCodes)
│   └── quote-token.module.ts             # Module declaration
```

**Path alias to add:** `@quote-token/*` -> `src/quote-token/*` in tsconfig.json and jest config.

### Pattern 1: Token Generation (Internal Service Only)
**What:** Service that generates cryptographically secure tokens and stores them in quote_tokens collection.
**When to use:** Called by Phase 18's send flow (not exposed via HTTP).
**Example:**
```typescript
// Source: Node.js crypto docs + existing codebase patterns
import { randomBytes } from "crypto";
import { Injectable } from "@nestjs/common";
import { DateTime } from "luxon";
import { ObjectId } from "mongodb";

@Injectable()
export class QuoteTokenCreator {
  private static readonly TOKEN_BYTES = 32;
  private static readonly EXPIRY_DAYS = 30;

  constructor(private readonly quoteTokenRepository: QuoteTokenRepository) {}

  public async create(quoteId: string, recipientEmail: string): Promise<IQuoteTokenDto> {
    const token = randomBytes(QuoteTokenCreator.TOKEN_BYTES).toString("base64url");
    const now = DateTime.now();
    const dto: IQuoteTokenDto = {
      id: new ObjectId().toString(),
      token,
      quoteId,
      expiresAt: now.plus({ days: QuoteTokenCreator.EXPIRY_DAYS }),
      sentAt: now,
      recipientEmail,
    };
    return this.quoteTokenRepository.create(dto);
  }
}
```

### Pattern 2: Public Controller (No Auth Guard)
**What:** Controller at `/v1/public` namespace that does NOT use `@UseGuards(JwtAuthGuard)`.
**When to use:** For all unauthenticated customer-facing endpoints.
**Example:**
```typescript
// Source: Existing QuoteController pattern, minus auth
import { Controller, Get, Param, HttpException, HttpStatus } from "@nestjs/common";
import { Throttle } from "@nestjs/throttler";

@Controller("v1/public")
export class PublicQuoteController {
  constructor(
    private readonly quoteTokenRetriever: QuoteTokenRetriever,
    // ... repositories for enrichment
  ) {}

  @Throttle({ default: { limit: 60, ttl: 60000 } })
  @Get("quote/:token")
  public async findByToken(@Param("token") token: string): Promise<IResponse<IPublicQuoteResponse>> {
    // 1. Look up token in quote_tokens collection
    // 2. Check expiry/revocation -> 410 Gone
    // 3. Check quote status (DELETED) -> 410 Gone
    // 4. Fetch quote + enrich (business name, customer name, job title)
    // 5. Map to IPublicQuoteResponse (customer-safe fields only)
    // 6. Return createResponse([response])
  }
}
```

### Pattern 3: Token Revocation on Quote Deletion
**What:** When a quote transitions to DELETED, revoke all its tokens by setting `revokedAt`.
**When to use:** Integrated into the existing `QuoteTransitionService` or called alongside it.
**Example:**
```typescript
// In QuoteTransitionService.transition() or as a listener
if (targetStatus === QuoteStatus.DELETED) {
  updated.deletedAt = DateTime.now();
  await this.quoteTokenRevoker.revokeAllForQuote(existing.id);
}
```

### Pattern 4: Public Response Mapping (Customer-Safe Fields)
**What:** A dedicated response interface and mapper that strips all internal data.
**When to use:** In the public controller to ensure no internal IDs or sensitive data leak.
**Example:**
```typescript
// IPublicQuoteResponse -- customer-safe subset
export interface IPublicQuoteLineItemResponse {
  quantity: number;
  unit: string;
  unitPrice: number;       // final price only (major units)
  lineTotal: number;       // final total only (major units)
  taxRate: number;
  type: string;            // material, labour, fee
  components?: IPublicQuoteLineItemResponse[];
}

export interface IPublicQuoteResponse {
  businessName: string;
  customerName: string;
  jobTitle: string;
  quoteNumber: string;
  quoteTitle: string;
  quoteDate: string;
  validUntil?: string;
  status: string;
  lineItems: IPublicQuoteLineItemResponse[];
  totals: {
    subTotal: number;
    taxTotal: number;
    total: number;
  };
}
```

### Anti-Patterns to Avoid
- **Reusing IQuoteResponse for public endpoint:** The existing `IQuoteResponse` contains all internal IDs. Create a separate `IPublicQuoteResponse` to make it impossible to accidentally leak data.
- **Adding token fields to the quotes collection:** CONTEXT.md explicitly says tokens go in a separate `quote_tokens` collection. Keeps concerns separated and allows multiple tokens per quote.
- **Using the existing QuoteRetriever for public access:** `QuoteRetriever` requires `authUser` for policy checks. The public controller should use its own repository-level access, bypassing authorization (the token IS the authorization).
- **Exposing a token creation HTTP endpoint:** Token generation is internal only. Phase 18 will call the service directly. No `POST /v1/public/quote-token` endpoint.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Token randomness | Custom random string generator | `crypto.randomBytes(32).toString("base64url")` | Cryptographically secure; handles entropy pool correctly; base64url is URL-safe |
| Rate limiting | Custom IP tracking middleware/guard | `@nestjs/throttler` with `@Throttle()` decorator | Handles proxy headers, storage, TTL windows, per-route config out of the box |
| HTTP 410 Gone response | Custom exception class | `new HttpException(response, HttpStatus.GONE)` | NestJS has `HttpStatus.GONE` (410) built in; use standard response format |

**Key insight:** The token generation and rate limiting are solved problems. The real work is in the data mapping (ensuring the public response excludes all internal fields) and the integration points (token revocation on deletion, enrichment without auth).

## Common Pitfalls

### Pitfall 1: Leaking Internal IDs in Public Response
**What goes wrong:** Using the same response mapper as the authenticated endpoint, accidentally exposing businessId, customerId, jobId, itemId.
**Why it happens:** Developer reuses existing `mapToResponse()` method which includes all fields.
**How to avoid:** Create a completely separate `IPublicQuoteResponse` interface and `mapToPublicResponse()` method. The public response interface should not extend or share fields with `IQuoteResponse`.
**Warning signs:** Public response interface importing from or extending `IQuoteResponse`.

### Pitfall 2: Enrichment Requires Auth User
**What goes wrong:** The existing `enrichAndMapToResponse()` in `QuoteController` calls `CustomerRetriever.findByIdOrFail(authUser, ...)` and `JobRetriever.findByIdOrFail(authUser, ...)` which require an authenticated user for policy checks.
**Why it happens:** All existing services enforce authorization.
**How to avoid:** The public controller should call the repositories directly (not the auth-wrapped services) for enrichment: `BusinessRepository.findByIdOrFail(businessId)`, `CustomerRepository.findByIdOrFail(customerId)`, `JobRepository` -- or create dedicated unauthenticated retriever methods. Since the token already authorizes access, repository-level access is appropriate.
**Warning signs:** Importing `QuoteRetriever`, `CustomerRetriever`, `JobRetriever` into the public module (these all require auth).

### Pitfall 3: Forgetting to Check Quote Status After Token Validation
**What goes wrong:** Token is valid but the quote has been deleted. Without checking quote status, deleted quote data is served.
**Why it happens:** Token validation and quote validation are separate concerns.
**How to avoid:** After finding a valid (non-expired, non-revoked) token, fetch the quote and check `status !== QuoteStatus.DELETED` before returning data.
**Warning signs:** No quote status check in the public endpoint handler.

### Pitfall 4: Missing MongoDB Index on Token Field
**What goes wrong:** Every public request does a full collection scan on `quote_tokens` to find the token string.
**Why it happens:** Forgetting to create an index since the collection is new.
**How to avoid:** Create a unique index on the `token` field and a compound index on `quoteId` for revocation queries.
**Warning signs:** Slow public endpoint responses as token count grows.

### Pitfall 5: ThrottlerGuard Applied Globally Breaking Authenticated Routes
**What goes wrong:** If ThrottlerModule is configured with a global guard, it applies to all routes including authenticated ones, potentially rate-limiting legitimate users.
**Why it happens:** `APP_GUARD` provider applies to every route.
**How to avoid:** Do NOT use `APP_GUARD` for `ThrottlerGuard`. Instead, apply `@UseGuards(ThrottlerGuard)` only on the public controller, or use `@SkipThrottle()` on authenticated controllers. Applying per-controller is cleaner.
**Warning signs:** Authenticated API calls returning 429 Too Many Requests.

### Pitfall 6: Token Comparison Timing Attacks
**What goes wrong:** Comparing tokens with `===` leaks information about partial matches via timing differences.
**Why it happens:** String comparison short-circuits on first mismatched character.
**How to avoid:** Not a real concern here because the token is looked up by value in MongoDB (database query timing dominates), not compared in application code. The lookup is `{ token: providedToken }` -- MongoDB handles the comparison. No timing attack vector.
**Warning signs:** N/A -- not applicable for database lookups.

## Code Examples

### Token Entity (MongoDB Document Shape)
```typescript
// Source: Existing entity patterns (IQuoteEntity, IBaseEntity)
import { IBaseEntity } from "@core/entities/base.entity";
import { ObjectId } from "mongodb";

export interface IQuoteTokenEntity extends IBaseEntity {
  token: string;              // base64url, 32 bytes
  quoteId: ObjectId;          // reference to quotes collection
  expiresAt: Date;            // 30 days from creation
  revokedAt?: Date;           // set on revocation
  sentAt: Date;               // when the email was sent
  recipientEmail: string;     // who received the link
}
```

### Token DTO
```typescript
// Source: Existing DTO patterns (IQuoteDto, IBaseResourceDto)
import { IBaseResourceDto } from "@core/data-transfer-objects/base-resource.dto";
import { DateTime } from "luxon";

export interface IQuoteTokenDto extends IBaseResourceDto {
  id: string;
  token: string;
  quoteId: string;
  expiresAt: DateTime;
  revokedAt?: DateTime;
  sentAt: DateTime;
  recipientEmail: string;
}
```

### Token Repository
```typescript
// Source: Existing repository patterns (QuoteRepository, BusinessRepository)
import { Injectable } from "@nestjs/common";
import { ObjectId } from "mongodb";
import { DateTime } from "luxon";
import { MongoDbFetcher } from "@core/services/mongo/mongo-db-fetcher.service";
import { MongoDbWriter } from "@core/services/mongo/mongo-db-writer.service";
import { AppLogger } from "@core/services/app-logger.service";
import { createAuditFields } from "@core/utilities/create-audit-fields.utility";

@Injectable()
export class QuoteTokenRepository {
  private static readonly COLLECTION = "quote_tokens";
  private readonly logger = new AppLogger(QuoteTokenRepository.name);

  constructor(
    private readonly fetcher: MongoDbFetcher,
    private readonly writer: MongoDbWriter,
  ) {}

  public async create(dto: IQuoteTokenDto): Promise<IQuoteTokenDto> {
    const entity: IQuoteTokenEntity = {
      _id: new ObjectId(),
      ...createAuditFields(),
      token: dto.token,
      quoteId: new ObjectId(dto.quoteId),
      expiresAt: dto.expiresAt.toJSDate(),
      sentAt: dto.sentAt.toJSDate(),
      recipientEmail: dto.recipientEmail,
    };
    await this.writer.insertOne<IQuoteTokenEntity>(QuoteTokenRepository.COLLECTION, entity);
    return this.toDto(entity);
  }

  public async findByToken(token: string): Promise<IQuoteTokenDto | null> {
    const entity = await this.fetcher.findOne<IQuoteTokenEntity>(
      QuoteTokenRepository.COLLECTION,
      { token },
    );
    if (!entity) return null;
    return this.toDto(entity);
  }

  public async revokeAllForQuote(quoteId: string): Promise<void> {
    await this.writer.updateMany<IQuoteTokenEntity>(
      QuoteTokenRepository.COLLECTION,
      { quoteId: new ObjectId(quoteId), revokedAt: { $exists: false } },
      { $set: { revokedAt: new Date() } },
    );
  }

  private toDto(entity: IQuoteTokenEntity): IQuoteTokenDto {
    return {
      id: entity._id.toString(),
      token: entity.token,
      quoteId: entity.quoteId.toString(),
      expiresAt: DateTime.fromJSDate(entity.expiresAt),
      revokedAt: entity.revokedAt ? DateTime.fromJSDate(entity.revokedAt) : undefined,
      sentAt: DateTime.fromJSDate(entity.sentAt),
      recipientEmail: entity.recipientEmail,
    };
  }
}
```

### ThrottlerModule Configuration
```typescript
// In QuoteTokenModule or AppModule
import { ThrottlerModule } from "@nestjs/throttler";

// Option A: Import in QuoteTokenModule only
@Module({
  imports: [
    ThrottlerModule.forRoot([{
      name: "public",
      ttl: 60000,   // 1 minute
      limit: 60,    // 60 requests per minute per IP
    }]),
    CoreModule,
    QuoteModule,  // to access QuoteRepository
    BusinessModule,
    CustomerModule,
    JobModule,
  ],
  controllers: [PublicQuoteController],
  providers: [QuoteTokenRepository, QuoteTokenCreator, QuoteTokenRetriever, QuoteTokenRevoker],
  exports: [QuoteTokenCreator, QuoteTokenRevoker],  // Phase 18 needs these
})
export class QuoteTokenModule {}
```

### Public Controller Error Handling
```typescript
// Custom error responses for the public endpoint
// Uses standard createResponse/createErrorResponse + HttpException

// Expired token -> 410 Gone
throw new HttpException(
  createErrorResponse([{
    code: "TOKEN_EXPIRED",
    message: "This quote link has expired. Please contact the tradesperson for an updated link.",
  }]),
  HttpStatus.GONE,
);

// Deleted quote -> 410 Gone
throw new HttpException(
  createErrorResponse([{
    code: "QUOTE_UNAVAILABLE",
    message: "This quote is no longer available.",
  }]),
  HttpStatus.GONE,
);

// Invalid/unknown token -> 404
throw new HttpException(
  createErrorResponse([{
    code: ErrorCodes.RESOURCE_NOT_FOUND,
    message: "Quote not found.",
  }]),
  HttpStatus.NOT_FOUND,
);
```

### MongoDB Index Strategy
```typescript
// To be created via migration or ensureIndex in repository
// Index 1: Unique index on token (for lookups)
db.collection("quote_tokens").createIndex({ token: 1 }, { unique: true });

// Index 2: Compound index for revocation queries
db.collection("quote_tokens").createIndex({ quoteId: 1, revokedAt: 1 });

// Index 3: TTL index for automatic cleanup of expired tokens (optional, nice-to-have)
// MongoDB TTL indexes automatically delete documents after a period
db.collection("quote_tokens").createIndex({ expiresAt: 1 }, { expireAfterSeconds: 0 });
```

## Discretion Recommendations

### Token encoding: base64url (recommended over hex)
- **base64url** produces 43 characters for 32 bytes (`randomBytes(32).toString("base64url")`)
- **hex** produces 64 characters for 32 bytes (`randomBytes(32).toString("hex")`)
- base64url is URL-safe by definition (no `+`, `/`, or `=`), shorter URLs, same security
- **Recommendation:** Use `base64url`

### Rate limiting: @nestjs/throttler (recommended)
- Official NestJS module, v6.5.0 supports NestJS 11
- Per-route `@Throttle()` decorator -- apply only to public controller
- In-memory store is fine for single-instance deployment (no Redis needed)
- **Recommendation:** Install `@nestjs/throttler@^6.5.0`, configure per-controller

### Token service structure: Three focused services
- `QuoteTokenCreator` -- generates and stores tokens (exported for Phase 18)
- `QuoteTokenRetriever` -- finds token, checks validity (used by public controller)
- `QuoteTokenRevoker` -- revokes all tokens for a quote (exported for deletion flow)
- **Recommendation:** Follows existing single-responsibility pattern (QuoteCreator, QuoteRetriever, etc.)

### Module placement: Separate `quote-token` module (recommended)
- Keeps public endpoint separate from authenticated quote endpoints
- Clean module boundary for imports/exports
- Phase 18 imports `QuoteTokenModule` to use `QuoteTokenCreator`
- The module needs to import `QuoteModule`, `BusinessModule`, `CustomerModule`, `JobModule` for enrichment (or just their repositories directly via `CoreModule`)
- **Recommendation:** New `src/quote-token/` directory with `QuoteTokenModule`

### Index strategy
- **Unique index on `token`:** Essential for fast lookups and uniqueness guarantee
- **Compound index on `{ quoteId: 1, revokedAt: 1 }`:** For efficient revocation queries
- **Optional TTL index on `expiresAt`:** MongoDB auto-deletes expired documents; reduces collection size over time without manual cleanup
- **Recommendation:** All three indexes. Create via the existing migration pattern (`scripts/create-migration.js`)

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| JWT tokens for stateless access links | Opaque random tokens with DB lookup | N/A -- project decision | Simpler, revocable, no secret key management for access links |
| `@nestjs/throttler` v4-5 | `@nestjs/throttler` v6.5.0 | 2025 | v6.5.0 added NestJS 11 support; throttlers array config |

## Open Questions

1. **Cross-module repository access for enrichment**
   - What we know: The public endpoint needs business name, customer name, and job title. These live in `BusinessRepository`, `CustomerRepository`, and `JobRepository` (or their services).
   - What's unclear: Whether to import the full modules (`BusinessModule`, `CustomerModule`, `JobModule`) or just expose repositories. Current `QuoteModule` imports `CustomerModule` and `JobModule` for the same enrichment pattern.
   - Recommendation: Import the existing modules and use their repositories directly (bypassing the auth-wrapped services). This follows how `QuoteModule` already imports `CustomerModule` and `JobModule`.

2. **Migration for indexes**
   - What we know: The project has a `scripts/create-migration.js` for creating migrations.
   - What's unclear: Whether indexes should be created via migration or via `ensureIndex` in repository constructor.
   - Recommendation: Use migration for production safety. Create indexes as part of the first plan.

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Jest 30.2.0 with ts-jest |
| Config file | package.json `jest` section |
| Quick run command | `npm run test -- --testPathPattern=quote-token` |
| Full suite command | `npm run test` |

### Phase Requirements -> Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| RESP-01-a | Token generation produces valid base64url string of correct length | unit | `npm run test -- --testPathPattern=quote-token-creator` | No -- Wave 0 |
| RESP-01-b | Token retrieval returns null for unknown token | unit | `npm run test -- --testPathPattern=quote-token-retriever` | No -- Wave 0 |
| RESP-01-c | Expired token is rejected (returns appropriate error) | unit | `npm run test -- --testPathPattern=quote-token-retriever` | No -- Wave 0 |
| RESP-01-d | Revoked token is rejected | unit | `npm run test -- --testPathPattern=quote-token-retriever` | No -- Wave 0 |
| RESP-01-e | Public response contains only customer-safe fields | unit | `npm run test -- --testPathPattern=public-quote` | No -- Wave 0 |
| RESP-01-f | Revoke all tokens for a quote works correctly | unit | `npm run test -- --testPathPattern=quote-token-revoker` | No -- Wave 0 |
| RESP-01-g | Quote deletion triggers token revocation | unit | `npm run test -- --testPathPattern=quote-transition` | No -- Wave 0 |

### Sampling Rate
- **Per task commit:** `npm run test -- --testPathPattern=quote-token`
- **Per wave merge:** `npm run test`
- **Phase gate:** Full suite green before `/gsd:verify-work`

### Wave 0 Gaps
- [ ] `src/quote-token/test/services/quote-token-creator.service.spec.ts` -- covers token generation
- [ ] `src/quote-token/test/services/quote-token-retriever.service.spec.ts` -- covers token validation (expired, revoked, valid)
- [ ] `src/quote-token/test/services/quote-token-revoker.service.spec.ts` -- covers bulk revocation
- [ ] `src/quote-token/test/controllers/public-quote.controller.spec.ts` -- covers endpoint responses and field filtering
- [ ] `src/quote-token/test/mocks/` -- shared mock factories for token DTOs

## Sources

### Primary (HIGH confidence)
- Existing codebase: `trade-flow-api/src/quote/` -- all patterns for controller, service, repository, entity, DTO, response
- Existing codebase: `trade-flow-api/src/core/` -- MongoDbFetcher, MongoDbWriter, createResponse, createHttpError, AppLogger
- Existing codebase: `trade-flow-api/src/auth/auth.guard.ts` -- JwtAuthGuard pattern to contrast against
- Node.js `crypto` module documentation -- `randomBytes()` for secure token generation
- NestJS official docs -- `HttpStatus.GONE` (410), `HttpException`, controller routing

### Secondary (MEDIUM confidence)
- [NestJS Throttler GitHub](https://github.com/nestjs/throttler) -- v6.5.0 confirmed NestJS 11 support
- [NestJS Throttler npm](https://www.npmjs.com/package/@nestjs/throttler) -- latest version 6.5.0
- [NestJS Throttler Issue #2235](https://github.com/nestjs/throttler/issues/2235) -- NestJS 11 compatibility confirmed resolved in 6.5.0

### Tertiary (LOW confidence)
- None -- all findings verified with primary or secondary sources

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- all libraries are either built-in (crypto), already in use (mongodb, luxon), or official NestJS ecosystem (throttler)
- Architecture: HIGH -- follows exact same patterns already established in the codebase (Controller -> Service -> Repository)
- Pitfalls: HIGH -- identified through direct analysis of existing code (auth bypass, response field filtering, enrichment dependencies)

**Research date:** 2026-03-15
**Valid until:** 2026-04-15 (stable domain, no rapidly changing libraries)
