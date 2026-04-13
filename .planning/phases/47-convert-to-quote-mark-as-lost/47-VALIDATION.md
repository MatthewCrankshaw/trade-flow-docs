---
phase: 47
slug: convert-to-quote-mark-as-lost
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-13
---

# Phase 47 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | jest 30.2.0 (API), vitest 4.1.3 (UI) |
| **Config file** | `trade-flow-api/jest.config.ts`, `trade-flow-ui/vitest.config.ts` |
| **Quick run command** | `npm run test -- --testPathPattern=estimate` |
| **Full suite command** | `npm run ci` |
| **Estimated runtime** | ~30 seconds |

---

## Sampling Rate

- **After every task commit:** Run `npm run test -- --testPathPattern=estimate`
- **After every plan wave:** Run `npm run ci`
- **Before `/gsd-verify-work`:** Full suite must be green
- **Max feedback latency:** 30 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Threat Ref | Secure Behavior | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|------------|-----------------|-----------|-------------------|-------------|--------|
| 47-01-01 | 01 | 1 | CONV-01 | — | N/A | unit | `npm run test -- --testPathPattern=estimate-to-quote-converter` | ❌ W0 | ⬜ pending |
| 47-01-02 | 01 | 1 | CONV-02 | — | Idempotency key prevents duplicate quote creation | unit | `npm run test -- --testPathPattern=estimate-to-quote-converter` | ❌ W0 | ⬜ pending |
| 47-01-03 | 01 | 1 | CONV-03 | — | N/A | unit | `npm run test -- --testPathPattern=estimate-to-quote-converter` | ❌ W0 | ⬜ pending |
| 47-01-04 | 01 | 1 | CONV-04 | — | N/A | unit | `npm run test -- --testPathPattern=estimate-to-quote-converter` | ❌ W0 | ⬜ pending |
| 47-02-01 | 02 | 1 | LOST-01 | — | N/A | unit | `npm run test -- --testPathPattern=estimate-lost-marker` | ❌ W0 | ⬜ pending |
| 47-02-02 | 02 | 1 | LOST-02 | — | Follow-ups cancelled on mark-lost | unit | `npm run test -- --testPathPattern=estimate-lost-marker` | ❌ W0 | ⬜ pending |
| 47-03-01 | 03 | 2 | CONV-05 | — | N/A | unit | `npm run test -- --testPathPattern=convert` | ❌ W0 | ⬜ pending |
| 47-03-02 | 03 | 2 | CONV-06 | — | N/A | unit | `npm run test -- --testPathPattern=convert` | ❌ W0 | ⬜ pending |
| 47-04-01 | 04 | 2 | LOST-01 | — | N/A | unit | `npm run test -- --testPathPattern=mark-lost` | ❌ W0 | ⬜ pending |
| 47-04-02 | 04 | 2 | LOST-02 | — | N/A | unit | `npm run test -- --testPathPattern=mark-lost` | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `src/estimate/test/services/estimate-to-quote-converter.service.spec.ts` — stubs for CONV-01 to CONV-04
- [ ] `src/estimate/test/services/estimate-lost-marker.service.spec.ts` — stubs for LOST-01, LOST-02

*Existing infrastructure covers framework and fixture needs.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Convert button opens quote in edit mode | CONV-05 | UI interaction flow | Click "Convert to Quote" on estimate detail, verify quote editor opens |
| Back-link navigation works | CONV-06 | Cross-page navigation | Click "Converted from E-YYYY-NNN" link on quote detail, verify estimate opens |
| Locked estimate hides Edit/Revise buttons | CONV-05 | Visual UI state | Convert estimate, return to detail, verify no Edit/Revise actions |
| Lost reason visible on locked detail | LOST-02 | Visual UI state | Mark estimate as lost, verify reason displayed on detail page |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 30s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
