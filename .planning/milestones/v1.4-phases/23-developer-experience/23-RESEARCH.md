# Phase 23: Developer Experience - Research

**Researched:** 2026-03-22
**Domain:** Docker, nodemon, npm scripts, NestJS CLI build configuration
**Confidence:** HIGH

## Summary

Phase 23 is a pure tooling/configuration phase. No new libraries are installed, no application code is written. The work involves creating `nodemon-worker.json` for hot-reload development, adding `worker:dev` and `worker:prod` npm scripts, adding a worker service to `docker-compose.yaml`, and extending the multi-stage Dockerfile with a worker production stage.

All patterns are already established in the codebase -- the existing `nodemon.json`, `docker-compose.yaml` api service, and Dockerfile production stage serve as direct templates. The research confirms there are no edge cases or gotchas beyond what the CONTEXT.md decisions already address.

**Primary recommendation:** Follow the existing codebase patterns exactly -- copy-adapt from api equivalents with minimal deviation.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- D-01: Worker runs as a separate Docker Compose service with its own development target and volume mounts -- fully isolated from the API service
- D-02: Worker gets trimmed environment variables: only `MONGO_URL`, `REDIS_URL`, `NODE_ENV` -- no PORT, CORS_ORIGIN, FIREBASE_*, or RESEND_API_KEY
- D-03: Worker skips `cap_add: SYS_ADMIN` and `security_opt: seccomp=unconfined` -- those are for Playwright in the API service
- D-04: Container name `trade-flow-worker` following existing pattern (`trade-flow-api`, `trade-flow-db`, `trade-flow-redis`)
- D-05: Worker depends on `mongo` and `redis` (same as API), mounts source and runs `npm run worker:dev`
- D-06: Builder stage runs both `npm run build` (API) and `nest build --config worker-cli.json` (worker) sequentially -- produces both `dist/main.js` and `dist/worker.js`
- D-07: No separate `worker-development` Dockerfile stage -- Docker Compose worker service mounts source and runs `npm run worker:dev` from a generic Node image
- D-08: New `worker` production stage: copies `dist/` from builder, uses `node:22-slim` base (same as API production), runs `node dist/worker.js`
- D-09: No port exposed on worker production stage -- worker has no HTTP server
- D-10: New `nodemon-worker.json` file for worker hot reload
- D-11: `npm run worker:dev` script uses `nodemon --config nodemon-worker.json`
- D-12: `npm run worker:prod` script runs `node dist/worker.js`

### Claude's Discretion
- Whether worker nodemon uses `--inspect=9230` (different debug port) or skips `--inspect` entirely
- Watch scope: full `src/` or narrowed to `src/worker/`, `src/queue/`, `src/core/`
- Delay and signal settings (likely same 1500ms / SIGTERM as API)
- Exec command format (`node -r ts-node/register src/worker.ts` vs `ts-node src/worker.ts`)
- Whether to add a `build:worker` or `build:all` convenience script
- Worker Docker Compose development image choice (reuse `development` stage or plain `node:22`)

### Deferred Ideas (OUT OF SCOPE)
None -- discussion stayed within phase scope.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| DEVX-01 | `nodemon-worker.json` and `npm run worker:dev` script for hot-reload development | Existing `nodemon.json` is the template; exec command uses `node -r ts-node/register src/worker.ts`; use `--inspect=9230` for debug port separation |
| DEVX-02 | `npm run worker:prod` script for production worker startup | Simple `node dist/worker.js` -- matches existing `start:prod` pattern |
| DEVX-03 | Worker runs as separate Docker Compose service in local dev environment | Existing api service is the template; stripped env vars per D-02; reuse `development` stage |
| DEVX-04 | Multi-stage Dockerfile updated with worker production stage | Builder stage adds `nest build --config worker-cli.json`; new `worker` stage copies dist, no port, runs `node dist/worker.js` |
</phase_requirements>

## Standard Stack

No new libraries required. This phase modifies only configuration files.

