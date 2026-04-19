---
status: diagnosed
phase: 54-user-management
source: [54-01-SUMMARY.md, 54-02-SUMMARY.md, 54-03-SUMMARY.md, 54-04-SUMMARY.md]
started: 2026-04-19T08:00:00Z
updated: 2026-04-19T09:00:00Z
---

## Current Test

[testing complete]

## Tests

### 1. Support Users Page Loads with Real Data
expected: Navigate to /support/users. The page loads showing a table of users fetched from the API. Each row displays Name, Email, Subscription status badge, and Created date. Loading state shows skeleton rows while data is fetching.
result: issue
reported: "When I login as a support user it takes me to the /onboarding page instead of taking me directly to the support page."
severity: major

### 2. Search Users by Name or Email
expected: Type a user's name or email into the search box. After a brief debounce (~300ms), the table filters to show only matching users. Clearing the search restores the full list.
result: pass

### 2b. Support Navigation Highlight
expected: Selecting "Users" in the support side menu highlights only the Users item.
result: issue
reported: "When I go to select the users navigation in the side menu, the dashboard remains selected and the users selection also becomes selected."
severity: cosmetic

### 3. Filter Users by Role
expected: Click the "Support" tab — only support users appear. Click "Customer" — only customer users appear. Click "All" — all users appear. Switching tabs resets to page 1.
result: pass

### 4. Filter Users by Subscription Status
expected: Use the subscription status dropdown to select "Active", "Trialing", "Past Due", "Canceled", or "Expired". The table filters to show only users with that subscription status. Selecting "All statuses" clears the filter.
result: issue
reported: "Trialing customers are not showing under the trialing section, canceled doesn't show under the canceled section. They all show as 'no subscription'."
severity: major

### 5. Pagination Controls
expected: If more users than the page limit exist, Previous/Next buttons and "Page X of Y" indicator appear below the table. Clicking Next advances to the next page. Clicking Previous goes back. Buttons are disabled at boundaries. Pagination hides when there is only 1 page.
result: pass

### 6. Subscription Status Badges
expected: Each user row shows a colored badge for their subscription status. "Trialing" shows a trial-styled badge, "Active" shows active-styled, "Past Due"/"Expired" show warning-styled, "Canceled" shows secondary-styled. Badges use semantic colors, not raw color values.
result: blocked
blocked_by: prior-phase
reason: "Cannot test other statuses because they always show 'No subscription' — depends on subscription status issue from Test 4."

