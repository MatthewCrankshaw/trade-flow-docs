---
phase: 21
slug: queue-module
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-22
---

# Phase 21 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Jest 30.2.0 |
| **Config file** | `package.json` jest section |
| **Quick run command** | `npm test -- --testPathPattern queue` |
| **Full suite command** | `npm test` |
| **Estimated runtime** | ~30 seconds |

---

## Sampling Rate

- **After every task commit:** Run `npm run validate` (typecheck + lint)
- **After every plan wave:** Run `npm test`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 30 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 21-01-01 | 01 | 1 | QUEUE-01 | smoke | `npm run start:dev` (boots without error) | N/A | ⬜ pending |
| 21-01-02 | 01 | 1 | QUEUE-02 | unit | `npm test -- --testPathPattern queue-producer` | ❌ W0 | ⬜ pending |
| 21-01-03 | 01 | 1 | QUEUE-03 | smoke | `npm run start:dev` (boots, connects to Redis) | N/A | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `src/queue/test/services/queue-producer.service.spec.ts` — stubs for QUEUE-02 (enqueueEcho calls queue.add)
- [ ] Framework install: None needed — Jest already configured with @queue/* moduleNameMapper alias

*Existing infrastructure covers most phase requirements. Only unit test file for QueueProducer needs creation.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Redis connection on startup | QUEUE-03 | Requires running Docker Redis | Run `docker compose up -d redis` then `npm run start:dev`, verify no connection errors in logs |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 30s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
