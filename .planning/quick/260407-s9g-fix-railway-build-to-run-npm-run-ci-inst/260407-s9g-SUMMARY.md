# Quick Task 260407-s9g: Summary

## Task
Fix Railway build to run `npm run ci` instead of `npm ci` before deployment

## Changes

### trade-flow-api/Dockerfile
- Added `RUN npm run ci` after `COPY . .` and before `RUN npm run build` in the builder stage
- This runs all quality gates (tests, lint, format, typecheck) during Docker build
- If any check fails, the Docker build fails and Railway rejects the deployment

### trade-flow-ui/nixpacks.toml (new file)
- Created Nixpacks configuration to override the default build phase
- Build phase now runs `npm run ci` before `npm run build`
- Nixpacks handles the install phase (`npm ci`) automatically

## Commits
- `9642daa` (trade-flow-api): chore(quick-260407-s9g): add CI quality gate to Railway API Dockerfile
- `d3a8f6f` (trade-flow-ui): chore(quick-260407-s9g): add CI quality gate to Railway UI via nixpacks.toml

## Outcome
Both Railway deployments now enforce the CI gate policy — failing tests, lint, formatting, or typecheck errors will block deployment at build time.
