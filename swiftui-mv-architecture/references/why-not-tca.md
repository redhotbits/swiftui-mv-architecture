# Why Not TCA — and How to Migrate

The Composable Architecture (TCA) is a popular SwiftUI architecture built on enum-typed actions, reducers, stores, and effects. It has a devoted community and is measurably better than MVVM on several axes (testability, enforced structure, undo/redo, time-travel debugging).

The critique below is not that TCA is *bad* — it is that TCA solves problems SwiftUI does not have, at a cost, by introducing patterns that Swift idiomatically handles more elegantly.

## The two fundamental issues

### 1. Tag-dispatch encoded as data

TCA represents behavior as enum cases and dispatches them through a reducer:

```swift
@Reducer
struct CounterFeature {
    @ObservableState
    struct State: Equatable { var count = 0 }

    enum Action {
        case increment, decrement, reset
    }

    var body: some ReducerOf<Self> {
        Reduce { state, action in
            switch action {
            case .increment: state.count += 1; return .none
            case .decrement: state.count -= 1; return .none
            case .reset:     state.count  = 0; return .none
            }
        }
    }
}
```

This is the classic "tag-dispatch" / "type-code" antipattern (see Fowler, *Refactoring*): encoding behavior as tagged data and interpreting it in a switch. Swift already has a better primitive for "behavior as data" — **closures**. A closure *is* behavior-as-data, and it executes directly without an interpreter layer.

The cost of the tag-dispatch form:
- Every new behavior requires touching: the enum, the reducer switch, any parent reducers that forward the action, any tests that pattern-match on actions.
- Composition is done via nested enums (`PullbackAction`) and `Scope` reducers — indirection on top of indirection.
- Behavior cannot be partially overridden — you cannot subclass an enum case.
- The reducer body is non-local: reading the code for "what happens on increment" means jumping through an enum case label.

With closures:
```swift
var increment: () -> Void { { state.count += 1 } }
```
Behavior is named where it lives. Composition is function composition. Override is closure wrapping. Tests call the closure directly.

### 2. State ownership hijacking

SwiftUI's runtime owns state. `@State`, `@Environment`, `@AppStorage`, `@FocusState`, `@SceneStorage`, `@Query` — these are all first-class and only work inside `DynamicProperty` contexts. TCA moves state into a `Store` (a class), which sits outside SwiftUI's graph.

Consequences:
- `@Environment` inside TCA state does not work. You get the `Dependencies` client system instead — a reimplementation.
- `@AppStorage` inside TCA state does not persist. You get `@Shared(.appStorage(...))` — another reimplementation.
- SwiftUI's automatic view-level observation tracking (from `@Observable`) is replaced by TCA's own observation machinery.
- SwiftUI scene/app lifecycle events have to be bridged back into actions.

TCA is, in effect, a parallel implementation of SwiftUI's state system — one that existed for good reasons before `@Observable` and the modern property wrappers, but whose reason for existing is thinner each year.

## TCA → MV concept mapping

| TCA                                    | MV equivalent                                       |
| -------------------------------------- | --------------------------------------------------- |
| `State` struct                         | `struct FeatureState` (same thing, minus `@ObservableState`) |
| `enum Action` + reducer                | Closures on a `FeatureLogic` struct                 |
| `Reducer` body `switch`                | One closure per former action case                  |
| `Effect.run { ... }`                   | `async` function invoked from a closure             |
| `Scope` / child reducer composition    | Logic structs compose via nested state + nested logic |
| `Store<State, Action>`                 | `@State` host holding the `State` struct            |
| `StoreOf<Feature>` in views            | `@Binding var state: FeatureState` + logic          |
| `store.send(.increment)`               | `logic.increment()`                                 |
| `TestStore` + action assertions        | XCTest asserting on `state` mutations directly      |
| `Dependencies` client                  | `@Environment` value or closure parameter           |
| `@Shared(.appStorage(...))`            | `@AppStorage` natively                              |
| `@Presents` / `@Reducer(state:action:)`| `@State` + child view with `@Binding`               |

## Migration sketch

### Before (TCA)

