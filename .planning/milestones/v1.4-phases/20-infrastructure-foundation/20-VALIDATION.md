---
phase: 20
slug: infrastructure-foundation
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-22
---

# Phase 20 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | jest 30.x (existing) |
| **Config file** | `trade-flow-api/package.json` (jest section) |
| **Quick run command** | `cd trade-flow-api && npx tsc --noEmit` |
| **Full suite command** | `cd trade-flow-api && npm run validate` |
| **Estimated runtime** | ~15 seconds |

---

## Sampling Rate

- **After every task commit:** Run `cd trade-flow-api && npx tsc --noEmit`
- **After every plan wave:** Run `cd trade-flow-api && npm run validate`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 15 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 20-01-01 | 01 | 1 | INFRA-01 | integration | `docker compose up -d redis && docker compose exec redis redis-cli CONFIG GET maxmemory-policy` | ✅ | ⬜ pending |
| 20-01-02 | 01 | 1 | INFRA-02 | config | `grep REDIS_URL trade-flow-api/.env.example` | ✅ | ⬜ pending |
| 20-01-03 | 01 | 1 | INFRA-03 | compile | `cd trade-flow-api && npx tsc --noEmit` | ✅ | ⬜ pending |
| 20-02-01 | 02 | 1 | INFRA-04 | compile | `cd trade-flow-api && npx tsc --noEmit` | ✅ | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

Existing infrastructure covers all phase requirements. No new test framework or stubs needed — validation is via TypeScript compilation, Docker Compose startup, and redis-cli commands.

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Redis container starts with noeviction | INFRA-01 | Requires Docker Compose running | `docker compose up -d redis && docker compose exec redis redis-cli CONFIG GET maxmemory-policy` — expect `noeviction` |
| Redis health check passes | INFRA-01 | Requires Docker Compose running | `docker compose ps redis` — expect `healthy` status |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 15s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
