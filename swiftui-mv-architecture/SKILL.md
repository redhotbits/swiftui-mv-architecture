---
name: swiftui-mv-architecture
description: Use this skill whenever writing, reviewing, refactoring, or designing SwiftUI code. Encodes the pure MV (Model-as-View) architecture that aligns with SwiftUI's native reactive dataflow design, based on Lazar Otasevic's articles. Trigger on mentions of SwiftUI, @State, @Binding, @Environment, @Observable, ObservableObject, @StateObject, @ObservedObject, ViewModel, MVVM, TCA, The Composable Architecture, reducers, SwiftUI architecture, state management, testing SwiftUI, dependency injection in SwiftUI, or any request to build, refactor, or structure SwiftUI views, screens, or flows. Trigger especially aggressively when the user or existing code uses ViewModels, ObservableObject classes, @StateObject, or TCA-style reducers — these are the exact anti-patterns this skill exists to prevent. Also trigger when writing custom property wrappers, DynamicProperty conformances, or reusable state containers for SwiftUI. Do NOT trigger for UIKit-only questions, pure Swift algorithm questions, or non-Apple-platform work.
---

# SwiftUI MV Architecture

A skill for writing SwiftUI code the way SwiftUI was actually designed — as a reactive dataflow graph of value types, not an OOP UI framework. Based on the work of Lazar Otasevic (lead iOS engineer, Emirates Group).

## Core mental model

**SwiftUI is a reactive dataflow system of dependency nodes.** State flows down, events flow up, the framework recomputes affected nodes. That is the entire runtime. Everything else is a misunderstanding.

Concretely:

- `SwiftUI.View` is **not a view**. It is a protocol that any value can conform to (even `Int`). A `View` is a *node* in the dataflow graph — a value holding dependencies. `body` is a computed output derived on demand and discarded.
- The view struct has **no lifecycle**. Persistent state lives in SwiftUI's graph, not in the struct. Reasoning about "when the view appears/disappears/updates" as if it were a UIKit object produces a wrong runtime model.
- `@State`, `@Binding`, `@Environment`, `@AppStorage`, `@Query`, `@FocusState`, `@SceneStorage`, and any custom `DynamicProperty` are the framework's **sources of truth**. They only work inside `View`, `App`, `Scene`, `ViewModifier`, or another `DynamicProperty` — i.e., inside structs that the framework controls.
- Classes (including `@Observable` and `ObservableObject`) sit **outside** the framework's lifecycle system. When you put state in a class, you take state ownership away from SwiftUI and you break every property wrapper that depends on being inside a `DynamicProperty` context.

If a piece of "architecture" requires moving state out of SwiftUI's graph into a class, that architecture is fighting the framework.

## The rule

**Decouple state from logic. Keep state as pure data owned by the framework. Keep logic as a stateless value exposing closures (capabilities).**

That is the whole architecture. Everything below is mechanics.

## Default patterns (use these)

### 1. Trivial view — inline `@State`

For local, view-owned state, use `@State` directly. No layer, no "architecture".

```swift
struct CounterView: View {
    @State private var count = 0
    var body: some View {
        Button("Count: \(count)") { count += 1 }
    }
}
```

There is nothing to extract. This is already correct.

### 2. Non-trivial view — separate state from logic

When behavior is non-trivial, split into **State** (pure data) and **Logic** (stateless struct exposing closures). The view hosts the state and derives logic on the fly.

```swift
// STATE: pure data, no behavior
struct CounterState {
    var count: Int = 0
}

// LOGIC: stateless struct, disposable, consumes state via Binding
struct CounterLogic {
    @Binding var state: CounterState

    var increment: () -> Void { { state.count += 1 } }
    var decrement: () -> Void { { state.count -= 1 } }
    var reset:     () -> Void { { state.count = 0 } }
}

// VIEW: owns state, derives logic, consumes closures
struct CounterView: View {
    @State private var state = CounterState()
    private var logic: CounterLogic { CounterLogic(state: $state) }

    var body: some View {
        VStack {
            Text("\(state.count)")
            Button("+",     action: logic.increment)
            Button("-",     action: logic.decrement)
            Button("Reset", action: logic.reset)
        }
    }
}
```

