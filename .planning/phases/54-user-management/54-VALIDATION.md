---
phase: 54
slug: user-management
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-18
---

# Phase 54 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework (API)** | Jest 30.2.0 |
| **Framework (UI)** | Vitest 4.1.3 |
| **Config file (API)** | `trade-flow-api/package.json` (jest config) |
| **Config file (UI)** | `trade-flow-ui/vitest.config.ts` |
| **Quick run command (API)** | `npm run test -- --testPathPattern=support` |
| **Quick run command (UI)** | `npm run test -- --testPathPattern=support` |
| **Full suite command (API)** | `npm run ci` |
| **Full suite command (UI)** | `npm run ci` |
| **Estimated runtime** | ~30 seconds (per repo) |

---

## Sampling Rate

- **After every task commit:** Run `npm run test -- --testPathPattern=support` (in affected repo)
- **After every plan wave:** Run `npm run ci` in both repos
- **Before `/gsd-verify-work`:** Full suite must be green in both repos
- **Max feedback latency:** 30 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Threat Ref | Secure Behavior | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|------------|-----------------|-----------|-------------------|-------------|--------|
| 54-01-01 | 01 | 1 | UMGT-01 | T-54-01 | Regex special chars escaped in search | unit | `npm run test -- --testPathPattern=query-filter-parser` | ❌ W0 | ⬜ pending |
| 54-01-02 | 01 | 1 | UMGT-01 | T-54-02 | Filter operators validated against allowlist | unit | `npm run test -- --testPathPattern=query-filter-parser` | ❌ W0 | ⬜ pending |
| 54-02-01 | 02 | 1 | UMGT-01 | T-54-03 | Permission guard on support endpoints | unit | `npm run test -- --testPathPattern=support-user` | ❌ W0 | ⬜ pending |
| 54-02-02 | 02 | 1 | UMGT-01 | — | Paginated user list with subscription lookup | unit | `npm run test -- --testPathPattern=support-user` | ❌ W0 | ⬜ pending |
| 54-03-01 | 03 | 2 | UMGT-03 | T-54-04 | Firebase metadata only fetched server-side | unit | `npm run test -- --testPathPattern=firebase-auth-metadata` | ❌ W0 | ⬜ pending |
| 54-04-01 | 04 | 2 | UMGT-04 | — | N/A | unit | `npm run test -- --testPathPattern=dashboard-metrics` | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `src/core/test/utilities/query-filter-parser.utility.spec.ts` — stubs for UMGT-01 filter parsing
- [ ] `src/support/test/services/support-user-retriever.service.spec.ts` — stubs for UMGT-01, UMGT-03
- [ ] `src/support/test/services/support-dashboard-metrics.service.spec.ts` — stubs for UMGT-04
- [ ] `src/support/test/mocks/` — shared mock generators for support user, subscription

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Metric cards navigate to user list with pre-applied filter | UMGT-04 | Click navigation with URL params | Click each metric card, verify URL contains correct filter param and list shows filtered results |
| User list row click navigates to detail page | UMGT-02 | Click navigation | Click a user row, verify navigation to `/support/users/:id` with correct user data |
| Pagination buttons work at boundaries | UMGT-01 | UI interaction with server state | Navigate to first page (Previous disabled), last page (Next disabled), verify page count |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 30s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
