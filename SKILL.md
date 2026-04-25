---
name: factory-di
description: >
  Swift dependency injection with hmlongco/Factory v2.5+. TRIGGER when code imports
  Factory, FactoryKit, or FactoryTesting; uses @Injected, @LazyInjected, @WeakLazyInjected,
  @DynamicInjected, @InjectedObject, @InjectedObservable; extends Container, SharedContainer,
  or ManagedContainer; references ContainerManager, Scope, ParameterFactory, ContainerTrait,
  FactoryContext, promised(), scopeOnParameters, .cached/.singleton/.shared/.graph;
  or discusses Factory DI, scopes, singleton tests, @Observable, or Swift Testing isolation.
  DO NOT TRIGGER for Swinject, Needle, Hilt/Dagger/Koin, or general Swift.
---

# Factory DI (hmlongco/Factory)

Factory v2.5+ | Swift 5.10+ | iOS 13+ | `import FactoryKit`

## CRITICAL: What LLMs Get Wrong

### 1. Import `FactoryKit`, not `Factory`

`Factory` is the legacy module kept for backward compatibility. It causes duplicate-module issues and breaks parallel testing. **Always use `import FactoryKit`** for new code. Exception: don't change an existing working `import Factory` unless migrating.

```swift
// CORRECT
import FactoryKit

// LEGACY — avoid for new projects
import Factory
```

### 2. `@MainActor` needed in TWO places

When a factory produces a `@MainActor`-isolated type, annotate BOTH the computed property AND the closure:

```swift
// CORRECT
@MainActor
var viewModel: Factory<ContentViewModel> {
    self { @MainActor in ContentViewModel() }
}

// WRONG — compiles but causes isolation errors
var viewModel: Factory<ContentViewModel> {
    self { ContentViewModel() }  // missing @MainActor
}
```

Same for `register`: `container.viewModel.register { @MainActor in MockVM() }`

### 3. `reset()` is destructive

Bare `reset()` uses `.all` — clears registrations, caches, defaultScope, options, and re-triggers autoRegister. Always specify options:

```swift
container.reset(options: .scope)         // clear caches only
container.reset(options: .registration)  // restore original factories only
container.reset()                        // NUCLEAR — clears everything
```

### 4. Singletons are NOT per-container

`push()`/`pop()` and `container.reset()` do NOT affect singletons. Singletons have their own global cache.

```swift
Scope.singleton.reset()                              // reset ALL singletons
container.mySingleton.reset(options: .scope)          // reset one singleton
container.mySingleton.register { MockSingleton() }    // also clears singleton cache
```

### 5. `@ObservationIgnored` + `@Observable` migration

In `@Observable` classes, `@Injected` properties MUST have `@ObservationIgnored`:

```swift
@Observable class MyViewModel {
    @ObservationIgnored @Injected(\.service) private var service  // CORRECT
    var data: [String] = []  // observed — no annotation needed
}
```

Also: do NOT use `@InjectedObject` with `@Observable`. Use `@InjectedObservable` instead.

### 6. `FactoryTesting` is test-target only — and don't confuse imports with dependencies

`FactoryTesting` is guarded by `#if DEBUG` and `#if swift(>=6.1)`. Never add it to app targets.

**Import vs dependency — these are different things:**

```swift
// Package.swift — target DEPENDENCIES (what SPM links)
.testTarget(
    name: "MyAppTests",
    dependencies: ["FactoryTesting"]  // YES
    // Do NOT add "FactoryKit" here — duplicates factories
)

// Test source file — IMPORTS (what Swift sees)
import FactoryTesting  // YES — provides ContainerTrait
import FactoryKit      // OK in source files — just don't add as target dependency above
```

Adding `FactoryKit` as a *target dependency* alongside your app creates duplicate factory registrations. Importing it in *source files* is fine — your test target already gets it transitively through your app.

### 7. Context/`onTest` can override later registrations

The "Factory wins" problem: inline modifiers are reapplied on every computed-property access. A later `.register` can be silently overridden by the inline context.

```swift
// The .onTest re-fires every time the computed property is accessed
var myService: Factory<MyServiceProtocol> {
    self { RealService() }.onTest { TestDefault() }
}
// This registration may be "lost" on next resolve:
Container.shared.myService.register { SpecialMock() }
```

**Fix**: Use `.once()` modifier or define contexts in `autoRegister`. See [references/wiring.md](references/wiring.md).

### 8. `autoRegister` won't clear singleton caches

By design — auto-registration runs on every container creation. Clearing singletons each time would defeat their purpose. Outside of `autoRegister`, `.register` DOES clear singleton caches.

### 9. Factory closures in Swift 6 are `@Sendable @isolated(any)`