### 7. Click User Row Navigates to Detail
expected: Clicking on a user row navigates the browser to /support/users/:id (where :id is that user's ID). The row has a pointer cursor and hover highlight on mouseover.
result: issue
reported: "Navigation works, but detail page shows 'No business associated' and 'No subscription' for all users even if they do have a business or subscription."
severity: major

### 8. Error State Display
expected: If the API is unreachable or returns an error, the page shows an alert with "Unable to load users. Please try again." and a "Try Again" button. Clicking "Try Again" refetches the data.
result: pass

### 9. Dashboard Metrics API Returns Membership Counts
expected: GET /v1/support/dashboard/metrics (with a valid support user JWT) returns a response containing totalUsers, activeTrials, activeSubscriptions, expiredSubscriptions, and canceledSubscriptions counts. The endpoint requires manage_users permission.
result: issue
reported: "The counts are all 0 except for the total users."
severity: major

### 10. User Detail API Returns User with Firebase Metadata
expected: GET /v1/support/users/:id (with a valid support user JWT) returns the user's email, name, supportRoleIds, supportRoleNames, subscription details, business details, and firebaseMetadata (creationTime, lastSignInTime). Returns 404 for invalid IDs.
result: issue
reported: "When I try to open the page for a support user I get 'Cannot read properties of null (reading replace)' TypeError in RoleBadge.tsx:5:32. The firebaseMetadata is always null for both creationTime and lastSignInTime."
severity: blocker

## Summary

total: 11
passed: 4
issues: 6
pending: 0
skipped: 0
blocked: 1

## Gaps

- truth: "Support user login should navigate to /support, not /onboarding"
  status: failed
  reason: "User reported: When I login as a support user it takes me to the /onboarding page instead of taking me directly to the support page."
  severity: major
  test: 1
  root_cause: "OnboardingGuard lacks support user bypass — checks hasDisplayName && hasBusiness only. Support users with no business always fail and redirect to /onboarding. Secondary: LoginPage savedLocation check runs before support role check, overriding role-based redirect."
  artifacts:
    - path: "trade-flow-ui/src/features/auth/components/OnboardingGuard.tsx"
      issue: "Missing support user bypass (PaywallGuard has one but OnboardingGuard does not)"
    - path: "trade-flow-ui/src/pages/LoginPage.tsx"
      issue: "savedLocation takes precedence over role-based redirect for support users"
  missing:
    - "Add support role bypass to OnboardingGuard matching PaywallGuard pattern"
    - "In LoginPage, ignore savedLocation when user has support roles"
  debug_session: ".planning/debug/support-login-redirect.md"

- truth: "Support nav menu should highlight only the active item"
  status: failed
  reason: "User reported: When I go to select the users navigation in the side menu, the dashboard remains selected and the users selection also becomes selected."
  severity: cosmetic
  test: 2b
  root_cause: "NavLink uses default prefix matching. Dashboard href '/support' is a prefix of '/support/users', so both match. NavLink missing 'end' prop for exact matching."
  artifacts:
    - path: "trade-flow-ui/src/components/layouts/DashboardLayout.tsx"
      issue: "NavLink missing 'end' prop for exact-match routes"
    - path: "trade-flow-ui/src/config/navigation.ts"
      issue: "Dashboard href '/support' is prefix of all support sub-routes"
  missing:
    - "Add 'end' prop to NavLink for the Dashboard item (or all nav items)"
  debug_session: ".planning/debug/nav-menu-highlight.md"

- truth: "Subscription status filter should show users with matching subscription status (trialing, canceled, etc.)"
  status: failed
  reason: "User reported: Trialing customers are not showing under the trialing section, canceled doesn't show under the canceled section. They all show as 'no subscription'."
  severity: major
  test: 4
  root_cause: "$lookup joins user._id (ObjectId converted to string via $toString) with subscription.userId, but subscription.userId stores the Firebase externalAuthUserId, not the MongoDB ObjectId string. Join never matches."
  artifacts:
    - path: "trade-flow-api/src/support/repositories/support-user.repository.ts"
      issue: "$lookup let uses { $toString: '$_id' } instead of '$externalAuthUserId' — lines 72-75"
  missing:
    - "Change $lookup let variable from { odUserId: { $toString: '$_id' } } to { odUserId: '$externalAuthUserId' }"
  debug_session: ".planning/debug/subscription-status-broken.md"

- truth: "User detail page should display business and subscription data for users who have them"
  status: failed
  reason: "User reported: Detail page shows 'No business associated' and 'No subscription' for all users even if they do have a business or subscription."
  severity: major
  test: 7
  root_cause: "Two $lookup failures: (1) Subscription $lookup uses wrong identifier — $toString of _id instead of $externalAuthUserId. (2) Business $lookup references non-existent $businessIds field on user document — businessIds is a derived value populated by UserRetriever service, not stored in MongoDB. Needs to go through businessusers join collection."
  artifacts:
    - path: "trade-flow-api/src/support/repositories/support-user.repository.ts"
      issue: "findByIdWithDetails: subscription $lookup uses wrong join key; business $lookup references non-existent $businessIds field"
  missing:
    - "Fix subscription $lookup to use $externalAuthUserId"
    - "Replace business $lookup with two-step: first $lookup businessusers, then $lookup businesses"
  debug_session: ".planning/debug/detail-page-missing-data.md"

- truth: "Dashboard metrics should return accurate subscription counts (activeTrials, activeSubscriptions, etc.)"
  status: failed
  reason: "User reported: The counts are all 0 except for the total users."
  severity: major
  test: 9
  root_cause: "Same $lookup join mismatch as Tests 4 and 7 — $toString of user._id vs subscription.userId (Firebase UID). totalUsers works because it counts user docs via $sum:1 after $unwind with preserveNullAndEmptyArrays, not dependent on subscription join."
  artifacts:
    - path: "trade-flow-api/src/support/services/support-dashboard-metrics.service.ts"
      issue: "$lookup let uses { $toString: '$_id' } instead of '$externalAuthUserId' — line 36"
  missing:
    - "Change $lookup let variable to use $externalAuthUserId"
  debug_session: ".planning/debug/dashboard-metrics-zeros.md"

- truth: "User detail page should render without errors, including for support users; firebaseMetadata should return real creationTime and lastSignInTime"
  status: failed
  reason: "User reported: TypeError in RoleBadge.tsx:5:32 — Cannot read properties of null (reading 'replace'). firebaseMetadata always null."
  severity: blocker
  test: 10
  root_cause: "RoleBadge crash: $lookup projects { name: 1 } but SupportRoleEntity field is 'roleName'. MongoDB returns docs with only _id, mapping produces [undefined], RoleBadge calls undefined.replace() and crashes. Firebase null: FirebaseAuthMetadataService swallows all errors silently — likely missing/invalid FIREBASE_CLIENT_EMAIL and FIREBASE_PRIVATE_KEY env vars."
  artifacts:
    - path: "trade-flow-api/src/support/repositories/support-user.repository.ts"
      issue: "$lookup projection uses 'name' instead of 'roleName' (line 159); mapping uses r.name instead of r.roleName (line 214)"
    - path: "trade-flow-ui/src/features/support/components/RoleBadge.tsx"
      issue: "No null guard on roleName.replace() call"
    - path: "trade-flow-api/src/support/services/firebase-auth-metadata.service.ts"
      issue: "Catches all errors silently, returns nulls — Firebase Admin SDK may not be initialized correctly"
  missing:
    - "Fix $lookup projection to use 'roleName' and update mapping"
    - "Add null guard in RoleBadge as defense-in-depth"
    - "Verify Firebase Admin SDK env vars are configured; consider surfacing init errors"
  debug_session: ".planning/debug/rolebadge-crash-firebase-null.md"
