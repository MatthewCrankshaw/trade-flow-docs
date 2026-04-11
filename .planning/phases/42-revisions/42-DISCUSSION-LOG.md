# Phase 42: Revisions - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-11
**Phase:** 42-revisions
**Areas discussed:** Revise endpoint shape, Chain identity & index, Concurrency & 409, Follow-up cancel hook

---

## Gray Area Selection

| Option | Description | Selected |
|--------|-------------|----------|
| Revise endpoint shape | Clone-then-PATCH vs one-shot body vs combined revise+resend; allowed-from statuses; line-item copy semantics | ✓ |
| Chain identity & index | rootEstimateId write policy, partial unique interaction with Phase 41's {businessId, number} index | ✓ |
| Concurrency & 409 | Operation order, 409 surface, status guard in atomic filter | ✓ |
| Follow-up cancel hook | Interface + no-op default vs placeholder vs full defer | ✓ |

**User's choice:** All four areas discussed.

---

## Revise Endpoint Shape

### Q1: Birth status + flow shape

| Option | Description | Selected |
|--------|-------------|----------|
| Clone → Draft, then PATCH + Send | Endpoint clones current revision as DRAFT with isCurrent=true. Trader then uses existing PATCH /v1/estimates/:id and later Phase 44's /send. Maximum symmetry with initial create–edit–send flow. | ✓ |
| One-shot body + born Sent | Endpoint accepts full new snapshot in body, creates the new row with status=SENT directly. No separate PATCH. | |
| Two-step: clone → Draft, trader edits inline, then explicit /send | Clone creates Draft; trader PATCHes until ready; trader calls /send. Two rows live simultaneously during editing. | |
| Combined revise+resend single call | POST body carries new snapshot AND triggers Phase 44's send flow. Atomic. Phase 42 and 44 tightly coupled. | |

**User's choice:** Clone → Draft, then PATCH + Send (D-REV-01).

### Q2: Old revision state during Draft editing window

| Option | Description | Selected |
|--------|-------------|----------|
| Old stays Sent, isCurrent=false | Only isCurrent flips. Status history preserved. Detail resolves by root → returns latest (the new Draft). | ✓ |
| Old stays Sent, isCurrent=false, but detail-by-root hides Draft if no other changes | Tries to avoid surfacing unfinished revision. Too clever. | |
| Old moves to a new SUPERSEDED status | Adds a value to EstimateStatus enum and every transition map row. More surface area. | |

**User's choice:** Old stays Sent, isCurrent=false (D-REV-04). Status lifecycle unchanged.

### Q3: Draft abandon / rollback rules

| Option | Description | Selected |
|--------|-------------|----------|
| Trader must DELETE the Draft to roll back | DELETE on the new Draft: sets deletedAt + status=DELETED on the new row, atomically flips isCurrent back to the predecessor. No auto-cleanup. | ✓ |
| Creating a second revision auto-discards any Draft-only predecessor | More forgiving, more code. | |
| Drafts persist forever | History panel filters them out for the customer-facing view. Simplest but pollutes the audit trail. | |

**User's choice:** Trader must DELETE the Draft (D-REV-05). EstimateDeleter extended to restore predecessor's isCurrent atomically.

### Q4: Allowed source statuses

| Option | Description | Selected |
|--------|-------------|----------|
| SENT only (strict, literal REV-02) | Only SENT with isCurrent=true can be revised. Once customer views or responds, revising is blocked. | |
| SENT, VIEWED, RESPONDED, SITE_VISIT_REQUESTED | Any non-terminal, non-Draft status with isCurrent=true. Blocks DRAFT, CONVERTED, DECLINED, EXPIRED, LOST, DELETED. | ✓ |
| Any non-terminal incl. DRAFT | Drafts should just be edited in place. Feels wrong. | |

**User's choice:** SENT, VIEWED, RESPONDED, SITE_VISIT_REQUESTED (D-REV-02).

### Q5: Line-item copy semantics

| Option | Description | Selected |
|--------|-------------|----------|
| Verbatim copy, status=PENDING on new rows | Clone every non-deleted line item; new _id, estimateId=new; pricing fields copied exactly; status reset to PENDING; bundle parent-child remapped. | ✓ |
| Verbatim copy, status preserved | Keep APPROVED/REJECTED. Arguably misleading since customer hasn't seen the new revision yet. | |
| Skip DELETED and REJECTED line items | Cleaner new revision but loses audit trail. | |

