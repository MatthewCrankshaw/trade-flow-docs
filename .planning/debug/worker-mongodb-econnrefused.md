---
status: investigating
trigger: "Worker service crashes on startup — ECONNREFUSED localhost:27017 in Railway (containerized)"
created: 2026-03-29T00:00:00Z
updated: 2026-03-29T00:00:00Z
---

## Current Focus

hypothesis: Worker Railway service is missing MONGODB_URI env var (or uses a different var name), falling back to localhost:27017 default
test: Read src/worker.ts and the worker's module DB config to find what env var name it reads
expecting: Either a hardcoded localhost fallback, a different env var name than the API uses, or a separate MongoClient config not reading MONGODB_URI
next_action: Open trade-flow-api repo and read src/worker.ts (or src/main-worker.ts), then the subscription module's MongoDB connection setup

## Symptoms

expected: Worker service starts successfully, SubscriptionRepository connects to MongoDB, worker processes jobs
actual: Worker crashes immediately on startup. SubscriptionRepository.ensureIndexes() fails with MongoServerSelectionError trying localhost:27017. Repeating crash loop every ~30s.
errors: |
  MongoServerSelectionError: connect ECONNREFUSED ::1:27017, connect ECONNREFUSED 127.0.0.1:27017
  at SubscriptionRepository.ensureIndexes (.../subscription/repositories/subscription.repository.js:112:20)
  at SubscriptionRepository.onModuleInit (.../subscription/repositories/subscription.repository.js:28:9)

  Server in topology description: 'localhost:27017' — connection string contains localhost, not real host
reproduction: Worker service deploys to Railway and crashes. Logs show it trying to connect to localhost:27017 in a container environment where no MongoDB is running locally.
started: New worker service — likely newly deployed, config not yet set up on Railway

## Eliminated

(none yet)

## Evidence

- timestamp: 2026-03-29T00:00:00Z
  checked: trade-flow-docs repo for worker source files
  found: No worker source files in docs repo — trade-flow-api is a separate repo not present in this workspace
  implication: Investigation must be done inside the trade-flow-api repo directly

## Resolution

root_cause:
fix:
verification:
files_changed: []
