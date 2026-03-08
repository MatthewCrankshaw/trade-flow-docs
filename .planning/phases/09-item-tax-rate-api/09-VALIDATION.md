---
phase: 9
slug: item-tax-rate-api
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-08
---

# Phase 9 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Jest (via NestJS Test utilities) |
| **Config file** | `jest` config in `package.json` |
| **Quick run command** | `npm run test -- --testPathPattern="item\|business\|quote" --no-coverage` |
| **Full suite command** | `npm run test` |
| **Estimated runtime** | ~30 seconds |

---

## Sampling Rate

- **After every task commit:** Run `npm run test -- --testPathPattern="item\|business\|quote" --no-coverage`
- **After every plan wave:** Run `npm run test`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 30 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 09-01-01 | 01 | 1 | ITAX-01 | unit | `npm run test -- --testPathPattern="item.repository" --no-coverage` | ❌ W0 | ⬜ pending |
| 09-01-02 | 01 | 1 | ITAX-02 | unit | `npm run test -- --testPathPattern="item-creator" --no-coverage` | ❌ W0 | ⬜ pending |
| 09-01-03 | 01 | 1 | ITAX-03 | unit | `npm run test -- --testPathPattern="item-updater" --no-coverage` | ❌ W0 | ⬜ pending |
| 09-01-04 | 01 | 1 | ITAX-04 | unit | `npm run test -- --testPathPattern="map-item-to-response\|map-create-item-request" --no-coverage` | ❌ W0 | ⬜ pending |
| 09-02-01 | 02 | 2 | ITAX-05 | unit | `npm run test -- --testPathPattern="business-creator\|default-business-items\|default-tax-rates" --no-coverage` | ✅ partial | ⬜ pending |
| 09-02-02 | 02 | 2 | QUOT-01 | unit | `npm run test -- --testPathPattern="quote-standard-line-item\|quote-bundle-line-item" --no-coverage` | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `src/item/test/services/item-creator.service.spec.ts` — stubs for ITAX-02
- [ ] `src/item/test/services/item-updater.service.spec.ts` — stubs for ITAX-03
- [ ] `src/item/test/controllers/mappers/map-create-item-request-to-dto.utility.spec.ts` — stubs for ITAX-04
- [ ] `src/item/test/controllers/mappers/map-item-to-response.utility.spec.ts` — stubs for ITAX-04
- [ ] `src/item/test/controllers/mappers/merge-existing-item-with-changes.utility.spec.ts` — stubs for ITAX-03/ITAX-04
- [ ] `src/item/test/repositories/item.repository.spec.ts` — stubs for ITAX-01
- [ ] `src/item/test/mocks/item-mock-generator.ts` — shared mock fixtures
- [ ] `src/quote/test/services/quote-standard-line-item-factory.service.spec.ts` — stubs for QUOT-01
- [ ] `src/quote/test/services/quote-bundle-line-item-factory.service.spec.ts` — stubs for QUOT-01
- [ ] Update `src/business/test/services/business-creator.service.spec.ts` — stubs for ITAX-05

---

## Manual-Only Verifications

*All phase behaviors have automated verification.*

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 30s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
