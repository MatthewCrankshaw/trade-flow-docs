# Phase 24: Playwright Bootstrap & Auth - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Downstream agents read `24-CONTEXT.md` instead.

**Session:** 2026-03-27
**Areas selected:** Firebase auth approach, Selector strategy, webServer config, Smoke test scope

---

## Firebase Auth Approach

**Q: How should the global setup project authenticate?**
Options: UI click-through first, Programmatic via REST API, Try UI first with fallback
**Selected:** Programmatic via REST API
*Reason: Firebase v12 IndexedDB persistence in Playwright storageState was flagged as MEDIUM confidence risk — programmatic approach avoids this uncertainty entirely.*

**Q: Should Phase 24 smoke test exercise the /login UI at all?**
Options: Auth is pure infrastructure, Phase 24 smoke test exercises /login UI too
**Selected:** Phase 24 smoke test exercises /login UI too
*Note: Even though auth setup is programmatic, the smoke test navigates to /login as an unauthenticated user to verify the login form renders.*

---

## Selector Strategy

**Q: How should Playwright tests locate UI elements?**
Options: ARIA/role-based, data-testid attributes, ARIA-first with data-testid fallback
**Selected:** ARIA-first with data-testid fallback
*ARIA/role selectors by default; add data-testid only where ARIA selectors are ambiguous. Minimal component changes.*

---

## webServer Config

**Q: Should playwright.config.ts auto-start the Vite dev server?**
Options: Auto-start Vite via webServer, Expect full stack already running
**Selected:** Auto-start Vite via webServer (reuseExistingServer: true)

**Q: What should happen if the NestJS API isn't reachable?**
Options: Fail fast with a clear error, Let tests fail naturally
**Selected:** Fail fast with a clear error
*Global setup pings API health endpoint before tests start; aborts with descriptive message if unreachable.*

---

## Smoke Test Scope

**Q: What should the Phase 24 smoke test verify?**
Options: Login UI renders + auth loads dashboard, Login UI + dashboard with API data, Login UI + protected route redirect
**Selected:** Login UI renders + auth loads dashboard
*Two assertions: (1) unauthenticated /login shows login form, (2) storageState → lands on /dashboard. No API data assertions.*

---

*Discussion completed: 2026-03-27*
