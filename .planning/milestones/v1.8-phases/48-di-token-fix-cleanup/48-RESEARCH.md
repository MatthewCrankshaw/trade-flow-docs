# Phase 48: DI Token Fix & Cleanup - Research

**Researched:** 2026-04-15
**Domain:** NestJS dependency injection, dead code removal, frontend type cleanup
**Confidence:** HIGH

## Summary

Phase 48 addresses four discrete cleanup items identified in the v1.8 milestone audit. The most critical is a NestJS DI token resolution bug where `EstimateModule` registers a local `{ provide: ESTIMATE_FOLLOWUP_CANCELLER, useClass: NoopEstimateFollowupCanceller }` provider that silently overrides the real `BullMQEstimateFollowupCanceller` exported by `EstimateFollowupsModule`. In NestJS, local providers always take precedence over imported module exports, meaning all follow-up cancellation calls within `EstimateModule` services (EstimateEmailSender, EstimateReviser, EstimateToQuoteConverter, EstimateLostMarker, EstimateResponseHandler) resolve to a no-op in production.

The remaining three items are straightforward dead code removal: (1) `NoopEstimateFollowupCanceller` is registered twice in `EstimateModule` providers (once as a class, once as a DI token), (2) `ConvertEstimateRequest` is an empty class defined but never imported anywhere, and (3) `site_visit_requested` status was removed from the backend enum in Phase 45-01 but remains in 6 frontend files creating dead branches.

**Primary recommendation:** Remove the local `ESTIMATE_FOLLOWUP_CANCELLER` provider and the `NoopEstimateFollowupCanceller` class provider from `EstimateModule`, relying entirely on the token exported by `EstimateFollowupsModule`. Delete the unused `ConvertEstimateRequest` file. Strip `site_visit_requested` from all 6 frontend files. Update affected tests.

## Standard Stack

No new libraries required. This phase operates entirely within the existing NestJS 11 and React 19 codebases.

### Core (existing, no changes)
| Library | Version | Purpose | Role in Phase |
|---------|---------|---------|---------------|
| @nestjs/core | 11.1.12 | DI container | Token resolution fix target |
| @nestjs/bullmq | (installed) | BullMQ integration | Provides real canceller |
| React | 19.2.0 | Frontend framework | Type/component cleanup |
| TypeScript | 5.9.3 | Both repos | Compile-time validation of removals |

## Architecture Patterns

### NestJS DI Token Resolution Order

The critical knowledge for this phase: [VERIFIED: codebase inspection + NestJS documentation pattern]

In NestJS, when a module both imports another module AND declares a local provider for the same token, the **local provider wins**. This is the root cause of the bug.

**Current state (broken):**
```
EstimateModule
  imports: [EstimateFollowupsModule]   // exports ESTIMATE_FOLLOWUP_CANCELLER -> BullMQ
  providers: [
    NoopEstimateFollowupCanceller,     // class provider (line 80) -- DUPLICATE
    { provide: ESTIMATE_FOLLOWUP_CANCELLER, useClass: NoopEstimateFollowupCanceller }  // token (line 81) -- OVERRIDES import
  ]
```

**Target state (fixed):**
```
EstimateModule
  imports: [EstimateFollowupsModule]   // exports ESTIMATE_FOLLOWUP_CANCELLER -> BullMQ
  providers: [
    // NO local ESTIMATE_FOLLOWUP_CANCELLER provider
    // NO NoopEstimateFollowupCanceller class provider
  ]
```

The `NoopEstimateFollowupCanceller` class itself should remain in the codebase at its current location (`src/estimate/services/`) because it is the legitimate fallback for test environments where unit tests mock the DI token with `useValue: mockFollowupCanceller`. The class is also used directly in `estimate-reviser.service.spec.ts` via `useClass: NoopEstimateFollowupCanceller`. The Noop class stays; only its registration in `EstimateModule.providers` is removed.

### Anti-Patterns to Avoid
- **Removing the Noop class entirely:** The class is used in tests (`estimate-reviser.service.spec.ts` line 57). Only remove its registration from the module providers array.
- **Moving Noop to EstimateFollowupsModule:** The Noop is not needed there -- that module always has BullMQ. The Noop exists for unit test isolation only.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| DI token fallback | Custom conditional logic | NestJS module import/export | The framework handles this natively |

## Common Pitfalls

### Pitfall 1: Removing the NoopEstimateFollowupCanceller class
**What goes wrong:** Tests that use `{ provide: ESTIMATE_FOLLOWUP_CANCELLER, useClass: NoopEstimateFollowupCanceller }` break.
**Why it happens:** Confusing "remove from module registration" with "delete the file".
**How to avoid:** Only remove the two lines from `EstimateModule.providers` (lines 80-81). Keep the class file.
**Warning signs:** Test compilation failures in `estimate-reviser.service.spec.ts`.

