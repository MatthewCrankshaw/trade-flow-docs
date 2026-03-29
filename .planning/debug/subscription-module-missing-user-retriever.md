---
status: awaiting_human_verify
trigger: "Worker crashes at startup — JwtAuthGuard can't resolve UserRetriever inside SubscriptionModule context"
created: 2026-03-29T00:00:00.000Z
updated: 2026-03-29T00:00:00.000Z
---

## Current Focus

hypothesis: REVISED — Adding AuthModule to WorkerModule was necessary but insufficient. The real issue is that SubscriptionModule declares controllers using @UseGuards(JwtAuthGuard) but JwtAuthGuard is never exported from AuthModule, and SubscriptionModule doesn't import AuthModule. NestJS tries to resolve JwtAuthGuard in SubscriptionModule's context and fails because UserCreator/UserRetriever are not available there.
test: (1) Export JwtAuthGuard from AuthModule, (2) Import AuthModule in SubscriptionModule
expecting: Both API and worker start without UnknownDependenciesException
next_action: Apply fix to AuthModule and SubscriptionModule

## Symptoms

expected: trade-flow-worker starts successfully with SubscriptionModule loaded
actual: Server crashes immediately with UnknownDependenciesException — Nest can't resolve JwtAuthGuard's dependency on UserRetriever within SubscriptionModule
errors: |
  ERROR [ExceptionHandler] UnknownDependenciesException: Nest can't resolve dependencies of the JwtAuthGuard (UserCreator, ?).
  Please make sure that the argument UserRetriever at index [1] is available in the SubscriptionModule module.
  type: 'JwtAuthGuard'
  context.dependencies: [UserCreator, UserRetriever]
reproduction: Start trade-flow-worker container
started: Introduced in phase 30 execution

## Eliminated

(none)

## Evidence

- timestamp: 2026-03-29T00:01:00.000Z
  checked: JwtAuthGuard constructor (auth.guard.ts)
  found: Requires UserCreator and UserRetriever via constructor injection
  implication: Any module using JwtAuthGuard must have these providers available

- timestamp: 2026-03-29T00:02:00.000Z
  checked: AuthModule (auth.module.ts)
  found: Imports UserModule, registers JwtAuthGuard as provider, but does NOT export JwtAuthGuard
  implication: JwtAuthGuard is resolved per-module context; importing AuthModule alone won't export the guard

- timestamp: 2026-03-29T00:03:00.000Z
  checked: SubscriptionModule (subscription.module.ts)
  found: imports only [QueueModule]. Controllers use @UseGuards(JwtAuthGuard). No AuthModule or UserModule imported.
  implication: NestJS tries to instantiate JwtAuthGuard in SubscriptionModule context but can't find UserRetriever

- timestamp: 2026-03-29T00:04:00.000Z
  checked: AppModule (app.module.ts)
  found: Imports both AuthModule and SubscriptionModule as siblings. AuthModule + UserModule at top level make JwtAuthGuard deps available to sibling modules.
  implication: This is why it works in AppModule but fails in WorkerModule (which lacks AuthModule)

- timestamp: 2026-03-29T00:05:00.000Z
  checked: WorkerModule (worker.module.ts)
  found: Imports [ConfigModule, LoggerModule, CoreModule, QueueModule, SubscriptionModule] -- no AuthModule, no UserModule
  implication: Confirms root cause -- worker context has no access to UserCreator/UserRetriever

- timestamp: 2026-03-29T00:06:00.000Z
  checked: AuthModule exports
  found: AuthModule does NOT export JwtAuthGuard. It only registers it as a provider internally.
  implication: Fix must either (a) make AuthModule export JwtAuthGuard and import AuthModule in SubscriptionModule, or (b) import both AuthModule and UserModule in SubscriptionModule so JwtAuthGuard can be instantiated

- timestamp: 2026-03-29T00:10:00.000Z
  checked: Human verification of WorkerModule fix
  found: Same error persists on BOTH API and worker. Error says "SubscriptionModule module" not "WorkerModule". Adding AuthModule to WorkerModule imports did not fix the issue.
  implication: The problem is at the SubscriptionModule level, not the parent module level.

- timestamp: 2026-03-29T00:11:00.000Z
  checked: All modules using JwtAuthGuard — none import AuthModule directly
  found: CustomerModule, BusinessModule, JobModule etc. all use @UseGuards(JwtAuthGuard) without importing AuthModule. None of them fail. AuthModule does NOT export JwtAuthGuard.
  implication: NestJS resolves guards via enhancer resolution chain (module -> host -> root). Other modules work because their guard is resolved from root (AppModule) injector. SubscriptionModule may be triggering a different code path.

- timestamp: 2026-03-29T00:12:00.000Z
  checked: AuthModule exports
  found: AuthModule has providers [JwtStrategy, JwtAuthGuard, FirebaseKeyProvider] but exports NOTHING. This means JwtAuthGuard is a module-scoped provider.
  implication: The fix must export JwtAuthGuard from AuthModule AND have SubscriptionModule import AuthModule so the guard and its deps are available in SubscriptionModule's context.

## Resolution

root_cause: SubscriptionModule declares controllers using @UseGuards(JwtAuthGuard) but neither imports AuthModule nor has JwtAuthGuard available as a provider. AuthModule registers JwtAuthGuard as a provider but never exports it, making it module-scoped. NestJS tries to instantiate JwtAuthGuard in SubscriptionModule's DI context and fails because UserCreator/UserRetriever (JwtAuthGuard's constructor deps from UserModule) are not available there. Other modules (Customer, Business, etc.) happened to work due to NestJS's enhancer resolution walking up to the root AppModule injector, but SubscriptionModule (being newly added) triggers the error.
fix: (1) Export JwtAuthGuard from AuthModule so consuming modules can access it. (2) Import AuthModule in SubscriptionModule so JwtAuthGuard and its dependency chain (UserModule -> UserCreator, UserRetriever) are available in SubscriptionModule's context. WorkerModule already has AuthModule imported from the previous fix attempt.
verification: TypeScript compilation passes. Awaiting human verification that both API and worker start without crash.
files_changed:
  - trade-flow-api/src/auth/auth.module.ts
  - trade-flow-api/src/subscription/subscription.module.ts
  - trade-flow-api/src/worker/worker.module.ts
