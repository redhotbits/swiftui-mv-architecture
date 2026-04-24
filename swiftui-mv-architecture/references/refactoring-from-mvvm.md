# Refactoring from MVVM to MV

MVVM-to-MV refactors are almost always smaller than the size of the ViewModel suggests. Most ViewModels contain only three things: a few `@Published` properties (= state), a few methods that mutate them (= logic), and a constructor that wires dependencies (= injection). Each of those maps cleanly onto a SwiftUI-native primitive.

## The mapping

| MVVM concept                         | MV replacement                                     |
| ------------------------------------ | -------------------------------------------------- |
| `class VM: ObservableObject`         | `struct State` (pure data)                         |
| `@Published var x`                   | `var x` inside the `State` struct                  |
| `func doSomething()`                 | Closure on a stateless `Logic` struct, or free func |
| `@StateObject var vm = VM()`         | `@State var state = State()`                       |
| `@ObservedObject var vm: VM`         | `@Binding var state: State`                        |
| `@EnvironmentObject var vm: VM`      | `@Environment(\.x)` with an `EnvironmentKey`       |
| VM-injected service                  | Closure parameter or `@Environment` value          |
| VM-owned Combine subscription        | `.task { … }` or a custom `DynamicProperty`        |
| VM init side-effect                  | `.task` / `.onAppear` on the view                  |

## Refactor 1 — Counter

### Before (MVVM)

```swift
final class CounterViewModel: ObservableObject {
    @Published var count = 0
    func increment() { count += 1 }
    func reset()     { count  = 0 }
}

struct CounterView: View {
    @StateObject private var vm = CounterViewModel()
    var body: some View {
        VStack {
            Text("\(vm.count)")
            Button("+",     action: vm.increment)
            Button("Reset", action: vm.reset)
        }
    }
}
```

### After (MV — minimal)

```swift
struct CounterView: View {
    @State private var count = 0
    var body: some View {
        VStack {
            Text("\(count)")
            Button("+")     { count += 1 }
            Button("Reset") { count  = 0 }
        }
    }
}
```

For something this small, the "state + logic + view" split is overkill. The MVVM layer existed for no reason. Delete it.

### After (MV — when you want the split)

For non-trivial views, the explicit State + Logic pattern:

```swift
struct CounterState { var count = 0 }

struct CounterLogic {
    @Binding var state: CounterState
    var increment: () -> Void { { state.count += 1 } }
    var reset:     () -> Void { { state.count  = 0 } }
}

struct CounterView: View {
    @State private var state = CounterState()
    private var logic: CounterLogic { CounterLogic(state: $state) }
    var body: some View {
        VStack {
            Text("\(state.count)")
            Button("+",     action: logic.increment)
            Button("Reset", action: logic.reset)
        }
    }
}
```

## Refactor 2 — Form with validation and submission

### Before (MVVM)

```swift
final class SignupViewModel: ObservableObject {
    @Published var email = ""
    @Published var password = ""
    @Published var isSubmitting = false
    @Published var error: String?

    private let api: API
    init(api: API) { self.api = api }

    var canSubmit: Bool {
        email.contains("@") && password.count >= 8 && !isSubmitting
    }

    func submit() async {
        isSubmitting = true
        defer { isSubmitting = false }
        do {
            try await api.signup(email: email, password: password)
        } catch {
            self.error = error.localizedDescription
        }
    }
}

struct SignupView: View {
    @StateObject var vm: SignupViewModel
    var body: some View { /* uses vm.email, vm.password, vm.canSubmit, vm.submit() */ }
}
```

### After (MV)

```swift
// STATE — pure data
struct SignupState {
    var email = ""
    var password = ""
    var isSubmitting = false
    var error: String?
}

// DERIVED DATA — computed, not stored
extension SignupState {
    var canSubmit: Bool {
        email.contains("@") && password.count >= 8 && !isSubmitting
    }
}

// LOGIC — stateless, disposable, consumes state + injected capability
struct SignupLogic {
    @Binding var state: SignupState
    let signup: (String, String) async throws -> Void

    var submit: () async -> Void {
        {
            state.isSubmitting = true
            defer { state.isSubmitting = false }
            do {
                try await signup(state.email, state.password)
            } catch {
                state.error = error.localizedDescription
            }
        }
    }
}

// VIEW — hosts state, derives logic
struct SignupView: View {
    @State private var state = SignupState()
    let signup: (String, String) async throws -> Void

    private var logic: SignupLogic {
        SignupLogic(state: $state, signup: signup)
    }

    var body: some View {
        Form {
            TextField("Email",    text: $state.email)
            SecureField("Password", text: $state.password)
            if let e = state.error { Text(e).foregroundStyle(.red) }
            Button("Sign up") { Task { await logic.submit() } }
                .disabled(!state.canSubmit)
        }
    }
}

// Composition root
SignupView(signup: API.live.signup)

// Preview
SignupView(signup: { _, _ in })
```

