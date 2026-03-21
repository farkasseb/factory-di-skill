# Factory Patterns & Core APIs

> **Read this file when**: defining factories, resolving dependencies, choosing between property wrappers, working with scopes, using ParameterFactory, creating custom containers, or understanding reset mechanics.

## Table of Contents

1. [Defining a Factory](#defining)
2. [Resolution Patterns](#resolution)
3. [Property Wrappers](#wrappers)
4. [Scopes](#scopes)
5. [ParameterFactory](#parameter-factory)
6. [Custom Containers](#custom-containers)
7. [Constructor Injection](#constructor-injection)
8. [Circular Dependencies](#circular)
9. [Reset Mechanics](#reset)

---

## Defining a Factory {#defining}

Factory definitions are **computed properties** on container extensions. Never stored or lazy.

```swift
// Idiomatic (preferred) — container creates the Factory via callAsFunction
extension Container {
    var myService: Factory<MyServiceProtocol> {
        self { MyService() }
    }
}

// Formal — explicit Factory construction (equivalent, more verbose)
extension Container {
    var myService: Factory<MyServiceProtocol> {
        Factory(self) { MyService() }
    }
}
```

**Why computed properties?** Factory structs are lightweight and transitory (like SwiftUI Views). They're created on access and immediately discarded after resolution. Stored/lazy properties create reference cycles and break Factory's registration override system.

### With scope

```swift
var myService: Factory<MyServiceProtocol> {
    self { MyService() }.cached
}
```

### With scope and context

```swift
var myService: Factory<MyServiceProtocol> {
    self { MyService() }
        .cached
        .onTest { MockService() }
}
```

---

## Resolution Patterns {#resolution}

### 1. callAsFunction (Service Locator)
```swift
let service = Container.shared.myService()
```

### 2. Explicit resolve
```swift
let service = Container.shared.myService.resolve()
```

### 3. @Injected (eager, on init)
```swift
class MyViewModel {
    @Injected(\.myService) private var service
}
```

### 4. @LazyInjected (lazy, on first access)
```swift
class MyViewModel {
    @LazyInjected(\.myService) private var service
}
```

### 5. @DynamicInjected (every access)
```swift
class MyViewModel {
    @DynamicInjected(\.myService) private var service
    // service is resolved fresh on every read
}
```

### 6. Constructor injection
```swift
class MyViewModel {
    let service: MyServiceProtocol
    init(container: Container) {
        service = container.myService()
    }
}
```

### 7. Custom container keyPath
```swift
@Injected(\MyContainer.myService) private var service
```

---

## Property Wrappers {#wrappers}

| Wrapper | Resolves | Returns | Getter | Use Case |
|---------|----------|---------|--------|----------|
| `@Injected` | On init | `T` | non-mutating | Default for classes |
| `@LazyInjected` | First access | `T` | **mutating** | Delay resolution; circular deps |
| `@WeakLazyInjected` | First access | `T?` | **mutating** | Delegate/parent-child (weak ref) |
| `@DynamicInjected` | Every access | `T` | non-mutating | Always-fresh; pair with scope |
| `@InjectedObject` | On init | `T: ObservableObject` | non-mutating | SwiftUI + ObservableObject |
| `@InjectedObservable` | First access (thunked) | `T: Observable` | non-mutating | SwiftUI + @Observable (iOS 17+) |

### LazyInjected mutating getter caveat

`LazyInjected.wrappedValue` has a `mutating get` because it initializes on first access. This means:
- In **structs**, every method that reads the property must be `mutating`
- Author's recommendation: avoid `@LazyInjected` in structs entirely; use `@Injected`, `@State`, or `@Observable` instead

### Projected values

- `$injected.resolve()` — force re-resolution
- `$injected.factory` — access the underlying Factory
- `$lazyInjected.resolvedOrNil()` — check if already resolved without triggering resolution (useful in `deinit`)

---

## Scopes {#scopes}

| Scope | Lifetime | Cache Location | Reset |
|-------|----------|---------------|-------|
| `.unique` (default) | None — new each time | N/A | N/A |
| `.cached` | Until cache/container reset | Container's cache | `container.reset(options: .scope)` |
| `.shared` | While strong refs exist | Container (weak ref) | Auto-releases |
| `.singleton` | Forever (global) | `Scope.singleton` cache | `Scope.singleton.reset()` |
| `.graph` | Single resolution cycle | Graph scope's own cache | Auto-clears after cycle |

### Custom scopes

```swift
extension Scope {
    static let session = Cached()  // Use `let`, not `var` — avoids concurrency warnings
}

// Usage
var myService: Factory<MyServiceProtocol> {
    self { MyService() }.scope(.session)
}

// Reset just session-scoped items
Container.shared.manager.reset(scope: .session)
```

### Time-to-live

```swift
var myService: Factory<MyServiceProtocol> {
    self { MyService() }.cached.timeToLive(300) // expires after 300 seconds
}
```

### Graph scope

Reuses instances within a single resolution cycle. If `ServiceA` and `ServiceB` both depend on `SharedDep`, graph scope ensures they get the same instance during one resolution:

```swift
var sharedDep: Factory<SharedDepProtocol> {
    self { SharedDep() }.graph
}
```

---

## ParameterFactory {#parameter-factory}

```swift
// Definition
extension Container {
    var parameterService: ParameterFactory<Int, MyServiceProtocol> {
        self { ParameterService(value: $0) }
    }
}

// Resolution — pass the parameter
let service = Container.shared.parameterService(42)
```

**Cannot use with @Injected wrappers** — no way to supply parameters at wrapper init.

### Multiple parameters (use tuple)

```swift
var tupleService: ParameterFactory<(Int, String), MyService> {
    self { (count, name) in MyService(count: count, name: name) }
}
```

### Scoped ParameterFactory

By default, a scoped ParameterFactory caches the **first** argument's result. Subsequent arguments are ignored.

Use `.scopeOnParameters` for per-parameter caching (parameter must be `Hashable`):

```swift
var service: ParameterFactory<Int, MyService> {
    self { MyService(value: $0) }.scopeOnParameters.cached
}
```

---

## Custom Containers {#custom-containers}

```swift
public final class MyContainer: SharedContainer {
    // Option A: @TaskLocal for test isolation (recommended)
    @TaskLocal public static var shared = MyContainer()

    // Option B: static let (if you don't need test trait isolation)
    // public static let shared = MyContainer()

    // NEVER: bare `static var` — causes concurrency warnings

    public let manager = ContainerManager()
    public init() {}
}

extension MyContainer {
    var myService: Factory<MyServiceProtocol> {
        self { MyService() }.cached
    }
}
```

### Default scope for all factories in a container

```swift
let container = MyContainer()
container.manager.defaultScope = Scope.cached
```

---

## Constructor Injection {#constructor-injection}

Resolve dependencies within the factory closure:

```swift
extension Container {
    var viewModel: Factory<ContentViewModel> {
        self { ContentViewModel(service: self.myService()) }
    }
    var myService: Factory<MyServiceProtocol> {
        self { MyService(network: self.networkService()) }
    }
    var networkService: Factory<NetworkProtocol> {
        self { URLSessionNetwork() }
    }
}
```

---

## Circular Dependencies {#circular}

Use `@LazyInjected` or `@WeakLazyInjected` to break circular dependency chains:

```swift
class ServiceA {
    @LazyInjected(\.serviceB) var b  // resolved later, not on init
}
class ServiceB {
    @Injected(\.serviceA) var a
}
```

Factory detects circular dependencies in DEBUG mode (default max depth: 8). Configure via:
```swift
Container.shared.manager.dependencyChainTestMax = 12
```

---

## Reset Mechanics {#reset}

### FactoryResetOptions

| Option | Clears | Use Case |
|--------|--------|----------|
| `.all` | Registrations + caches + options + defaultScope + re-triggers autoRegister | Full reset (destructive!) |
| `.registration` | Only registration overrides | Restore original factories, keep caches |
| `.scope` | Only scope caches | Keep registrations, clear cached instances |
| `.context` | Only context overrides | Keep registrations and caches |
| `.none` | Nothing | No-op |

### Per-container
```swift
container.reset(options: .scope)        // surgical
container.manager.reset(options: .all)  // equivalent to container.reset()
```

### Per-factory
```swift
container.myService.reset()                    // clears registration + scope
container.myService.reset(options: .scope)     // clears only cached instance
```

### Per-scope type
```swift
container.manager.reset(scope: .cached)  // clears all .cached items in this container
Scope.singleton.reset()                  // clears ALL singletons globally
```

**Remember**: `container.reset()` with no arguments uses `.all` — this is destructive. Always specify options when you don't want a full reset.