**User's choice:** Verbatim copy, status=PENDING (D-LI-01, D-LI-02, D-LI-03).

---

## Chain Identity & Index

### Q1: rootEstimateId write policy on root rows

| Option | Description | Selected |
|--------|-------------|----------|
| Root row has rootEstimateId=null; revisions point at the root _id | Matches Phase 41's current null default. Queries awkward. | ✓ (initially, then flipped — see conflict resolution) |
| Root row rootEstimateId=self._id (post-insert update) | Every row in a chain has rootEstimateId set consistently. | ✓ (after conflict resolution) |
| Drop rootEstimateId entirely; always walk parentEstimateId chain | Clunky with partial unique indexes. | |

**User's initial choice:** rootEstimateId=null on root. Flipped after the null-collision conflict was surfaced — final answer: **rootEstimateId=self._id on root** (D-CHAIN-01, D-CHAIN-03, D-CHAIN-04).

### Q2: parentEstimateId vs rootEstimateId semantics

| Option | Description | Selected |
|--------|-------------|----------|
| parentEstimateId = immediately previous revision's _id; rootEstimateId = chain origin | Linked-list predecessor + denormalised root. Both present on every row. | ✓ |
| parentEstimateId === rootEstimateId always (flat) | Loses "immediate predecessor" info. | |
| Drop parentEstimateId, keep only rootEstimateId + revisionNumber | Requires removing a Phase 41-reserved field. | |

**User's choice:** parentEstimateId = immediately previous; rootEstimateId = chain origin (D-CHAIN-02).

### Q3: Phase 41 {businessId, number} index rework

| Option | Description | Selected |
|--------|-------------|----------|
| Extend partial filter to {deletedAt: null, isCurrent: true} | Only current, non-deleted row enforces uniqueness. Historical revisions can coexist. | ✓ |
| Drop {businessId, number} uniqueness entirely; rely on counter atomicity | Simpler but loses defence-in-depth. | |
| Add {businessId, number, revisionNumber} unique partial | Does not give "exactly one current per chain" guarantee. | |

**User's choice:** Extend partial filter to {deletedAt: null, isCurrent: true} (D-CHAIN-06).

### Q4: Partial unique index for "one current per chain"

| Option | Description | Selected |
|--------|-------------|----------|
| {rootEstimateId: 1, isCurrent: 1} unique, partial on {isCurrent: true} | Matches SC #1 phrasing. Clean mental model. | ✓ |
| {parentEstimateId: 1, isCurrent: 1} unique, partial on {isCurrent: true} | Research §3 phrasing. Requires parentEstimateId on root rows to equal self (awkward). | |

**User's choice:** {rootEstimateId: 1, isCurrent: 1} partial unique (D-CHAIN-05).

### Q5: Null-collision conflict resolution

Conflict surfaced between Q1 answer (rootEstimateId=null on root) and Q4 answer (partial unique on {rootEstimateId, isCurrent}). Multiple root rows would all hash to (null, true) and collide.

| Option | Description | Selected |
|--------|-------------|----------|
| Flip root policy: rootEstimateId = self._id on root | Insert, then immediate update. Partial unique index works as written. Phase 42 retrofits EstimateCreator and optionally backfills. | ✓ |
| Keep rootEstimateId=null on root, narrow partial filter with $type check | Partial index excludes root rows. Root row isCurrent=true unguarded by this index. | |
| Keep rootEstimateId=null AND use parentEstimateId for unique index | Same null-exclusion trick via parentEstimateId. Loses "rootEstimateId is chain identity" mental model. | |

**User's choice:** Flip root policy to rootEstimateId=self._id. Fold retrofit into Phase 41 planning if possible; Phase 42 backfill as fallback (D-CHAIN-03, D-CHAIN-04).

---

## Concurrency & 409

### Q1: Write order (downgrade vs insert first)

