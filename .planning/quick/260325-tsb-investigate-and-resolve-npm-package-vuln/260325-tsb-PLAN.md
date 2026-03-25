---
phase: quick
plan: 260325-tsb
type: execute
wave: 1
depends_on: []
files_modified:
  - /Users/mattc/Documents/projects/agent/trade-flow-api/package.json
  - /Users/mattc/Documents/projects/agent/trade-flow-api/package-lock.json
  - /Users/mattc/Documents/projects/agent/trade-flow-ui/package.json
  - /Users/mattc/Documents/projects/agent/trade-flow-ui/package-lock.json
autonomous: true
must_haves:
  truths:
    - "npm audit in trade-flow-api reports no high/critical vulnerabilities"
    - "npm audit in trade-flow-ui reports no high/critical vulnerabilities"
    - "Both projects build and pass tests after dependency updates"
  artifacts:
    - path: "trade-flow-api/package-lock.json"
      provides: "Updated dependency tree with patched versions"
    - path: "trade-flow-ui/package-lock.json"
      provides: "Updated dependency tree with patched versions"
  key_links: []
---

<objective>
Investigate and resolve npm package vulnerabilities in both trade-flow-api and trade-flow-ui codebases.

Purpose: Eliminate known security vulnerabilities in project dependencies.
Output: Both repos with resolved or documented vulnerabilities; builds and tests still pass.
</objective>

<execution_context>
@$HOME/.claude/get-shit-done/workflows/execute-plan.md
@$HOME/.claude/get-shit-done/templates/summary.md
</execution_context>

<context>
@.planning/STATE.md
@/Users/mattc/Documents/projects/agent/trade-flow-api/package.json
@/Users/mattc/Documents/projects/agent/trade-flow-ui/package.json
</context>

<tasks>

<task type="auto">
  <name>Task 1: Audit and fix trade-flow-api vulnerabilities</name>
  <files>/Users/mattc/Documents/projects/agent/trade-flow-api/package.json, /Users/mattc/Documents/projects/agent/trade-flow-api/package-lock.json</files>
  <action>
    1. Run `npm audit` in `/Users/mattc/Documents/projects/agent/trade-flow-api/` to identify all vulnerabilities. Capture full output.
    2. Run `npm audit fix` to auto-resolve safe semver-compatible patches.
    3. If high/critical vulnerabilities remain, review each:
       - If `npm audit fix --force` would only bump minor/patch versions, apply it.
       - If a force fix would bump a major version, evaluate the breaking change. For NestJS ecosystem packages (@nestjs/*), do NOT force-bump major versions -- document the vulnerability instead.
       - For non-critical packages where major bumps are safe (test-only deps, build tools), apply the fix.
    4. After fixes, run `npm run build` to confirm the project compiles.
    5. Run `npm test` to confirm tests pass.
    6. If any vulnerabilities cannot be safely resolved, document them (package name, severity, reason skipped) in the summary.
  </action>
  <verify>
    <automated>cd /Users/mattc/Documents/projects/agent/trade-flow-api && npm audit 2>&1 | tail -20 && npm run build 2>&1 | tail -5</automated>
  </verify>
  <done>trade-flow-api has zero high/critical vulnerabilities (or remaining ones are documented with justification). Build and tests pass.</done>
</task>

<task type="auto">
  <name>Task 2: Audit and fix trade-flow-ui vulnerabilities</name>
  <files>/Users/mattc/Documents/projects/agent/trade-flow-ui/package.json, /Users/mattc/Documents/projects/agent/trade-flow-ui/package-lock.json</files>
  <action>
    1. Run `npm audit` in `/Users/mattc/Documents/projects/agent/trade-flow-ui/` to identify all vulnerabilities. Capture full output.
    2. Run `npm audit fix` to auto-resolve safe semver-compatible patches.
    3. If high/critical vulnerabilities remain, review each:
       - If `npm audit fix --force` would only bump minor/patch versions, apply it.
       - If a force fix would bump a major version, evaluate the breaking change. For React/Vite ecosystem packages, do NOT force-bump major versions -- document the vulnerability instead.
       - For non-critical packages where major bumps are safe (test-only deps, build tools), apply the fix.
    4. After fixes, run `npm run build` to confirm the project compiles.
    5. If any vulnerabilities cannot be safely resolved, document them (package name, severity, reason skipped) in the summary.
  </action>
  <verify>
    <automated>cd /Users/mattc/Documents/projects/agent/trade-flow-ui && npm audit 2>&1 | tail -20 && npm run build 2>&1 | tail -5</automated>
  </verify>
  <done>trade-flow-ui has zero high/critical vulnerabilities (or remaining ones are documented with justification). Build passes.</done>
</task>

</tasks>

<verification>
- `npm audit` in both repos shows 0 high/critical vulnerabilities (or justified exceptions)
- `npm run build` succeeds in both repos
- `npm test` succeeds in trade-flow-api
</verification>

<success_criteria>
Both trade-flow-api and trade-flow-ui have resolved all auto-fixable vulnerabilities. Any remaining vulnerabilities are documented with severity and reason they cannot be safely resolved. Both projects build successfully after changes.
</success_criteria>

<output>
After completion, create `.planning/quick/260325-tsb-investigate-and-resolve-npm-package-vuln/260325-tsb-SUMMARY.md`
</output>