### Tools Already in Place
| Tool | Version | Purpose | Role in Phase |
|------|---------|---------|---------------|
| nodemon | ^3.1.9 | File watcher with auto-restart | Worker hot reload via `nodemon-worker.json` |
| ts-node | ^10.9.2 | TypeScript execution without compilation | Worker dev execution via `-r ts-node/register` |
| @nestjs/cli | ^11.0.16 | NestJS build tooling | `nest build --config worker-cli.json` in builder stage |
| Docker | Multi-stage | Containerization | Worker production stage + compose service |

**No `npm install` needed for this phase.**

## Architecture Patterns

### Files to Create
```
trade-flow-api/
  nodemon-worker.json          # NEW - worker hot reload config
```

### Files to Modify
```
trade-flow-api/
  package.json                  # ADD worker:dev, worker:prod scripts
  docker-compose.yaml           # ADD worker service
  Dockerfile                    # UPDATE builder stage, ADD worker production stage
```

### Pattern 1: nodemon-worker.json (NEW file)

**What:** Separate nodemon config for worker process, mirroring existing `nodemon.json` for the API.

**Template (existing `nodemon.json`):**
```json
{
  "watch": ["src"],
  "ext": "ts",
  "ignore": ["src/**/*.spec.ts"],
  "exec": "node --inspect -r ts-node/register src/main.ts",
  "delay": "1500",
  "signal": "SIGTERM"
}
```

**Worker version -- recommended:**
```json
{
  "watch": ["src"],
  "ext": "ts",
  "ignore": ["src/**/*.spec.ts"],
  "exec": "node --inspect=9230 -r ts-node/register src/worker.ts",
  "delay": "1500",
  "signal": "SIGTERM"
}
```

**Discretion decisions (recommended):**
- **Debug port:** Use `--inspect=9230` -- allows simultaneous debugging of API (9229) and worker (9230). Omitting `--inspect` would mean no debugger attachment for the worker, which is a poor developer experience.
- **Watch scope:** Watch full `src/` -- the worker imports from `@core/*`, `@queue/*`, and `@worker/*`. Changes to shared modules (e.g., CoreModule, QueueModule) should also trigger restart. Narrowing to specific directories would miss shared dependency changes.
- **Delay/signal:** Same 1500ms / SIGTERM -- no reason to differ from the API config.
- **Exec format:** `node -r ts-node/register src/worker.ts` -- matches the existing API pattern exactly. Using `ts-node src/worker.ts` directly would also work but deviates from the established convention.

### Pattern 2: npm Scripts (package.json modification)

**Existing scripts to reference:**
```json
"start:dev": "nest start --watch",
"start:prod": "node dist/main",
"build:worker": "nest build --config worker-cli.json"
```

**Scripts to add:**
```json
"worker:dev": "nodemon --config nodemon-worker.json",
"worker:prod": "node dist/worker"
```

**Note:** `build:worker` already exists in package.json. A `build:all` convenience script (`npm run build && npm run build:worker`) could be useful but is not required by any DEVX requirement. Recommend adding it since it documents the builder stage's intent.

### Pattern 3: Docker Compose Worker Service

**Template (existing api service, stripped down per D-02/D-03):**
```yaml
worker:
  container_name: trade-flow-worker
  build:
    context: .
    target: development
  volumes:
    - .:/app
    - node_modules:/app/node_modules
  depends_on:
    mongo:
      condition: service_healthy
    redis:
      condition: service_healthy
  environment:
    MONGO_URL: mongodb://mongo:27017
    REDIS_URL: redis://redis:6379
    NODE_ENV: development
  networks:
    - app-network
  command: sh -c "npm install && npm run worker:dev"
```

**Key differences from api service:**
- No `ports` -- worker has no HTTP server
- No `CORS_ORIGIN`, `PORT`, `FIREBASE_*`, `PLAYWRIGHT_*`, `RESEND_API_KEY` env vars
- No `cap_add` or `security_opt` -- no Playwright/browser needs
- Uses `npm run worker:dev` instead of `npm run start:dev`

