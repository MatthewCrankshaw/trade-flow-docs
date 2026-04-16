---
phase: 48-di-token-fix-cleanup
verified: 2026-04-16T07:00:00Z
status: gaps_found
score: 3/4
overrides_applied: 0
gaps:
  - truth: "site_visit_requested status is removed from all frontend types, components, filter tabs, and status mappings — no dead branches remain"
    status: failed
    reason: "\"site_visit_requested\" literal survives in REVISABLE_STATUSES tuple in EstimateActionStrip.tsx line 15. The tuple is cast to `readonly string[]` at the call site (line 53) which defeats TypeScript's union exhaustiveness check and prevented the compiler from catching this during Plan 02 execution."
    artifacts:
      - path: "trade-flow-ui/src/features/estimates/components/EstimateActionStrip.tsx"
        issue: "Line 15: `const REVISABLE_STATUSES = [\"sent\", \"viewed\", \"responded\", \"site_visit_requested\"] as const;` — stale literal not removed. Line 53: `(REVISABLE_STATUSES as readonly string[]).includes(estimate.status)` widening cast hides the dead member from TypeScript."
    missing:
      - "Remove `\"site_visit_requested\"` from the REVISABLE_STATUSES tuple on line 15"
      - "Replace the `as readonly string[]` widening cast on line 53 with proper `readonly EstimateStatus[]` typing on the constant declaration so TypeScript enforces exhaustiveness"
---

# Phase 48: DI Token Fix & Cleanup — Verification Report

**Phase Goal:** Fix the critical DI token resolution risk where NoopEstimateFollowupCanceller overrides the real BullMQ canceller, clean up dead code (duplicate noop registration, empty ConvertEstimateRequest), and remove the dead `site_visit_requested` status from 6+ frontend files after its backend removal in Phase 45-01.
**Verified:** 2026-04-16T07:00:00Z
**Status:** gaps_found
**Re-verification:** No — initial verification

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | `EstimateModule` no longer registers a local `{ provide: ESTIMATE_FOLLOWUP_CANCELLER, useClass: NoopEstimateFollowupCanceller }` provider — `BullMQEstimateFollowupCanceller` from `EstimateFollowupsModule` is the only binding resolved in production | VERIFIED | `grep "NoopEstimateFollowupCanceller" estimate.module.ts` → 0 matches. `grep "ESTIMATE_FOLLOWUP_CANCELLER" estimate.module.ts` → 0 matches. `EstimateFollowupsModule` is imported via `forwardRef(() => EstimateFollowupsModule)` and exports `ESTIMATE_FOLLOWUP_CANCELLER` bound to `BullMQEstimateFollowupCanceller` via `useExisting`. Commit `13bba57` confirmed on `trade-flow-api` main. |
| 2 | `NoopEstimateFollowupCanceller` is registered exactly once (in `EstimateFollowupsModule` for test/dev), not duplicated | VERIFIED | `grep -r "NoopEstimateFollowupCanceller" trade-flow-api/src/` — zero matches in `estimate.module.ts` and zero matches in `estimate-followups.module.ts`. Class file at `estimate/services/noop-estimate-followup-canceller.service.ts` still exists for test isolation use. It appears only in the test spec's `useClass` wiring (correct). Note: `EstimateFollowupsModule` does not directly register `NoopEstimateFollowupCanceller` — it registers `BullMQEstimateFollowupCanceller` via `useExisting`. The Noop is test-only; this satisfies the intent of SC-2 (no duplicate production registration). |
| 3 | `ConvertEstimateRequest` empty class is removed and its references updated | VERIFIED | `test ! -f trade-flow-api/src/estimate/requests/convert-estimate.request.ts` → DELETED. `grep -r "ConvertEstimateRequest" trade-flow-api/src/` → 0 matches. Commit `8dbefa4` confirmed on `trade-flow-api` main. |
| 4 | `site_visit_requested` status is removed from all frontend types, components, filter tabs, and status mappings — no dead branches remain | FAILED | One surviving occurrence: `trade-flow-ui/src/features/estimates/components/EstimateActionStrip.tsx:15` — `const REVISABLE_STATUSES = ["sent", "viewed", "responded", "site_visit_requested"] as const;`. Confirmed by direct grep. The other 5 files in the plan (estimate.ts, EstimateDetailPage, EstimatesPage, EstimatesCardList, EstimatesTable) are clean. The stale literal was missed because line 53 casts the tuple to `readonly string[]` before calling `.includes()`, which bypassed TypeScript's union check during CI. |