| Option | Description | Selected |
|--------|-------------|----------|
| Insert new first, then downgrade old | Incompatible with partial unique index without transactions. Rejected. | |
| Downgrade old first, then insert new | Atomic findOneAndUpdate with status filter; compensating rollback on insert failure. Brief zero-current window. | ✓ |
| Single findOneAndUpdate + explicit try/finally | Same as option B but with rollback path spelled out. Equivalent. | |

**User's choice:** Downgrade old first, then insert new (D-CONC-01, D-CONC-05).

### Q2: 409 surface

| Option | Description | Selected |
|--------|-------------|----------|
| Dedicated ErrorCodes.ESTIMATE_REVISION_CONFLICT → HTTP 409 | New error code, new ConflictError class mapped via createHttpError. | ✓ |
| Reuse InvalidRequestError → HTTP 422 | Violates SC #4 which literally says 409 Conflict. | |
| Silent retry once, then 409 | Masks client bugs. | |

**User's choice:** Dedicated 409 with new ConflictError class (D-CONC-07, D-CONC-08).

### Q3: Status guard placement

| Option | Description | Selected |
|--------|-------------|----------|
| Yes — filter includes status $in [SENT, VIEWED, RESPONDED, SITE_VISIT_REQUESTED] | Atomic. Race-free. | ✓ |
| No — service reads status, validates, then updates | Race window. Violates SC #4 spirit. | |

**User's choice:** Status guard in the atomic findOneAndUpdate filter (D-CONC-02, D-CONC-03).

---

## Follow-up Cancel Hook

### Q1: Interface vs placeholder vs defer

| Option | Description | Selected |
|--------|-------------|----------|
| Interface + no-op default, Phase 46 replaces binding | IEstimateFollowupCanceller with NoopEstimateFollowupCanceller default. Phase 46 rebinds the DI token. | ✓ |
| Empty placeholder method on EstimateReviser itself | Phase 42 and 46 touch the same file. | |
| Defer entirely to Phase 46 | SC #3 not met end-to-end in Phase 42. | |

**User's choice:** Interface + no-op default (D-HOOK-01, D-HOOK-02).

### Q2: Interface location and DI wiring

| Option | Description | Selected |
|--------|-------------|----------|
| Interface in @estimate/services/; string token ESTIMATE_FOLLOWUP_CANCELLER | Co-located with other estimate services. Standard NestJS DI rebinding via useExisting/useClass. | ✓ |
| Interface under @core/interfaces/ | Overkill for one consumer. | |
| No interface — direct class injection, Phase 46 subclasses | Loses interface benefit. | |

**User's choice:** Interface in @estimate/services/ with string token (D-HOOK-02).

### Q3: When does cancellation happen?

| Option | Description | Selected |
|--------|-------------|----------|
| Nothing — Draft revisions never have follow-ups scheduled | Cancellation happens at SEND time (Phase 44) on the NEW revision. Rollback-via-DELETE leaves old revision's follow-ups on original schedule. | ✓ |
| Cancel at POST /revisions time; no un-cancel on Draft delete | Matches SC #3 literal wording but loses follow-up schedule on rollback. | |
| Cancel at POST + reschedule on Draft delete | Complex and arguably wrong. | |

**User's choice:** Nothing at POST time; Phase 44 cancels on send (D-HOOK-03, D-HOOK-04). Triggers SC #3 amendment captured in D-HOOK-05.

---

## Claude's Discretion

Deferred to planner (per D-DISCRETION in CONTEXT.md):
- Service naming (`EstimateReviser` expected but flexible)
- Controller placement of new routes
- Line-item clone helper location (service vs repository private method)
- Bootstrap ordering for index drop-and-recreate
- Exact `ConflictError` base class shape (must match existing error hierarchy)
- Mock generator extensions
- Concurrent-revise integration test design (must assert exactly one success, one 409, one current row)

## Deferred Ideas

See `<deferred>` section in 42-CONTEXT.md for the full list. Notable:

- `SUPERSEDED` enum value rejected (isCurrent bit carries the meaning)
- Combined revise+resend endpoint rejected
- Revising a Draft rejected
- Trimmed revision summary DTO rejected (full DTOs in history)
- Silent 409 retry rejected
- MongoDB transactions ruled out (codebase convention)
- SC #3 amendment is MANDATORY, not deferred
- Auto-cleanup of abandoned Drafts rejected
