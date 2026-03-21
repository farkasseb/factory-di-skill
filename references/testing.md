# Testing with Factory

> **Read this file when**: writing unit tests, mocking dependencies, using Swift Testing parallel tests, testing singletons, setting up XCTest with Factory, or configuring FactoryTesting.

## Table of Contents

1. [Swift Testing with ContainerTrait](#container-trait)
2. [Transform Sugar](#transform-sugar)
3. [Actor-Isolated Factories in Traits](#actor-traits)
4. [Custom Container Traits](#custom-traits)
5. [Multiple Container Traits](#multiple-traits)
6. [XCTest Patterns](#xctest)
7. [XCTest Parallel Testing](#xctest-parallel)
8. [Singleton Testing](#singletons)
9. [FactoryTesting Setup](#setup)
10. [Task.detached Pitfall](#task-detached)
11. [Common Setup Patterns](#common-setup)

---

## Swift Testing with ContainerTrait {#container-trait}

The `.container` trait provides a fresh, isolated container per test — enabling safe parallel execution.

```swift
import Testing
import FactoryTesting

@Suite(.container)  // every test gets its own container
struct MyTests {
    @Test func testLoaded() async {
        Container.shared.accountService.register { MockAccountService() }
        let vm = Container.shared.accountViewModel()
        await vm.load()
        #expect(vm.isLoaded)
    }

    @Test func testError() async {
        Container.shared.accountService.register { MockErrorService() }
        let vm = Container.shared.accountViewModel()
        await vm.load()
        #expect(vm.isError)
    }
}
```

How it works: `ContainerTrait` uses `@TaskLocal` to give each test its own `Container.shared` instance AND its own singleton scope (via `Scope.$singleton.withValue(Scope.singleton.clone())`).

### Suite trait is recursive

`@Suite(.container)` applies to all child suites and tests:

```swift
@Suite(.container)
struct ParentTests {
    @Test func testA() async { /* gets own container */ }

    @Suite
    struct ChildTests {
        @Test func testB() async { /* also gets own container */ }
    }
}
```

### Per-test trait

```swift
struct MyTests {
    @Test(.container)
    func testA() async { /* isolated */ }

    @Test  // no trait — uses shared global container
    func testB() async { /* NOT isolated */ }
}
```

---

## Transform Sugar {#transform-sugar}

Register mocks inline with the trait:

```swift
@Test(.container {
    $0.accountService.register { MockAccountService() }
    $0.networkService.register { MockNetwork() }
})
func testLoaded() async {
    let vm = Container.shared.accountViewModel()
    await vm.load()
    #expect(vm.isLoaded)
}
```

The transform closure is `@Sendable (Container) async -> Void` — async is key for actor-isolated factories.

---

## Actor-Isolated Factories in Traits {#actor-traits}

When registering `@MainActor` factories in a trait transform, use `await`:

```swift
// Given:
extension Container {
    @MainActor
    var viewModel: Factory<ContentViewModel> {
        self { @MainActor in ContentViewModel() }
    }
}

// In test:
@Test(.container {
    await $0.viewModel.register { @MainActor in MockViewModel() }
})
func testViewModel() async { ... }
```

Without `await`, Swift 6 emits: "Call to main actor-isolated initializer in a synchronous nonisolated context".

---

## Custom Container Traits {#custom-traits}

For custom containers, define a trait extension:

```swift
// 1. Custom container must use @TaskLocal
public final class MyContainer: SharedContainer {
    @TaskLocal public static var shared = MyContainer()
    public let manager = ContainerManager()
}

// 2. Define trait in test target
extension Trait where Self == ContainerTrait<MyContainer> {
    static var myContainer: ContainerTrait<MyContainer> {
        .init(shared: MyContainer.$shared, container: .init())
    }
}

// 3. Use it
@Test(.myContainer)
func testCustom() async {
    MyContainer.shared.myService.register { MockService() }
    ...
}
```

---

## Multiple Container Traits {#multiple-traits}

When code depends on multiple containers:

```swift
@Test(.container, .myContainer)
func testMultiple() async {
    let sut1 = Container.shared.service()
    let sut2 = MyContainer.shared.service()
    ...
}
```

---

## XCTest Patterns {#xctest}

### Push/Pop (recommended for XCTest)

```swift
final class MyTests: XCTestCase {
    override func setUp() {
        Container.shared.manager.push()
        Container.shared.setupMocks()
    }

    override func tearDown() {
        Container.shared.manager.pop()
    }

    func testSomething() async {
        Container.shared.service.register { MockService() }
        let vm = Container.shared.viewModel()
        await vm.load()
        XCTAssertTrue(vm.isLoaded)
    }
}
```

### Reset pattern (simpler, no teardown)

```swift
override func setUp() {
    Container.shared.reset()
    Container.shared.setupMocks()
}
```

### Passed container (fully isolated)

```swift
func testSomething() {
    let container = Container()
    container.service.register { MockService() }
    let vm = MyViewModel(container: container)
    vm.load()
    XCTAssertTrue(vm.isLoaded)
}
```

---

## XCTest Parallel Testing {#xctest-parallel}

Use `Container.$shared.withValue` for XCTest parallel isolation:

```swift
final class ParallelXCTest: XCTestCase {
    func testFoo() {
        let container = Container()
        let expectation = expectation(description: "foo")

        Container.$shared.withValue(container) {
            Container.shared.service.register { MockFoo() }
            let sut = MyUseCase()
            XCTAssertEqual(sut.result, "foo")
            expectation.fulfill()
        }

        waitForExpectations(timeout: 1)
    }
}
```

---

## Singleton Testing {#singletons}

**Critical**: Singletons are NOT per-container. `push()`/`pop()` and `container.reset()` do NOT affect them.

### ContainerTrait handles singletons automatically

`ContainerTrait.provideScope` wraps tests in `Scope.$singleton.withValue(Scope.singleton.clone())` — each test gets its own singleton scope copy.

### Manual singleton reset

```swift
// Reset ALL singletons globally
Scope.singleton.reset()

// Reset a specific singleton factory
Container.shared.mySingleton.reset(options: .scope)

// Or register a new mock (clears the singleton cache for that factory)
Container.shared.mySingleton.register { MockSingleton() }
```

### autoRegister does NOT clear singleton caches

Inside `autoRegister`, calling `.register` on a singleton factory will NOT clear its cache. This is by design — auto-registration runs on every container creation, and clearing singletons each time would defeat their purpose.

```swift
extension Container: @retroactive AutoRegistering {
    func autoRegister() {
        // This registration does NOT clear the singleton cache
        mySingleton.register { ProductionSingleton() }
    }
}
```

Outside of `autoRegister`, `.register` DOES clear the singleton cache.

---

## FactoryTesting Setup {#setup}

### SPM

```swift
.testTarget(
    name: "MyAppTests",
    dependencies: ["FactoryTesting"]
)
```

### In test files

```swift
import Testing
import FactoryTesting
```

**CRITICAL — distinguish imports from dependencies:**

| What | Where | OK? |
|------|-------|-----|
| `"FactoryTesting"` in `.testTarget(dependencies:)` | Package.swift | YES |
| `"FactoryKit"` in `.testTarget(dependencies:)` | Package.swift | NO — duplicates factories |
| `import FactoryTesting` in test source file | .swift file | YES |
| `import FactoryKit` in test source file | .swift file | YES — get it transitively through your app |

- `FactoryTesting` is guarded by `#if DEBUG` and `#if swift(>=6.1)` — never add to app targets
- Your test target gets `FactoryKit` types transitively through the app target dependency — adding it again as a direct target dependency creates duplicate factory registrations

---

## Task.detached Pitfall {#task-detached}

`Task.detached` does NOT inherit `@TaskLocal` values. Mocked dependencies registered via `ContainerTrait` or `$shared.withValue` will NOT resolve in detached tasks:

```swift
@Test(.container {
    $0.service.register { MockService() }
})
func testDetached() async {
    // This resolves the MOCK (inherits TaskLocal)
    let service1 = Container.shared.service()

    Task.detached {
        // This resolves the ORIGINAL (TaskLocal lost!)
        let service2 = Container.shared.service()
    }
}
```

**Fix**: Use `Task {}` (inherits TaskLocal) or pass the container/service explicitly.

---

## Common Setup Patterns {#common-setup}

### Shared mock setup

```swift
extension Container {
    func setupMocks() {
        myService.register { MockService() }
        network.register { MockNetwork() }
        analytics.register { MockAnalytics() }
    }
}
```

### Container transformation

```swift
let container = Container().with {
    $0.myService.register { MockService() }
    $0.network.register { MockNetwork() }
}
```

### UITesting with launch arguments

```swift
// Test case
let app = XCUIApplication()
app.launchArguments.append("mock1")
app.launch()

// In app's autoRegister
extension Container: @retroactive AutoRegistering {
    func autoRegister() {
        #if DEBUG
        myService.onArg("mock1") { MockService() }
        #endif
    }
}
```