Notes:

- Derived state (`canSubmit`) lives as a *computed property on State*, not on Logic. It is a function of state only — not behavior.
- The injected dependency (`signup`) is a closure, not a protocol. Compose in live implementations at the root; stub in previews/tests.
- The view uses `$state.email` directly for `TextField` bindings — no intermediate `@Published`/`@Binding` glue.

## Refactor 3 — List/Detail with async load

### Before (MVVM)

```swift
@MainActor
final class ProductListViewModel: ObservableObject {
    @Published var products: [Product] = []
    @Published var isLoading = false
    private let api: API
    init(api: API) { self.api = api }
    func load() async {
        isLoading = true
        defer { isLoading = false }
        products = (try? await api.listProducts()) ?? []
    }
}

struct ProductListView: View {
    @StateObject var vm: ProductListViewModel
    var body: some View {
        List(vm.products) { ProductRow(product: $0) }
            .overlay { if vm.isLoading { ProgressView() } }
            .task { await vm.load() }
    }
}
```

### After (MV)

```swift
struct ProductListView: View {
    let listProducts: () async throws -> [Product]

    @State private var products: [Product] = []
    @State private var isLoading = false

    var body: some View {
        List(products) { ProductRow(product: $0) }
            .overlay { if isLoading { ProgressView() } }
            .task {
                isLoading = true
                defer { isLoading = false }
                products = (try? await listProducts()) ?? []
            }
    }
}

// Root
ProductListView(listProducts: API.live.listProducts)

// Preview
ProductListView(listProducts: { [.sample1, .sample2] })
```

The task closure body *is* the load logic. There is no reason to extract it into a separate type — it runs in one place, in one context, and it is short. If it grows, pull it into a `Logic` struct per the pattern above.

## Refactor 4 — Shared state across screens

### Before (MVVM)

```swift
final class AppViewModel: ObservableObject {
    @Published var user: User?
    @Published var cart: [Product] = []
    // ...dozens of fields
}

// Injected via .environmentObject, everyone accesses everything
```

### After (MV)

Split by concern, inject each piece as `@Environment` values or typed sources of truth. If you truly have one big blob of app state, wrap it in a `@Observable` class hosted at the root with `@State`:

```swift
@Observable
final class AppSession {
    var user: User?
    var cart: [Product] = []
}

@main
struct MyApp: App {
    @State private var session = AppSession()
    var body: some Scene {
        WindowGroup {
            RootView().environment(session)
        }
    }
}

struct SomeChildView: View {
    @Environment(AppSession.self) private var session
    var body: some View { Text(session.user?.name ?? "guest") }
}
```

A class is acceptable here — but note:

- It is **hosted by `@State` at the root**, so SwiftUI controls its lifecycle.
- It contains **only state** (stored properties), not behavior. Logic that mutates it lives in separate stateless structs.
- It is *one* shared session, not a per-screen `ViewModel`.

This is the principled escape hatch. Use it sparingly.

## Refactor 5 — Side effects / subscriptions

### Before (MVVM)

```swift
final class BrightnessVM: ObservableObject {
    @Published var brightness: Double = 0.5
    private var c: AnyCancellable?
    init() {
        brightness = ExternalDisplay.currentBrightness
        c = ExternalDisplay.brightnessPublisher.sink { [weak self] in
            self?.brightness = $0
        }
    }
    func set(_ v: Double) {
        brightness = v
        ExternalDisplay.setBrightness(v)
    }
}
```

### After (MV)

Wrap the external system in a custom `DynamicProperty` (see `custom-dynamic-properties.md`). The view gains a `Binding<Double>` and knows nothing about subscriptions, retain cycles, or `[weak self]`.

## A general procedure

1. Identify what the ViewModel actually owns: stored properties vs computed vs methods vs side-effect subscriptions.
2. Move stored properties into a `State` struct (or straight into `@State` if local).
3. Move computed properties onto `State` as computed vars (they are functions of data).
4. Move methods into a `Logic` struct that takes `@Binding var state: State` and any injected closures.
5. Move subscriptions into a custom `DynamicProperty` (for external systems) or into `.task` (for one-shot loads).
6. Replace constructor-injected services with closure parameters or `@Environment` values.
7. Delete the class. Delete `@StateObject`. Delete `[weak self]`.

## When NOT to refactor

Do not refactor ViewModels *just because*. Refactor when:

- You are about to add a feature that bumps into the MVVM limitations (property wrappers not working, shared state coordination getting tangled, previews requiring extensive setup).
- You are rewriting the screen anyway.
- The team has agreed to migrate.

"Rewrite the whole app to MV" is rarely the right pitch. "Let's do the next feature in MV and leave the legacy screens alone" almost always is.