Not just `@Sendable`. The actual typealias: `@Sendable @isolated(any) () -> T`. This affects storing or passing factory closures.

### 10. `ParameterFactory` cannot use `@Injected` wrappers

No way to supply parameters at wrapper init time. Resolve directly:
```swift
let service = Container.shared.parameterService(42)
```
Also: scoped ParameterFactory caches the first argument. Use `.scopeOnParameters` for per-argument caching (parameter must be `Hashable`).

## Basic Templates

```swift
// Define
extension Container {
    var myService: Factory<MyServiceProtocol> {
        self { MyService() }
    }
}

// Inject in a class
class MyViewModel {
    @Injected(\.myService) private var service
}

// Mock in test
Container.shared.myService.register { MockService() }
```

## Routing: When to Read Each Reference

- **Defining factories, resolution, scopes, wrappers, ParameterFactory, custom containers, reset** → [references/patterns.md](references/patterns.md)
- **Testing, mocking, ContainerTrait, parallel tests, singletons in tests** → [references/testing.md](references/testing.md)
- **SwiftUI, @Observable, @InjectedObject, previews, @MainActor** → [references/swiftui.md](references/swiftui.md)
- **Multi-module, AutoRegistering, contexts, `.once()`, decorators, debugging** → [references/wiring.md](references/wiring.md)

## Property Wrapper Quick Reference

| Wrapper | Resolves | Type | Key Caveat |
|---------|----------|------|------------|
| `@Injected` | Init | `T` | Default choice for classes |
| `@LazyInjected` | First access | `T` | Mutating getter — avoid in structs |
| `@WeakLazyInjected` | First access | `T?` | Weak ref; for delegate/parent-child |
| `@DynamicInjected` | Every access | `T` | Always fresh; pair with scope if stateful |
| `@InjectedObject` | Init | `T: ObservableObject` | SwiftUI only; uses StateObject; iOS 14+ |
| `@InjectedObservable` | First access | `T: Observable` | SwiftUI only; uses State; iOS 17+ |

## Scope Quick Reference

| Scope | Lifetime | Managed By | Notes |
|-------|----------|-----------|-------|
| `.unique` | None | N/A | Default; new instance every time |
| `.cached` | Until reset | Container | `container.reset(options: .scope)` |
| `.shared` | While strong refs | Container (weak) | Auto-releases when unreferenced |
| `.singleton` | Global | `Scope.singleton` | NOT per-container; `Scope.singleton.reset()` |
| `.graph` | Resolution cycle | Graph scope | Reuses within one resolve chain |

## Decision Trees

### How to inject?

1. SwiftUI View resolves ViewModel from Factory? → `@InjectedObject` (ObservableObject) or `@InjectedObservable` (@Observable). If the VM manages its own deps via `@Injected`, direct `@StateObject`/`@State` ownership is also valid.
2. Class needs service at init? → `@Injected(\.service)` or constructor injection
3. Need lazy/deferred resolution? → `@LazyInjected(\.service)`
4. Need always-fresh instance? → `@DynamicInjected(\.service)`
5. Need to pass parameters? → `ParameterFactory` (no wrapper; resolve directly)

### Which scope?

1. Stateless or cheap to create? → `.unique` (default)
2. Shared within a session/feature? → `.cached` or custom scope
3. Lives as long as someone holds it? → `.shared`
4. One global instance forever? → `.singleton` (use sparingly; complicates testing)
5. Same instance within one resolve chain? → `.graph`

## Behavioral Rules

**Hard rules (correctness):**
- Factory definitions MUST be computed properties — never stored or lazy
- `@MainActor` on both the factory property AND the closure
- Use explicit `reset(options:)` — bare `reset()` is destructive
- `FactoryTesting` in test targets only
- Custom scopes: `static let`, never `static var`
- Custom `SharedContainer`: `final class`, `shared` as `@TaskLocal static var` or `static let` — never bare `static var`
- `await` for @MainActor factories in ContainerTrait closures
- After changing a scoped context, also `.reset(.scope)` to clear stale cache

**Defaults (prefer unless context dictates otherwise):**
- Prefer `self { }` sugar over `Factory(self) { }`
- Prefer protocol return types: `Factory<MyProtocol>` not `Factory<ConcreteType>`
- Prefer constructor injection over property wrappers in new code
- Prefer `cached` over `singleton` unless truly global instance needed
- Always include test isolation in generated test code (ContainerTrait or push/pop)
- Put mutable contexts in `autoRegister`, not inline in factory definitions
- Use `promised()` for cross-module optional services

**Migration / compatibility:**
- `@retroactive AutoRegistering` when extending default Container in Swift 6 (suppresses conformance warning)
- Upstream docs/examples may still show `import Factory` — this is migration drift, not necessarily wrong code
