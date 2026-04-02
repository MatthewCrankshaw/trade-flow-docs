---
phase: 36
slug: public-landing-page-and-route-restructure
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-02
---

# Phase 36 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | No frontend test framework (ESLint + TypeScript static analysis only) |
| **Config file** | `tsconfig.app.json` / `eslint.config.js` |
| **Quick run command** | `npx tsc --noEmit` |
| **Full suite command** | `npx tsc --noEmit && npx eslint .` |
| **Estimated runtime** | ~15 seconds |

---

## Sampling Rate

- **After every task commit:** Run `npx tsc --noEmit`
- **After every plan wave:** Run `npx tsc --noEmit && npx eslint .`
- **Before `/gsd:verify-work`:** Full suite must be green + `npx vite build` for bundle analysis
- **Max feedback latency:** 15 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 36-01-01 | 01 | 1 | LAND-01 | manual + static | `npx tsc --noEmit` | N/A | ⬜ pending |
| 36-01-02 | 01 | 1 | LAND-02 | manual | Visual inspection in browser | N/A | ⬜ pending |
| 36-01-03 | 01 | 1 | LAND-03 | manual | Visual inspection in browser | N/A | ⬜ pending |
| 36-01-04 | 01 | 1 | LAND-04 | manual | Visual inspection in browser | N/A | ⬜ pending |
| 36-01-05 | 01 | 1 | LAND-05 | manual | Click CTAs, verify navigation | N/A | ⬜ pending |
| 36-02-01 | 02 | 1 | LAND-06 | manual | Log in, visit `/`, verify redirect | N/A | ⬜ pending |
| 36-02-02 | 02 | 1 | -- | smoke | `npx vite build` + inspect chunk sizes | N/A | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

Existing infrastructure covers all phase requirements. No frontend test framework needed — project conventions confirm static analysis only for UI.

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Landing page renders at root URL | LAND-01 | UI visual — no test framework | Navigate to `/` in browser, verify hero, features, pricing, nav |
| Hero section with value proposition | LAND-02 | Visual layout check | Verify hero section has headline, subtext, and CTA button |
| Feature highlights for jobs/quotes/scheduling | LAND-03 | Visual layout check | Verify 3+ feature cards with icons and descriptions |
| Pricing section shows GBP 6/month | LAND-04 | Visual content check | Verify pricing card displays £6/month |
| Navigation to sign up or log in | LAND-05 | User interaction flow | Click CTAs, verify `/login` and `/login?mode=signup` navigation |
| Authenticated user redirected to dashboard | LAND-06 | Auth state interaction | Log in, visit `/`, verify redirect to `/dashboard` |
| Bundle isolation | -- | Build output inspection | Run `npx vite build`, verify landing chunk has no Redux/feature imports |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 15s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
