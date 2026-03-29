---
phase: 32
slug: subscription-gate-and-subscribe-pages
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-29
---

# Phase 32 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | vitest (via Vite) |
| **Config file** | `vite.config.ts` |
| **Quick run command** | `npm test -- --run` |
| **Full suite command** | `npm test -- --run` |
| **Estimated runtime** | ~10 seconds |

---

## Sampling Rate

- **After every task commit:** Run `npm test -- --run`
- **After every plan wave:** Run `npm test -- --run`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 10 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 32-01-01 | 01 | 1 | GATE-01 | manual | Browser navigation | N/A | ⬜ pending |
| 32-01-02 | 01 | 1 | GATE-02 | manual | Browser navigation | N/A | ⬜ pending |
| 32-01-03 | 01 | 1 | GATE-03 | manual | Browser navigation | N/A | ⬜ pending |
| 32-01-04 | 01 | 1 | GATE-04 | manual | Browser check | N/A | ⬜ pending |
| 32-01-05 | 01 | 1 | GATE-05 | manual | Browser check | N/A | ⬜ pending |
| 32-02-01 | 02 | 2 | ACQ-04 | manual | Browser flow | N/A | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

Existing infrastructure covers all phase requirements. This is a frontend UI phase — validation is primarily visual/behavioral through browser testing.

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Business route redirect to /subscribe | GATE-01 | Route guard behavior requires browser navigation | Navigate to /customers without subscription, verify redirect to /subscribe |
| Support role bypass | GATE-02 | Role-based routing requires auth context | Login as support user, navigate to business routes, verify no redirect |
| Settings accessible without subscription | GATE-03 | Route structure verification | Navigate to /settings without subscription, verify page loads |
| No flash for subscribed users | GATE-04 | Visual timing behavior | Login with active subscription, navigate to business route, verify no flash |
| Loading skeleton during fetch | GATE-05 | Visual loading state | Throttle network, navigate to business route, verify skeleton shown |
| Subscribe success polling | ACQ-04 | Async webhook + polling flow | Complete checkout, land on /subscribe/success, verify polling then confirmation |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 10s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
