---
status: resolved
trigger: "PATCH /v1/business/:id/quote-settings returns 500 with MongoDB ConflictingUpdateOperators on 'updatedAt'"
created: 2026-03-21T00:00:00Z
updated: 2026-03-21T00:00:00Z
---

## Current Focus

hypothesis: CONFIRMED - updatedAt set in both $set and $setOnInsert causes MongoDB ConflictingUpdateOperators on upsert insert path
test: typecheck and lint pass after fix
expecting: no new errors introduced
next_action: await human verification that PATCH endpoint works

## Symptoms

expected: Saving quote settings should succeed (PATCH /v1/business/:id/quote-settings)
actual: 500 Internal Server Error — MongoDB ConflictingUpdateOperators on 'updatedAt' path
errors: {"errorResponse":{"ok":0,"errmsg":"Updating the path 'updatedAt' would create a conflict at 'updatedAt'","code":40,"codeName":"ConflictingUpdateOperators"}}
reproduction: Save quote settings via the UI
started: Likely introduced by recent Phase 18 or 19 changes

## Eliminated

## Evidence

- timestamp: 2026-03-21
  checked: quote-settings.repository.ts upsert method (lines 38-54)
  found: $set contains ...updateAuditFields() which returns { updatedAt }, and $setOnInsert contains ...createAuditFields() which returns { createdAt, updatedAt }. On insert path of upsert, MongoDB applies BOTH operators, and updatedAt appears in both $set and $setOnInsert.
  implication: This is the direct cause of the ConflictingUpdateOperators error. MongoDB cannot set the same field in both $set and $setOnInsert.

- timestamp: 2026-03-21
  checked: update-audit-fields.utility.ts and create-audit-fields.utility.ts
  found: updateAuditFields() returns { updatedAt: new Date() }, createAuditFields() returns { createdAt: now, updatedAt: now }
  implication: Confirms the overlap — both utilities produce updatedAt

- timestamp: 2026-03-21
  checked: typecheck and lint after fix
  found: typecheck passes clean, no new lint errors in quote-settings module
  implication: fix is type-safe

## Resolution

root_cause: In QuoteSettingsRepository.upsert(), the $set operator includes updatedAt (via updateAuditFields()) and the $setOnInsert operator also includes updatedAt (via createAuditFields()). On the insert path of an upsert, MongoDB applies both operators simultaneously, and having the same field in both $set and $setOnInsert triggers ConflictingUpdateOperators error code 40.
fix: Replaced ...createAuditFields() in $setOnInsert with just { createdAt: new Date() }, so updatedAt is only set via $set (through updateAuditFields()). Removed unused createAuditFields import.
verification: TypeScript typecheck and lint pass. No quote-settings tests exist to run.
files_changed: [trade-flow-api/src/quote-settings/repositories/quote-settings.repository.ts]