**Score:** 3/4 truths verified

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `trade-flow-api/src/estimate/estimate.module.ts` | Clean module with no local followup canceller override; imports `EstimateFollowupsModule` | VERIFIED | No `NoopEstimateFollowupCanceller`, no `ESTIMATE_FOLLOWUP_CANCELLER` in providers/exports. `EstimateFollowupsModule` imported via `forwardRef`. |
| `trade-flow-api/src/estimate/requests/convert-estimate.request.ts` | Deleted — empty class with zero references | VERIFIED | File does not exist. Zero remaining references in codebase. |
| `trade-flow-ui/src/types/estimate.ts` | Clean `EstimateStatus` union without `site_visit_requested` | VERIFIED | Union contains: `"draft" \| "sent" \| "viewed" \| "responded" \| "converted" \| "declined" \| "expired" \| "lost" \| "deleted"`. No `site_visit_requested`. |
| `trade-flow-ui/src/features/estimates/components/EstimateActionStrip.tsx` | No `site_visit_requested` references | STUB/FAILED | Line 15 contains `"site_visit_requested"` in `REVISABLE_STATUSES`. Line 53 uses `as readonly string[]` cast hiding the type error. |

### Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| `trade-flow-api/src/estimate/estimate.module.ts` | `trade-flow-api/src/estimate-followups/estimate-followups.module.ts` | NestJS module import (`EstimateFollowupsModule`) | WIRED | `forwardRef(() => EstimateFollowupsModule)` in imports array confirmed at line 53. `EstimateFollowupsModule` exports `ESTIMATE_FOLLOWUP_CANCELLER` bound to `BullMQEstimateFollowupCanceller`. |
| `trade-flow-ui/src/types/estimate.ts` | `trade-flow-ui/src/pages/EstimateDetailPage.tsx` | TypeScript type import (`EstimateStatus`) | WIRED | `EstimatesPage.tsx` imports `EstimateStatus` from `@/types` and uses it in `TAB_STATUS_GROUPS: Record<EstimateTabValue, readonly EstimateStatus[]>`. No `site_visit_requested` present in that file. |

### Data-Flow Trace (Level 4)

Not applicable — this phase is pure dead-code cleanup with no new dynamic data rendering. No new components or data flows were introduced.

### Behavioral Spot-Checks

| Behavior | Check | Result | Status |
|----------|-------|--------|--------|
| `estimate.module.ts` has no local ESTIMATE_FOLLOWUP_CANCELLER provider | `grep "ESTIMATE_FOLLOWUP_CANCELLER" estimate.module.ts` | 0 matches | PASS |
| `ConvertEstimateRequest` file deleted | `test ! -f convert-estimate.request.ts` | DELETED | PASS |
| `EstimateStatus` type excludes `site_visit_requested` | Read `trade-flow-ui/src/types/estimate.ts` | Union clean | PASS |
| Zero `site_visit_requested` in all frontend files | `grep -r "site_visit_requested" trade-flow-ui/src/` | 1 match in `EstimateActionStrip.tsx:15` | FAIL |
| API commits present on main | `git log --oneline -5` in `trade-flow-api` | `13bba57`, `8dbefa4` confirmed | PASS |
| UI commit present on main | `git log --oneline -5` in `trade-flow-ui` | `8223315` confirmed | PASS |

### Requirements Coverage

No requirement IDs were declared in either plan's `requirements` field. The phase goal notes it "secures FUP-05, REV-05, LOST-02, CONV-05 indirectly" — these are not directly owned by this phase and are not verified here.

### Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| `trade-flow-ui/src/features/estimates/components/EstimateActionStrip.tsx` | 15 | `"site_visit_requested"` in `REVISABLE_STATUSES` tuple — dead literal from removed backend status | Blocker | Directly contradicts SC-4. Allows a future developer to infer this is a valid status. Propagation risk. |
| `trade-flow-ui/src/features/estimates/components/EstimateActionStrip.tsx` | 50, 51, 53 | `as readonly string[]` widening cast on status tuples defeats TypeScript union exhaustiveness — violates project rule against `as` type assertions | Warning | Caused SC-4 failure to go undetected through CI. Disables compiler guard for all four status checks. |
| `trade-flow-ui/src/features/estimates/components/EstimateActionStrip.tsx` | 80 | `estimate.convertedToQuoteId!` non-null assertion on optional field | Warning | Silently hides potential backend regression; violates project rule against non-null assertions. Not in scope for this phase but flagged by code review. |

### Human Verification Required

None — all checks are programmatically verifiable.

### Gaps Summary

SC-4 ("site_visit_requested removed from all frontend files") is not fully met. Five of the six target files are clean. The sixth, `EstimateActionStrip.tsx`, has one surviving occurrence at line 15 in the `REVISABLE_STATUSES` tuple. This was missed because the tuple is immediately widened to `readonly string[]` on line 53, which bypasses TypeScript's union narrowing and allowed the stale literal to pass the `tsc` check during Plan 02 CI.

The fix is a two-line change: remove `"site_visit_requested"` from the tuple on line 15, and replace the `as readonly string[]` cast pattern with `const REVISABLE_STATUSES: readonly EstimateStatus[] = [...]` so the compiler enforces correctness going forward.

The three SC-1 through SC-3 items are fully complete and verified against the actual codebase.

---

_Verified: 2026-04-16T07:00:00Z_
_Verifier: Claude (gsd-verifier)_