### Pitfall 2: Forgetting to remove ESTIMATE_FOLLOWUP_CANCELLER from EstimateModule exports
**What goes wrong:** EstimateModule re-exports the token (line 95), which could create confusion about who owns the token.
**Why it happens:** The export was added when EstimateModule was the original owner of the token.
**How to avoid:** Remove `ESTIMATE_FOLLOWUP_CANCELLER` from `EstimateModule.exports` array (line 95). The canonical export point is `EstimateFollowupsModule`.
**Warning signs:** Circular or ambiguous token resolution in other modules that import EstimateModule.

### Pitfall 3: Incomplete site_visit_requested removal
**What goes wrong:** TypeScript compiles fine but a dead value remains in a filter array or status map.
**Why it happens:** Status maps are `Record<EstimateStatus, ...>` -- removing from the type union forces removal from all maps. But inline arrays like `["sent", "viewed", "responded", "site_visit_requested"]` are plain strings and won't trigger type errors.
**How to avoid:** Use grep to find ALL occurrences (6 files confirmed), not just type-driven removals.
**Warning signs:** `npm run ci` passes but dead string literals remain.

### Pitfall 4: Breaking the estimate-reviser DI test
**What goes wrong:** `estimate-reviser.service.spec.ts` has a test "resolves ESTIMATE_FOLLOWUP_CANCELLER to NoopEstimateFollowupCanceller" (line 261). This test validates the old (broken) behavior.
**Why it happens:** The test was written when the Noop was the intended resolution for Phase 42 (before BullMQ was wired in Phase 46).
**How to avoid:** Update or remove this test. The spec already uses `useClass: NoopEstimateFollowupCanceller` in its test module setup (line 57), which is correct for unit test isolation. The assertion about "DI injection resolution" on line 260-263 should be removed since it tests module-level wiring, not unit behavior.
**Warning signs:** Test still passes but asserts wrong production behavior.

## Code Examples

### Fix 1: EstimateModule provider cleanup

**File:** `trade-flow-api/src/estimate/estimate.module.ts`

Remove from providers array (lines 80-81):
```typescript
// REMOVE these two lines:
NoopEstimateFollowupCanceller,
{ provide: ESTIMATE_FOLLOWUP_CANCELLER, useClass: NoopEstimateFollowupCanceller },
```

Remove from exports array (line 95):
```typescript
// REMOVE this line:
ESTIMATE_FOLLOWUP_CANCELLER,
```

Remove unused imports (lines 32, 35):
```typescript
// REMOVE these imports:
import { ESTIMATE_FOLLOWUP_CANCELLER } from "@estimate/services/estimate-followup-canceller.interface";
import { NoopEstimateFollowupCanceller } from "@estimate/services/noop-estimate-followup-canceller.service";
```

### Fix 2: Delete ConvertEstimateRequest

**File:** `trade-flow-api/src/estimate/requests/convert-estimate.request.ts`

Delete this file entirely. It contains only `export class ConvertEstimateRequest {}` and is never imported anywhere. [VERIFIED: grep for both class name and file path returned zero import references]

### Fix 3: Remove site_visit_requested from frontend

**6 files to update:**

1. `trade-flow-ui/src/types/estimate.ts` (line 8) -- remove from `EstimateStatus` union type
2. `trade-flow-ui/src/pages/EstimateDetailPage.tsx` (line 48) -- remove from status-to-badge-variant map
3. `trade-flow-ui/src/pages/EstimatesPage.tsx` (lines 30, 33) -- remove from `TAB_STATUS_GROUPS` all and responded arrays
4. `trade-flow-ui/src/features/estimates/components/EstimatesCardList.tsx` (line 24) -- remove from status badge variant map
5. `trade-flow-ui/src/features/estimates/components/EstimatesTable.tsx` (line 24) -- remove from status badge variant map
6. `trade-flow-ui/src/features/estimates/components/EstimateActionStrip.tsx` (line 50) -- remove from `canResend` status array

## Affected Files Inventory

### Backend (trade-flow-api)

| File | Change Type | Details |
|------|------------|---------|
| `src/estimate/estimate.module.ts` | Edit | Remove Noop provider, Noop class, token from providers and exports; remove 2 imports |
| `src/estimate/requests/convert-estimate.request.ts` | Delete | Empty dead class, zero references |
| `src/estimate/test/services/estimate-reviser.service.spec.ts` | Edit | Remove/update "DI injection resolution" test (lines 260-264) |

### Frontend (trade-flow-ui)

| File | Change Type | Details |
|------|------------|---------|
| `src/types/estimate.ts` | Edit | Remove `"site_visit_requested"` from `EstimateStatus` union |
| `src/pages/EstimateDetailPage.tsx` | Edit | Remove `site_visit_requested` entry from badge variant map |
| `src/pages/EstimatesPage.tsx` | Edit | Remove from 2 arrays in `TAB_STATUS_GROUPS` |
| `src/features/estimates/components/EstimatesCardList.tsx` | Edit | Remove from badge variant map |
| `src/features/estimates/components/EstimatesTable.tsx` | Edit | Remove from badge variant map |
| `src/features/estimates/components/EstimateActionStrip.tsx` | Edit | Remove from `canResend` status array |

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework (API) | Jest 30.2.0 |
| Framework (UI) | Vitest 4.1.3 |
| Quick run (API) | `npm run test -- --passWithNoTests` |
| Quick run (UI) | `npm run test` |
| Full suite (API) | `npm run ci` |
| Full suite (UI) | `npm run ci` |