```swift
@Reducer
struct LoginFeature {
    @ObservableState
    struct State: Equatable {
        var email = ""
        var password = ""
        var isSubmitting = false
        var error: String?
    }

    enum Action {
        case emailChanged(String)
        case passwordChanged(String)
        case submitTapped
        case submitResponse(Result<User, Error>)
    }

    @Dependency(\.authClient) var auth

    var body: some ReducerOf<Self> {
        Reduce { state, action in
            switch action {
            case let .emailChanged(v):    state.email = v;    return .none
            case let .passwordChanged(v): state.password = v; return .none
            case .submitTapped:
                state.isSubmitting = true
                return .run { [email = state.email, password = state.password] send in
                    await send(.submitResponse(Result { try await auth.login(email, password) }))
                }
            case let .submitResponse(.success(user)):
                state.isSubmitting = false
                state.error = nil
                return .none
            case let .submitResponse(.failure(error)):
                state.isSubmitting = false
                state.error = error.localizedDescription
                return .none
            }
        }
    }
}

struct LoginView: View {
    @Bindable var store: StoreOf<LoginFeature>
    var body: some View {
        Form {
            TextField("Email", text: $store.email.sending(\.emailChanged))
            SecureField("Password", text: $store.password.sending(\.passwordChanged))
            if let e = store.error { Text(e).foregroundStyle(.red) }
            Button("Log in") { store.send(.submitTapped) }.disabled(store.isSubmitting)
        }
    }
}
```

### After (MV)

```swift
struct LoginState {
    var email = ""
    var password = ""
    var isSubmitting = false
    var error: String?
}

struct LoginLogic {
    @Binding var state: LoginState
    let login: (String, String) async throws -> User

    var submit: () async -> Void {
        {
            state.isSubmitting = true
            defer { state.isSubmitting = false }
            do {
                _ = try await login(state.email, state.password)
                state.error = nil
            } catch {
                state.error = error.localizedDescription
            }
        }
    }
}

struct LoginView: View {
    @State private var state = LoginState()
    let login: (String, String) async throws -> User

    private var logic: LoginLogic { LoginLogic(state: $state, login: login) }

    var body: some View {
        Form {
            TextField("Email",    text: $state.email)
            SecureField("Password", text: $state.password)
            if let e = state.error { Text(e).foregroundStyle(.red) }
            Button("Log in") { Task { await logic.submit() } }
                .disabled(state.isSubmitting)
        }
    }
}
```

What was eliminated:
- `enum Action` (4 cases) → gone
- Reducer `switch` (5 cases, one per action) → gone
- `Effect.run` wrapping → plain `async` call
- `sending(\.emailChanged)` binding forwarding → native `$state.email` binding
- `@Bindable var store` + `StoreOf<Feature>` → `@State` + a binding
- `@Dependency(\.authClient)` → plain closure parameter

The whole feature is a struct, a logic struct, and a view. The code you read is the code that runs.

## Testing comparison

### TCA
```swift
let store = TestStore(initialState: LoginFeature.State()) {
    LoginFeature()
} withDependencies: {
    $0.authClient.login = { _, _ in .mock }
}

await store.send(.emailChanged("a@b.com")) { $0.email = "a@b.com" }
await store.send(.passwordChanged("12345678")) { $0.password = "12345678" }
await store.send(.submitTapped) { $0.isSubmitting = true }
await store.receive(\.submitResponse.success) { $0.isSubmitting = false }
```

### MV
```swift
var state = LoginState(email: "a@b.com", password: "12345678")
let logic = LoginLogic(state: Binding(get: { state }, set: { state = $0 }),
                       login: { _, _ in .mock })

await logic.submit()

XCTAssertFalse(state.isSubmitting)
XCTAssertNil(state.error)
```

The MV version does not need a test-only framework (`TestStore`). It tests Swift values with Swift assertions.

## What TCA genuinely gives you, and how to replicate it

1. **Enforced structure across a large team.** TCA is prescriptive — every feature looks the same. MV is permissive. A team using MV needs conventions (file layout, naming, a preferred shape for Logic structs) written down. This is a real cost.
2. **Time-travel debugging.** Actions-as-data let you log and replay. If you need this (rare), you can approximate it in MV by logging state diffs, or by building a thin action-log layer on top of closures *where it matters*. You do not need to pay the tax everywhere.
3. **Parent-child action forwarding for complex navigation.** TCA has worked hard on `@Presents`, `@Reducer(state:action:)`. In MV the equivalent is passing down `@Binding`s and child logic — less ceremony, though also less scaffolding if your team relies on the scaffolding.
4. **Exhaustive testing.** TCA's `TestStore` enforces asserting on every state change. In MV you choose what to assert. A team could adopt a convention ("tests must assert final state") to get equivalent discipline.

If your team has invested in TCA and ships reliably, do not migrate for philosophical reasons. Migrate if the cost/benefit tilts — new features, simpler onboarding, dropping a dependency, removing a class of bugs from state ownership. Flag the real tradeoffs; do not oversell MV as strictly superior.