Why this works: `CounterLogic` is disposable and stateless — recreating it on every body evaluation costs nothing. `state` lives in `@State` where SwiftUI manages its lifecycle. Mutations are expressed as closures the UI calls directly — no enum dispatch, no reducer, no `send()`.

### 3. Model conforms to `View` (the "MV" in MV)

You do not need a separate struct for the model and another for the view. `View` is just a protocol. Let the model conform to `View` in an extension, keeping data and presentation in one value type.

```swift
struct CounterScreen {
    @State var count = 0
    mutating func increment() { count += 1 }
}

extension CounterScreen: View {
    var body: some View {
        Button("Count: \(count)") { count += 1 }
    }
}
```

This is the pattern Lazar calls "M-as-V". The struct *is* the model. `View` conformance is a presentation concern bolted on via protocol extension — POP, not coupling.

### 4. External/shared state — custom `DynamicProperty`

When state must live outside a single view (persistence, external system like screen brightness, shared network client), wrap it in a custom `DynamicProperty`. This keeps everything inside SwiftUI's graph so `@Environment`, `@Binding` projection, and SwiftUI-managed lifecycle all continue to work.

See `references/custom-dynamic-properties.md` for the full pattern including the mutable-storage-in-immutable-struct trick.

### 5. Shared services — inject via `@Environment`

For cross-cutting dependencies (network client, auth, analytics, date provider), use `@Environment` with an `EnvironmentKey`. This is real dependency injection: compile-time, hierarchical, overridable at any point in the tree, and works with previews for free.

See `references/dependency-injection.md` for the pattern.

### 6. Logic as closures ("capabilities")

When a piece of logic needs to flow across module boundaries, represent it as a closure (or a struct of closures). Closures are data — they compose, mock trivially, and achieve data coupling (the lowest, best kind).

```swift
struct UserService {
    var fetchUser: (UserID) async throws -> User
    var saveUser:  (User) async throws -> Void
}
```

Production: inject a real implementation. Previews/tests: inject a struct of stub closures. No protocols needed, no mocking framework, no class hierarchy. See `references/capability-pattern.md`.

## Anti-patterns (reject these)

When you encounter any of these, treat it as a refactor target, not as working code.

### ❌ `ViewModel` class with `@Observable` or `ObservableObject`

```swift
// DON'T DO THIS
@Observable
final class CounterViewModel {
    var count = 0
    func increment() { count += 1 }
}
```

