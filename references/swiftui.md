# Factory + SwiftUI

> **Read this file when**: using Factory in SwiftUI views, integrating with @Observable or ObservableObject, setting up previews, dealing with @MainActor factory definitions, or choosing between InjectedObject and InjectedObservable.

## Table of Contents

1. [ObservableObject + @InjectedObject](#observable-object)
2. [@Observable + @InjectedObservable](#observable)
3. [@ObservationIgnored Requirement](#observation-ignored)
4. [Migration: ObservableObject to @Observable](#migration)
5. [@MainActor Factory Patterns](#main-actor)
6. [SwiftUI Preview Helpers](#previews)
7. [Choosing the Right Wrapper](#choosing)

---

## ObservableObject + @InjectedObject {#observable-object}

`@InjectedObject` uses `StateObject` internally — it owns the instance.

```swift
// ViewModel
class ContentViewModel: ObservableObject {
    @Injected(\.myService) private var service
    @Published var results: [Item] = []
    func load() async { results = await service.fetch() }
}

// Factory
extension Container {
    var contentViewModel: Factory<ContentViewModel> {
        self { ContentViewModel() }
    }
}

// View
struct ContentView: View {
    @InjectedObject(\.contentViewModel) private var viewModel
    var body: some View {
        List(viewModel.results) { item in Text(item.name) }
            .task { await viewModel.load() }
    }
}
```

Available: iOS 14+, macOS 11+.

### Alternative: Direct @StateObject

If the ViewModel manages its own dependencies via `@Injected`, no Factory needed for the VM itself:

```swift
struct ContentView: View {
    @StateObject private var viewModel = ContentViewModel()
    var body: some View { ... }
}
```

---

## @Observable + @InjectedObservable {#observable}

`@InjectedObservable` uses `@State` internally. It "thunks" the value — only one instance is created for the view's lifetime.

```swift
// ViewModel (iOS 17+)
@Observable
class ContentViewModel {
    @ObservationIgnored @Injected(\.myService) private var service
    var results: [Item] = []
    func load() async { results = await service.fetch() }
}

// Factory
extension Container {
    var contentViewModel: Factory<ContentViewModel> {
        self { ContentViewModel() }
    }
}

// View
struct ContentView: View {
    @InjectedObservable(\.contentViewModel) var viewModel
    var body: some View {
        List(viewModel.results) { item in Text(item.name) }
            .task { await viewModel.load() }
    }
}
```

Available: iOS 17+, macOS 14+.

---

## @ObservationIgnored Requirement {#observation-ignored}

**Always add `@ObservationIgnored` before `@Injected` in `@Observable` classes.** Without it, the `@Observable` macro's generated backing storage conflicts with `@Injected`'s property wrapper storage.

```swift
@Observable
class MyViewModel {
    // CORRECT
    @ObservationIgnored @Injected(\.service) private var service

    // WRONG — will cause issues
    // @Injected(\.service) private var service

    var data: [String] = []  // No @ObservationIgnored needed — this IS observed
}
```

Rule: `@ObservationIgnored` on injected dependencies (private, not observed), no annotation on published state.

---

## Migration: ObservableObject to @Observable {#migration}

| Before (ObservableObject) | After (@Observable) |
|---------------------------|---------------------|
| `class VM: ObservableObject` | `@Observable class VM` |
| `@Published var data` | `var data` |
| `@Injected(\.svc) var svc` | `@ObservationIgnored @Injected(\.svc) var svc` |
| `@InjectedObject(\.vm) var vm` | `@InjectedObservable(\.vm) var vm` |
| `@StateObject var vm` | `@State var vm` |
| `@ObservedObject var vm` | Direct reference (auto-tracked) |

**Do NOT use `@InjectedObject` with `@Observable`** — it requires `ObservableObject` conformance. Use `@InjectedObservable` instead.

---

## @MainActor Factory Patterns {#main-actor}

When a ViewModel is `@MainActor`, the factory needs `@MainActor` in **two places**:

```swift
@MainActor
@Observable
class ContentViewModel {
    @ObservationIgnored @Injected(\.myService) private var service
    var results: [Item] = []
}

// Factory — @MainActor on BOTH property AND closure
extension Container {
    @MainActor
    var contentViewModel: Factory<ContentViewModel> {
        self { @MainActor in ContentViewModel() }
    }
}
```

### Registering @MainActor mocks

```swift
// In production code or tests
Container.shared.contentViewModel.register {
    @MainActor in MockViewModel()
}
```

### In ContainerTrait transforms (use await)

```swift
@Test(.container {
    await $0.contentViewModel.register { @MainActor in MockViewModel() }
})
func testViewModel() async { ... }
```

---

## SwiftUI Preview Helpers {#previews}

### Instance method

```swift
#Preview {
    Container.shared.preview {
        $0.myService.register { MockService() }
        $0.network.register { MockNetwork() }
    }
    ContentView()
}
```

### Static method

```swift
#Preview {
    Container.preview {
        $0.myService.register { MockService() }
    }
    ContentView()
}
```

### Without helper (using `let _ =`)

```swift
#Preview {
    let _ = Container.shared.with {
        $0.myService.register { MockService() }
    }
    ContentView()
}
```

### Per-factory preview shortcut

```swift
extension Container {
    var myService: Factory<MyServiceProtocol> {
        self { MyService() }
            .preview { MockService() }  // only in previews
    }
}
```

### Using contexts for previews

```swift
extension Container: @retroactive AutoRegistering {
    func autoRegister() {
        #if DEBUG
        myService.onPreview { MockService() }
        #endif
    }
}
```

---

## Choosing the Right Wrapper {#choosing}

### For SwiftUI Views

| Your ViewModel Type | Use This Wrapper | Notes |
|---------------------|------------------|-------|
| `ObservableObject` | `@InjectedObject(\.vm)` | iOS 14+, uses StateObject |
| `@Observable` | `@InjectedObservable(\.vm)` | iOS 17+, uses State, thunked |

### For Non-View Classes

| Need | Use |
|------|-----|
| Resolve once at init | `@Injected(\.service)` |
| Resolve lazily | `@LazyInjected(\.service)` |
| Always fresh | `@DynamicInjected(\.service)` |
| Weak reference | `@WeakLazyInjected(\.service)` |

### Key Decision: Does the View Own the ViewModel?

- **Yes (typical)** → `@InjectedObject` or `@InjectedObservable` — both own the instance like `@StateObject`/`@State`
- **No (passed from parent)** → Resolve via Factory in parent, pass as parameter