**Docker Compose dev image choice (recommended):** Reuse the existing `development` Dockerfile stage. It already has `npm install` (including devDependencies) and `nodemon` globally installed. Using a plain `node:22` image would require duplicating the dependency install logic.

### Pattern 4: Dockerfile Updates

**Builder stage update -- add worker build after API build:**
```dockerfile
# Build application
RUN npm run build
RUN nest build --config worker-cli.json
```

**Critical:** `worker-cli.json` has `deleteOutDir: false`, so building the worker after the API will NOT clobber `dist/main.js`. This was a locked decision from Phase 22.

**New worker production stage:**
```dockerfile
# Worker production stage
FROM node:22-slim AS worker

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install only production dependencies
RUN npm ci --only=production

# Copy built application from builder stage
COPY --from=builder /app/dist ./dist

# Set environment variables for production
ENV NODE_ENV=production

CMD ["node", "dist/worker.js"]
```

**Key differences from api production stage:**
- No `EXPOSE` -- worker has no HTTP server (D-09)
- CMD is `node dist/worker.js` instead of `npm run start:prod`
- Stage name is `worker` (not `production`)

### Anti-Patterns to Avoid
- **Sharing a single nodemon config for both API and worker:** They have different entry points and debug ports. Separate configs is the correct approach.
- **Adding HTTP port to worker:** The worker uses `createApplicationContext()` which has no HTTP server. Exposing a port would be misleading.
- **Using `nest build` with `deleteOutDir: true` for worker:** This would delete `dist/main.js`. The Phase 22 decision to set `deleteOutDir: false` in `worker-cli.json` prevents this.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| TypeScript hot reload | Custom file watcher | nodemon + ts-node | Already in devDependencies, proven pattern in codebase |
| Multi-target Docker build | Separate Dockerfiles | Multi-stage build | Single Dockerfile, shared builder stage, DRY |
| Process signal handling | Custom shutdown logic | Already in worker.ts | SIGTERM/SIGINT handlers already implemented in Phase 22 |

## Common Pitfalls

### Pitfall 1: Builder Stage Build Order
**What goes wrong:** Running `nest build --config worker-cli.json` before `npm run build` when worker-cli.json has `deleteOutDir: false` but nest-cli.json has `deleteOutDir: true`.
**Why it happens:** `npm run build` (which uses nest-cli.json with `deleteOutDir: true`) would wipe the entire `dist/` directory, including the previously built `dist/worker.js`.
**How to avoid:** Always build API first (`npm run build`), then worker (`nest build --config worker-cli.json`). The API build clears dist/, then the worker build adds to it.
**Warning signs:** `dist/worker.js` missing after Docker build.

### Pitfall 2: Debug Port Collision
**What goes wrong:** Both API and worker try to use `--inspect` on port 9229 simultaneously.
**Why it happens:** Default `--inspect` binds to port 9229. If both nodemon configs use the default, only one can start.
**How to avoid:** Worker uses `--inspect=9230` (or a different port).
**Warning signs:** "Address already in use" error when starting worker alongside API.

### Pitfall 3: node_modules Volume Sync in Docker Compose
**What goes wrong:** Worker container starts but can't find dependencies.
**Why it happens:** The `node_modules` named volume persists between container rebuilds. If package.json changes, the volume may have stale deps.
**How to avoid:** Use the same `command: sh -c "npm install && npm run worker:dev"` pattern that the api service uses. Both services share the same `node_modules` named volume.
**Warning signs:** Module not found errors in worker container logs.

### Pitfall 4: Missing tsconfig-paths Registration
**What goes wrong:** Worker dev mode crashes with "Cannot find module @worker/..." errors.
**Why it happens:** `ts-node` alone doesn't resolve path aliases. The `-r tsconfig-paths/register` flag is needed.
**How to avoid:** The exec command should be `node --inspect=9230 -r ts-node/register -r tsconfig-paths/register src/worker.ts`. Check what the existing API nodemon config uses -- if it works without `tsconfig-paths/register`, then the NestJS `nest start --watch` may handle this differently than raw nodemon.
**Warning signs:** Path alias resolution errors at runtime.