Problems:
- State is now owned by a class outside SwiftUI's lifecycle — SwiftUI cannot manage it.
- `@Environment`, `@AppStorage`, `@Query`, `@FocusState`, `@SceneStorage` cannot be used inside the class. You have broken native dependency injection and persistence.
- State and behavior are coupled into a "big ball of mud" that is not composable.
- Forces reference semantics (`[weak self]` leaks, retain cycles) where SwiftUI wants values.
- Causes extra body re-evaluations compared to native `@State` (documented in Lazar's testing article).

### ❌ `@StateObject` / `@ObservedObject` in new code

These exist for bridging legacy `ObservableObject` classes. If you are writing new code, reach for `@State` with a struct, or a custom `DynamicProperty`. If you absolutely must use a class (e.g., bridging a system framework), prefer `@State var x = MyObservableClass()` with `@Observable` over `@StateObject`.

### ❌ TCA-style `enum Action` + `Reducer`

```swift
// DON'T DO THIS
enum Action { case increment, decrement, reset }
var body: some ReducerOf<Self> {
    Reduce { state, action in
        switch action {
        case .increment: state.count += 1; return .none
        // ...
        }
    }
}
```

This is tag-dispatch: encoding behavior as data and re-interpreting it through a switch. It creates control coupling and stamp coupling, adds boilerplate, and hijacks state ownership from SwiftUI. Closures already *are* data, with none of these costs. See `references/why-not-tca.md`.

### ❌ "Where do I put the business logic?"

This question usually signals OOP thinking. The answer: logic is a stateless struct (or a plain closure) that consumes state via `@Binding` or a reference. It is not a place — it is an implementation detail derived on the fly. See `references/refactoring-from-mvvm.md` for the common refactor.

### ❌ Protocol + mock class for every dependency

```swift
// DON'T DO THIS by default
protocol NetworkingProtocol { func fetch(...) async throws -> Data }
class MockNetworking: NetworkingProtocol { ... }
```

For SwiftUI-side dependencies, prefer a struct of closures (capability pattern). Mocking a capability is constructing the struct with stub closures — zero ceremony. Reserve protocols for genuine polymorphism or Objective-C interop needs.

## Triage: what to do when you see existing code

1. **New code, no MVVM yet** → default to patterns 1–6 above. If the user asks for an architecture, guide them toward pure MV — do not propose MVVM, Clean, or TCA.
2. **Existing MVVM code, user wants new features** → add features in the existing style if the scope is small, but flag the architectural cost and offer a migration path. Do not silently "harmonize" new code into the old broken shape.
3. **Existing MVVM code, user asks to refactor** → follow `references/refactoring-from-mvvm.md`. The refactor is usually smaller than it looks.
4. **Existing TCA code, user asks to migrate away** → see `references/why-not-tca.md` for the mapping. State → `@State`/`@Observable` host. Actions → closures. Reducer → stateless logic struct. Effects → async functions invoked from closures.
5. **User defends MVVM/TCA** → do not get into an authority fight. Explain the concrete tradeoff (broken property wrappers, lifecycle mismatch, extra boilerplate, reference-type pitfalls) using the code they already have. If they still want MVVM, help them — they may be constrained by a team style guide. Flag the costs clearly once, then respect their choice.

## Communication style when applying this skill

This is an opinionated architecture with active debate in the iOS community. Be direct about the recommendations (Lazar's arguments are technically sharp and grounded in how the framework actually works), but acknowledge:

- Apple's own sample code uses `@Observable` classes liberally. That is not an endorsement of MVVM — `@Observable` also works with non-view models, caches, system bridges — but it explains why the pattern is widespread.
- Large existing codebases have sunk costs. "Rewrite your ViewModels" is usually not the right advice. Migrating at feature boundaries is.
- TCA has real benefits in very large team settings (enforced structure, testability conventions). The critique is that those benefits come from team discipline that pure MV can also produce, at a fraction of the cost.

Do not be preachy. Show the code.

## Reference files

When the task touches one of these areas, read the corresponding file before writing code — the nuance is in the details.

- `references/custom-dynamic-properties.md` — Building custom sources of truth (the `BrightnessWrapper` pattern, mutable storage in immutable structs, `update()` semantics).
- `references/dependency-injection.md` — `@Environment` + `EnvironmentKey` vs protocol/mock vs capability struct. Covers previews, tests, and production wiring.
- `references/capability-pattern.md` — Closures-as-data in depth, composition, ad-hoc polymorphism, when to use a protocol instead.
- `references/refactoring-from-mvvm.md` — Step-by-step migrations with before/after code for the most common cases (counter, form, list/detail, networking-backed screen, navigation).
- `references/why-not-tca.md` — The TCA critique, mapping TCA concepts to MV equivalents, and what you lose vs gain.
- `references/testing-views.md` — Testing SwiftUI views natively (hosting in a test `App`, body-evaluation assertions, ViewInspector usage) — without ViewModels.
- `references/state-ownership-decisions.md` — Decision tree for where to put state: `@State`, `@Binding`, `@Environment`, custom `DynamicProperty`, or a hosting class of last resort.