### Phase Requirements to Test Map

| SC # | Behavior | Test Type | Automated Command | Exists? |
|------|----------|-----------|-------------------|---------|
| SC-1 | EstimateModule no longer registers local ESTIMATE_FOLLOWUP_CANCELLER | unit | `npm run test -- estimate-reviser.service.spec.ts` (updated) | Needs update |
| SC-2 | NoopEstimateFollowupCanceller registered exactly once (in tests only) | manual grep | `grep -r "NoopEstimateFollowupCanceller" src/estimate/estimate.module.ts` | N/A |
| SC-3 | ConvertEstimateRequest removed | smoke | Verify file deleted + `npm run ci` passes | N/A |
| SC-4 | site_visit_requested removed from frontend | unit + typecheck | `npm run ci` in trade-flow-ui | Implicit via TypeScript |

### Wave 0 Gaps
None -- existing test infrastructure covers all phase requirements. Removing `site_visit_requested` from the `EstimateStatus` type union will cause TypeScript compilation errors in any file that still references it, serving as a compile-time validation gate.

## Security Domain

### Applicable ASVS Categories

| ASVS Category | Applies | Rationale |
|---------------|---------|-----------|
| V2 Authentication | No | No auth changes |
| V3 Session Management | No | No session changes |
| V4 Access Control | No | No authorization changes |
| V5 Input Validation | No | Removing an empty request class (no validation impact) |
| V6 Cryptography | No | No crypto changes |

This phase is a pure code cleanup with no security surface changes. The DI fix restores *intended* security behavior (follow-up cancellation actually works), but introduces no new attack vectors.

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Phase 42: Noop canceller as placeholder | Phase 46: BullMQ real canceller | Phase 46 execution | Local Noop provider was never removed, creating the override bug |
| Phase 41: site_visit_requested in status enum | Phase 45-01: removed from backend | Phase 45-01 execution | Frontend was not updated in the same plan |

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | BullMQ/Redis is always available in production (no need for Noop fallback in prod) | Architecture Patterns | If Redis is sometimes absent, removing the Noop fallback would cause DI resolution failure at startup |
| A2 | No other module imports EstimateModule specifically for the ESTIMATE_FOLLOWUP_CANCELLER export | Pitfall 2 | Removing from exports could break another module's DI |

**A1 mitigation:** QueueModule throws on missing REDIS_URL [VERIFIED: queue.module.ts line 15-17], confirming Redis is a hard requirement. The Noop was a Phase 42 development artifact, not a production fallback.

**A2 mitigation:** grep for `ESTIMATE_FOLLOWUP_CANCELLER` shows the token is only injected within EstimateModule services and tested in EstimateModule test files. EstimateFollowupsModule is the canonical exporter. [VERIFIED: codebase grep]

## Open Questions

None -- all four success criteria are fully understood with verified codebase evidence.

## Environment Availability

Step 2.6: SKIPPED (no external dependencies -- this phase is purely code/config changes in both repos).

## Sources

### Primary (HIGH confidence)
- Codebase inspection: `trade-flow-api/src/estimate/estimate.module.ts` -- DI registration pattern verified
- Codebase inspection: `trade-flow-api/src/estimate-followups/estimate-followups.module.ts` -- BullMQ canceller export verified
- Codebase inspection: `trade-flow-api/src/estimate/requests/convert-estimate.request.ts` -- empty class, zero imports verified via grep
- Codebase inspection: 6 frontend files with `site_visit_requested` confirmed via grep
- Codebase inspection: `trade-flow-api/src/estimate/enums/estimate-status.enum.ts` -- no `site_visit_requested` in backend enum (removed in Phase 45-01)
- Codebase inspection: `trade-flow-api/src/queue/queue.module.ts` -- REDIS_URL is required (throws on missing)
- `.planning/v1.8-MILESTONE-AUDIT.md` -- gap identification source

### Secondary (MEDIUM confidence)
- NestJS DI resolution order (local providers override imported exports) -- well-documented NestJS behavior [ASSUMED from training knowledge, consistent with observed bug behavior]

## Metadata

**Confidence breakdown:**
- DI token fix: HIGH -- codebase evidence directly confirms the bug and the fix
- Dead code removal: HIGH -- grep confirms zero references to ConvertEstimateRequest
- Frontend cleanup: HIGH -- exact files and line numbers identified, TypeScript will enforce completeness
- Test impact: HIGH -- affected test identified and fix approach clear

**Research date:** 2026-04-15
**Valid until:** 2026-05-15 (stable -- no upstream changes expected)
