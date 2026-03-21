# Factory Wiring & Composition

> **Read this file when**: working with multi-module architecture, AutoRegistering, promised factories, contexts, decorators, the `.once()` modifier, debugging/tracing, or functional injection.

## Table of Contents

1. [Multi-Module Architecture](#multi-module)
2. [AutoRegistering](#auto-registering)
3. [Contexts](#contexts)
4. [The `.once()` Modifier](#once)
5. [Decorators](#decorators)
6. [Debugging and Tracing](#debugging)
7. [Functional Injection](#functional)

---

## Multi-Module Architecture {#multi-module}

### Same-module pattern

Protocol and implementation in same module — straightforward:

```swift
// In MyModule
public protocol AccountLoading { func load() -> [Account] }

extension Container {
    public var accountLoader: Factory<AccountLoading> {
        self { AccountLoaderImpl() }
    }
}
```

### Cross-module with promised()

When the protocol is in one module and the implementation is in another:

```swift
// In ModuleA (protocol module)
extension Container {
    public var accountLoader: Factory<AccountLoading?> { promised() }
}

// In ModuleB (implementation module) — registers in app target
extension Container: @retroactive AutoRegistering {
    public func autoRegister() {
        accountLoader.register { RealAccountLoader() }
    }
}
```

`promised()` returns `nil` by default but triggers `fatalError` in DEBUG if resolved without registration — catching missing wiring early.

### Adaptor pattern for third-party libraries

```swift
// Wrapper protocol in your module
protocol AnalyticsEngine { func track(_ event: String) }

// Adaptor in app target
class FirebaseAnalyticsAdaptor: AnalyticsEngine {
    func track(_ event: String) { Analytics.logEvent(event, parameters: nil) }
}

extension Container {
    var analytics: Factory<AnalyticsEngine> {
        self { FirebaseAnalyticsAdaptor() }
    }
}
```

---

## AutoRegistering {#auto-registering}

Called once before the first resolution on a container. Use for cross-module wiring, context setup, and default scope configuration.

```swift
extension Container: @retroactive AutoRegistering {
    func autoRegister() {
        // Cross-module wiring
        accountLoader.register { RealAccountLoader() }

        // Context-based mocking
        #if DEBUG
        analytics.onTest { MockAnalytics() }
        myService.onPreview { PreviewService() }
        #endif

        // Default scope
        manager.defaultScope = Scope.cached
    }
}
```

**Key behaviors**:
- `@retroactive` is required in Swift 6 when extending the default `Container`
- Called on each container **instance** creation, before first resolve
- `reset(options: .all)` re-triggers autoRegister on next resolve
- `reset(options: .registration)` also re-triggers autoRegister
- **Registration inside autoRegister does NOT clear singleton caches** — by design

---

## Contexts {#contexts}

### Available contexts

| Context | Availability | Check |
|---------|-------------|-------|
| `.test` / `onTest` | DEBUG only | `XCTestCase` in environment |
| `.preview` / `onPreview` | DEBUG only | `XCODE_RUNNING_FOR_PREVIEWS` |
| `.debug` / `onDebug` | DEBUG only | `#if DEBUG` |
| `.simulator` / `onSimulator` | Always | `#if targetEnvironment(simulator)` |
| `.device` / `onDevice` | Always | `!simulator` |
| `.arg(String)` / `onArg` | DEBUG only | `ProcessInfo.arguments.contains` |
| `.args([String])` / `onArgs` | DEBUG only | Multiple argument match |

### Precedence order

`arg(s)` > `preview` > `test` > `simulator` > `device` > `debug` > `registered` > `original`

### The "Factory wins" problem

Inline context modifiers are reapplied on **every** computed property access:

```swift
extension Container {
    var myService: Factory<MyServiceProtocol> {
        self { RealService() }
            .onTest { TestService() }  // reapplied every time
    }
}

// Later in test:
Container.shared.myService.register { SpecialMock() }
// On next resolve, the .onTest context is reapplied and may override SpecialMock!
```

**Solutions**:
1. Use `.once()` modifier (see below)
2. Define contexts in `autoRegister` instead of inline
3. If you change a scoped context, also `.reset(.scope)` to clear stale cache

### Runtime arguments

```swift
FactoryContext.setArg("dark", forKey: "theme")
FactoryContext.removeArg(forKey: "theme")

extension Container {
    var styleSystem: Factory<Theme> {
        self { StandardTheme() }
            .onArg("light") { LightTheme() }
            .onArg("dark") { DarkTheme() }
    }
}
```

---

## The `.once()` Modifier {#once}

Prevents a modifier from being reapplied after first execution. Critical for contexts that should not override later registrations:

```swift
extension Container {
    var myService: Factory<MyServiceProtocol> {
        self { RealService() }
            .onTest { TestService() }
            .once()  // context applied once, then locked
    }
}

// Now this registration will stick:
Container.shared.myService.register { SpecialMock() }
```

Without `.once()`, inline contexts are recalculated on every access to the computed property, potentially "winning" over later registrations.

**Best practice**: Use `.once()` on any factory with inline contexts that might be overridden in tests. Or better yet, define contexts in `autoRegister`.

---

## Decorators {#decorators}

### Per-factory decorator

Runs on every resolution, including cached returns:

```swift
var myService: Factory<MyServiceProtocol> {
    self { MyService() }
        .cached
        .decorator { print("Resolved: \(type(of: $0))") }
}
```

### Per-container decorator

Sees every dependency resolved by the container:

```swift
Container.shared.decorator { instance in
    print("Container resolved: \(type(of: instance))")
}

// Remove decorator
Container.shared.decorator(nil)
```

---

## Debugging and Tracing {#debugging}

### Enable tracing (DEBUG only)

```swift
Container.shared.manager.trace = true
```

### Custom logger

```swift
Container.shared.manager.logger = { message in
    os_log("%{public}s", message)
}
```

### Trace output format

```
0: Factory.Container.cycleDemo<CycleDemo> = N:105553131389696
1:     Factory.Container.aService<AServiceType> = N:105553119821680
2:         Factory.Container.implementsAB<AServiceType & BServiceType> = N:105553119821680
3:             Factory.Container.networkService<NetworkService> = N:105553119770688
1:     Factory.Container.bService<BServiceType> = N:105553119821680
2:         Factory.Container.implementsAB<AServiceType & BServiceType> = C:105553119821680
```

- Depth (0 = root)
- Factory path and type
- `N:` = new instance, `C:` = cached, `F:` = from registration override

### Circular dependency detection

Factory detects circular chains in DEBUG mode. Default max depth: 8.

```swift
Container.shared.manager.dependencyChainTestMax = 12  // increase if needed
```

---

## Functional Injection {#functional}

Inject functions instead of objects:

```swift
extension Container {
    var mathService: Factory<(Int, Int) -> Int> {
        self { { $0 + $1 } }
    }
}

// Usage
let add = Container.shared.mathService()
let result = add(2, 3) // 5

// Mock
Container.shared.mathService.register { { _, _ in 42 } }
```

Useful for injecting pure functions, closures, or simple transformations without creating wrapper types.
