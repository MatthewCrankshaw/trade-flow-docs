---
phase: 50
slug: response-display-convert-route-fix
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-15
---

# Phase 50 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | jest 30.x (API), vitest 4.x (UI) |
| **Config file** | `trade-flow-api/jest.config.ts`, `trade-flow-ui/vitest.config.ts` |
| **Quick run command** | `npm run test -- --testPathPattern={file}` |
| **Full suite command** | `npm run ci` |
| **Estimated runtime** | ~30 seconds |

---

## Sampling Rate

- **After every task commit:** Run `npm run test -- --testPathPattern={file}`
- **After every plan wave:** Run `npm run ci`
- **Before `/gsd-verify-work`:** Full suite must be green
- **Max feedback latency:** 30 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Threat Ref | Secure Behavior | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|------------|-----------------|-----------|-------------------|-------------|--------|
| 50-01-01 | 01 | 1 | RESP-08 | — | N/A | unit | `npm run test -- --testPathPattern=estimate` | ✅ | ⬜ pending |
| 50-01-02 | 01 | 1 | RESP-08 | — | N/A | unit | `npm run test -- --testPathPattern=estimate` | ✅ | ⬜ pending |
| 50-02-01 | 02 | 2 | RESP-08 | — | N/A | unit | `npm run test -- --testPathPattern=Estimate` | ❌ W0 | ⬜ pending |
| 50-02-02 | 02 | 2 | CONV-03 | — | N/A | unit | `npm run test -- --testPathPattern=App` | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- Existing infrastructure covers all phase requirements.

*Backend has existing estimate test structure. Frontend components can be verified via CI gate.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Response card renders correctly with real data | RESP-08 | Visual verification of card layout | Navigate to estimate detail, verify response card shows type/message/reason/timestamp |
| Convert navigate lands on quote edit | CONV-03 | Route navigation requires browser | Click Convert to Quote, verify redirect to quote edit page |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 30s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
