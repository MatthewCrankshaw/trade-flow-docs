---
phase: quick
plan: 260325-tsb
subsystem: security
tags: [npm, audit, vulnerabilities, picomatch, ajv, dependencies]

requires: []
provides:
  - Zero npm audit vulnerabilities in both trade-flow-api and trade-flow-ui
affects: []

tech-stack:
  added: []
  patterns:
    - "npm overrides for transitive dependency vulnerability resolution"

key-files:
  created: []
  modified:
    - trade-flow-api/package.json
    - trade-flow-api/package-lock.json
    - trade-flow-ui/package-lock.json

key-decisions:
  - "Used npm overrides for picomatch and ajv in trade-flow-api instead of force-downgrading @nestjs/cli from v11 to v7"

patterns-established: []

requirements-completed: []

duration: 2min
completed: 2026-03-25
---

# Quick Task 260325-tsb: Investigate and Resolve npm Package Vulnerabilities

**Resolved all npm audit vulnerabilities in both repos: picomatch 4.0.4 override and ajv 8.18.0 override in API, picomatch patch in UI**

## Performance

- **Duration:** 2 min
- **Started:** 2026-03-25T21:27:59Z
- **Completed:** 2026-03-25T21:30:17Z
- **Tasks:** 2
- **Files modified:** 3

## Accomplishments
- Resolved 7 vulnerabilities (2 high, 5 moderate) in trade-flow-api via npm overrides
- Resolved 1 high vulnerability in trade-flow-ui via npm audit fix
- Both projects build successfully and trade-flow-api passes all 312 tests

## Vulnerability Details

### trade-flow-api (before fix)

| Package | Severity | Advisory | Resolution |
|---------|----------|----------|------------|
| picomatch 4.0.0-4.0.3 | high (x2) | GHSA-c2c7-rcm5-vvqj (ReDoS via extglob quantifiers) | Override to 4.0.4 |
| ajv 7.0.0-alpha.0-8.17.1 | moderate (x5) | GHSA-2g4f-4pwh-qvx6 (ReDoS with $data option) | Override to 8.18.0 |

Both vulnerabilities were in transitive dependencies of `@nestjs/cli` and `@nestjs/schematics` (devDependencies). The only auto-fix npm offered was downgrading `@nestjs/cli` from v11 to v7 -- a breaking major version change that would break the build tooling. Instead, npm overrides were used to force the patched versions of the transitive dependencies.

### trade-flow-ui (before fix)

| Package | Severity | Advisory | Resolution |
|---------|----------|----------|------------|
| picomatch 4.0.0-4.0.3 | high (x1) | GHSA-c2c7-rcm5-vvqj (ReDoS via extglob quantifiers) | npm audit fix (semver-compatible) |

This was a direct semver-compatible fix, no overrides or force needed.

## Task Commits

Each task was committed atomically in its respective repo:

1. **Task 1: Audit and fix trade-flow-api vulnerabilities** - `8db0d7a` in trade-flow-api (fix)
2. **Task 2: Audit and fix trade-flow-ui vulnerabilities** - `7fe865d` in trade-flow-ui (fix)

## Files Created/Modified
- `trade-flow-api/package.json` - Added overrides for picomatch and ajv
- `trade-flow-api/package-lock.json` - Updated dependency tree with patched versions
- `trade-flow-ui/package-lock.json` - Updated picomatch to patched version

## Decisions Made
- Used npm `overrides` field in trade-flow-api/package.json to force picomatch@4.0.4 and ajv@8.18.0 instead of downgrading @nestjs/cli from v11 to v7 (which would be a breaking change to the build tooling)

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered

None.

## User Setup Required

None - no external service configuration required.

## Known Stubs

None.

---
*Quick task: 260325-tsb*
*Completed: 2026-03-25*
