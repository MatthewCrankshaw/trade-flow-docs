---
phase: 37
slug: onboarding-wizard-pages
status: validated
nyquist_compliant: true
wave_0_complete: true
created: 2026-04-07
updated: 2026-04-07
---

# Phase 37 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework (API)** | Jest 30.x |
| **Config file (API)** | `trade-flow-api/package.json` jest section |
| **Quick run command (API)** | `cd trade-flow-api && npx jest --passWithNoTests` |
| **Full suite command (API)** | `cd trade-flow-api && npm run test` |
| **Framework (UI)** | Vitest 4.1.3 + @testing-library/react |
| **Config file (UI)** | `trade-flow-ui/vitest.config.ts` |
| **Quick run command (UI)** | `cd trade-flow-ui && npx vitest run src/features/onboarding src/features/auth/components/__tests__/OnboardingGuard.test.tsx src/features/subscription/components/__tests__/TrialBadge.test.tsx` |
| **Full suite command (UI)** | `cd trade-flow-ui && npm run test` |

---

## Sampling Rate

- **After every task commit:** Run `cd trade-flow-api && npx jest --passWithNoTests`
- **After every plan wave:** Run `cd trade-flow-api && npm run test`
- **Before `/gsd-verify-work`:** Full suite must be green
- **Max feedback latency:** ~5 seconds (API unit tests)

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 37-01-01 | 01 | 1 | ONBD-05, ONBD-06, ONBD-07 | unit (UI) | `cd trade-flow-ui && npx vitest run src/features/onboarding/components/__tests__/ProfileStep.test.tsx src/features/onboarding/hooks/__tests__/useOnboardingStep.test.ts` | ✅ | ✅ green |
| 37-01-02 | 01 | 1 | ONBD-01, ONBD-07 | unit (UI) | `cd trade-flow-ui && npx vitest run src/features/auth/components/__tests__/OnboardingGuard.test.tsx` | ✅ | ✅ green |
| 37-02-01 | 02 | 2 | ONBD-02 | unit (UI) | `cd trade-flow-ui && npx vitest run src/features/onboarding/components/__tests__/BusinessStep.test.tsx` | ✅ | ✅ green |
| 37-02-02 | 02 | 2 | ONBD-03 | unit (UI) | `cd trade-flow-ui && npx vitest run src/features/onboarding/components/__tests__/SetupLoadingScreen.test.tsx` | ✅ | ✅ green |
| 37-03-01 | 03 | 1 | TRIAL-02, TRIAL-03 | unit (UI) | `cd trade-flow-ui && npx vitest run src/features/subscription/components/__tests__/TrialBadge.test.tsx` | ✅ | ✅ green |
| 37-03-02 | 03 | 1 | TRIAL-02 | unit (UI) | `cd trade-flow-ui && npx vitest run src/features/subscription/components/__tests__/TrialBadge.test.tsx` | ✅ | ✅ green |
| 37-04-01 | 04 | 1 | ONBD-04 | unit (API) | `cd trade-flow-api && npx jest src/business/test/services/business-creator.service.spec.ts --no-coverage` | ✅ | ✅ green |
| 37-04-02 | 04 | 1 | ONBD-01-07, TRIAL-02-03 | docs | N/A | N/A | ✅ complete |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ escalated*

---

## Wave 0 Requirements

- [x] Install Vitest + @testing-library/react in `trade-flow-ui` for component unit testing
- [x] Configure `vitest.config.ts` with `test` block for Vitest
- [x] Globals enabled (describe, it, expect available without imports)

*Resolved by prior phase work (quick-260407-orl). Vitest 4.1.3 confirmed installed and configured.*

---

## Manual-Only Verifications

All previously manual-only behaviors are now covered by automated unit tests. No manual-only items remain for this phase.

---

## Nyquist Audit 2026-04-07 (gap closure)

| Metric | Count |
|--------|-------|
| Gaps found | 6 |
| Resolved | 6 |
| Escalated | 0 |

**Resolution:** Wave 0 infrastructure (Vitest) was already installed by prior phase work. All 6 UI gaps filled with behavioral unit tests using vi.mock for RTK Query hooks and React Router, @testing-library/react for rendering, @testing-library/user-event for interactions.

### Tests Created

| File | Tests | Requirements Covered |
|------|-------|----------------------|
| `src/features/onboarding/components/__tests__/ProfileStep.test.tsx` | 6 | ONBD-05 |
| `src/features/onboarding/hooks/__tests__/useOnboardingStep.test.ts` | 4 | ONBD-06 |
| `src/features/auth/components/__tests__/OnboardingGuard.test.tsx` | 5 | ONBD-01, ONBD-07 |
| `src/features/onboarding/components/__tests__/BusinessStep.test.tsx` | 7 | ONBD-02 |
| `src/features/onboarding/components/__tests__/SetupLoadingScreen.test.tsx` | 9 | ONBD-03 |
| `src/features/subscription/components/__tests__/TrialBadge.test.tsx` | 11 | TRIAL-02, TRIAL-03 |

**Total: 42 tests, all green.**

---

## Validation Sign-Off

- [x] All tasks have automated verify commands
- [x] Sampling continuity: API tests run after each commit
- [x] Wave 0 complete — Vitest installed and configured
- [x] No watch-mode flags
- [x] Feedback latency < 5s (API tests), < 2s (UI unit tests)
- [x] `nyquist_compliant: true` set in frontmatter

**Approval:** complete 2026-04-07
