---
phase: 24
slug: playwright-bootstrap-auth
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-27
---

# Phase 24 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Playwright 1.51+ |
| **Config file** | `trade-flow-ui/playwright.config.ts` — Wave 0 installs |
| **Quick run command** | `npx playwright test --project=chromium` |
| **Full suite command** | `npx playwright test` |
| **Estimated runtime** | ~30 seconds |

---

## Sampling Rate

- **After every task commit:** Run `npx playwright test --project=chromium`
- **After every plan wave:** Run `npx playwright test`
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** 30 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 24-01-01 | 01 | 0 | FOUND-01 | install | `npx playwright test --version` | ❌ W0 | ⬜ pending |
| 24-01-02 | 01 | 0 | FOUND-01 | config | `npx playwright test --list` | ❌ W0 | ⬜ pending |
| 24-01-03 | 01 | 1 | FOUND-01 | e2e | `npx playwright test --project=setup` | ❌ W0 | ⬜ pending |
| 24-01-04 | 01 | 1 | FOUND-02 | e2e | `npx playwright test --project=chromium` | ❌ W0 | ⬜ pending |
| 24-01-05 | 01 | 1 | FOUND-02 | e2e smoke | `npx playwright test e2e/tests/smoke.spec.ts` | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `trade-flow-ui/playwright.config.ts` — Playwright config with webServer, projects, storageState
- [ ] `trade-flow-ui/e2e/global-setup.ts` — Global setup project (authenticates once, writes storageState)
- [ ] `trade-flow-ui/e2e/tests/smoke.spec.ts` — Smoke test stubs for FOUND-01, FOUND-02
- [ ] `trade-flow-ui/e2e/fixtures/` — Fixture directory
- [ ] `trade-flow-ui/e2e/helpers/` — Helpers directory
- [ ] `@playwright/test` installed in `trade-flow-ui` package.json

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Firebase test user pre-created in Console | FOUND-01 | Requires Firebase Console access, cannot be automated | Verify user exists at console.firebase.google.com → Authentication → Users |
| API health check message when NestJS not running | FOUND-01 | Requires intentional negative state | Stop API, run `npx playwright test` — confirm "API not reachable at http://localhost:3000" message appears |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 30s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