**CRITICAL CHECK:** The existing `nodemon.json` uses `node --inspect -r ts-node/register src/main.ts` -- it does NOT include `-r tsconfig-paths/register`. This works because `ts-node` respects `tsconfig.json` paths when configured properly, OR because NestJS bootstrapping resolves them. The worker uses the same `tsconfig.json`, so the same exec pattern should work. If it doesn't, add `-r tsconfig-paths/register` to the exec command.

## Code Examples

### nodemon-worker.json (complete file)
```json
{
  "watch": ["src"],
  "ext": "ts",
  "ignore": ["src/**/*.spec.ts"],
  "exec": "node --inspect=9230 -r ts-node/register src/worker.ts",
  "delay": "1500",
  "signal": "SIGTERM"
}
```

### package.json scripts additions
```json
"worker:dev": "nodemon --config nodemon-worker.json",
"worker:prod": "node dist/worker"
```

### docker-compose.yaml worker service
```yaml
# Worker Service
worker:
  container_name: trade-flow-worker
  build:
    context: .
    target: development
  volumes:
    - .:/app
    - node_modules:/app/node_modules
  depends_on:
    mongo:
      condition: service_healthy
    redis:
      condition: service_healthy
  environment:
    MONGO_URL: mongodb://mongo:27017
    REDIS_URL: redis://redis:6379
    NODE_ENV: development
  networks:
    - app-network
  command: sh -c "npm install && npm run worker:dev"
```

### Dockerfile builder stage (updated)
```dockerfile
# Build application
RUN npm run build
RUN nest build --config worker-cli.json
```

### Dockerfile worker production stage (new)
```dockerfile
# Worker production stage
FROM node:22-slim AS worker

WORKDIR /app

COPY package*.json ./

RUN npm ci --only=production

COPY --from=builder /app/dist ./dist

ENV NODE_ENV=production

CMD ["node", "dist/worker.js"]
```

## State of the Art

No changes to tooling since project inception. All tools (nodemon, Docker multi-stage, NestJS CLI) are stable and well-established.

| Aspect | Current State | Notes |
|--------|--------------|-------|
| nodemon | Stable, unchanged for years | JSON config file approach is standard |
| Docker multi-stage | Standard practice | Multiple named stages, `--target` for compose |
| NestJS CLI build | Supports `--config` flag | Allows separate build configs per entry point |

## Open Questions

1. **tsconfig-paths/register in nodemon exec**
   - What we know: Existing API nodemon config does NOT include `-r tsconfig-paths/register`, yet path aliases work.
   - What's unclear: Whether this is because `ts-node` reads `tsconfig.json` paths natively, or because something else resolves them. `tsconfig-paths` IS in devDependencies.
   - Recommendation: Start with the same exec format as the API (`node --inspect=9230 -r ts-node/register src/worker.ts`). If path aliases fail at runtime, add `-r tsconfig-paths/register`. This is a quick iteration if needed.

## Sources

### Primary (HIGH confidence)
- Existing codebase files: `nodemon.json`, `docker-compose.yaml`, `Dockerfile`, `package.json`, `worker-cli.json`, `nest-cli.json`, `src/worker.ts` -- all read directly from the repository
- CONTEXT.md decisions (D-01 through D-12) -- locked user decisions constraining implementation

### Secondary (MEDIUM confidence)
- nodemon JSON config format -- stable, well-documented, matches existing usage in codebase
- Docker multi-stage build patterns -- standard Docker documentation, matches existing Dockerfile

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH - no new libraries, all tools already in use
- Architecture: HIGH - direct copy-adapt from existing codebase patterns
- Pitfalls: HIGH - build order and port collision are well-understood issues

**Research date:** 2026-03-22
**Valid until:** 2026-04-22 (stable tooling, no version sensitivity)
