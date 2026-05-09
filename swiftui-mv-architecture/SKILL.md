---
name: swiftui-mv-architecture
description: Use this skill whenever writing, reviewing, refactoring, or designing SwiftUI code. Encodes the pure MV (Model-as-View) architecture that aligns with SwiftUI's native reactive dataflow design, based on Lazar Otasevic's articles. Trigger on mentions of SwiftUI, @State, @Binding, @Environment, @Observable, ObservableObject, @StateObject, @ObservedObject, ViewModel, MVVM, TCA, The Composable Architecture, reducers, SwiftUI architecture, state management, testing SwiftUI, dependency injection in SwiftUI, or any request to build, refactor, or structure SwiftUI views, screens, or flows. Trigger especially aggressively when the user or existing code uses ViewModels, ObservableObject classes, @StateObject, or TCA-style reducers — these are the exact anti-patterns this skill exists to prevent. Also trigger when writing custom property wrappers, DynamicProperty conformances, or reusable state containers for SwiftUI, when extracting reusable views/components, when forking a component to change its look, or when designing a styling API for a custom view (ButtonStyle/ToggleStyle/LabelStyle-style protocols, makeBody, style configurations, environment-propagated styles). Do NOT trigger for UIKit-only questions, pure Swift algorithm questions, or non-Apple-platform work., file organization for SwiftUI projects, folder structure, component reuse, custom styles, styleable components
---

# SwiftUI MV Architecture

A skill for writing SwiftUI code the way SwiftUI was actually designed — as a reactive dataflow graph of value types, not an OOP UI framework. Based on the work of Lazar Otasevic (lead iOS engineer, Emirates Group).

## Scan first

Don't read this whole file end-to-end. Route to the section you need.

**Writing or editing any SwiftUI code** → read `references/code-style.md` once before the first edit. Every new view also gets a `#Preview` block (see Previews section). Both non-negotiable.

| If the task is… | Go to |
|---|---|
| "Where should this state live?" | Default patterns 1–6 + `references/state-ownership-decisions.md` |
| Existing `ViewModel` / `@StateObject` / TCA reducer to refactor | Anti-patterns + Triage + `references/refactoring-from-mvvm.md` or `references/why-not-tca.md` |
| New project setup, file structure | File organization + App entry point + their references |
| Building or refactoring a custom view/component | Reusable components + `references/custom-component-styling.md` |
| Styling a built-in view (`Button`, `Toggle`, `Label`, …) | Reusable styles + `references/style-protocols.md` |
| Custom source of truth (persistence, system bridge, sensor) | `references/custom-dynamic-properties.md` |
| Using `@Query` / SwiftData fetch in a non-trivial parent | `references/swiftdata-query.md` |
| Injecting a service / dependency | `references/dependency-injection.md` + `references/capability-pattern.md` |
| Building HTTP requests (with `declarative-requests-swift`) | `references/declarative-requests-networking.md` |
| Writing tests for a SwiftUI view | `references/testing-views.md` |
| Considering a class for new code | "When classes ARE allowed" section |
| Writing a custom `init` for a View | "View init: data plumbing only" section |
| User defends MVVM/TCA | Triage step 5 + Communication style |

If nothing matches, default to **patterns 1–6 below** — they cover most real SwiftUI work.

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

**Don't let `CounterState` grow into a `ViewState` umbrella.** A single-field state struct works because the field has real meaning — `count` *is* the counter's model. The trap is treating "view state" itself as the grouping and bundling unrelated screen properties together because they render on the same screen:

```swift
// ❌ accidental UI grouping — split the view tomorrow and this struct can't split with it
struct SearchViewState {
    var results: [Result] = []
    var query: String = ""
    var isLoading = false
    var errorMessage: String?
    var scrollTargetId: Int?
}

struct SearchLogic {
    @Binding var state: SearchViewState  // every method gets the whole struct — interface segregation broken
    func load() async { ... }
}
```

The umbrella struct is brittle (you can't split it when the view splits) and forces every method on `Logic` to receive fields it doesn't touch. Once state outgrows a single meaningful field, **inline `@State`s on the view** and move helpers into a `<View>+Logic.swift` file as free functions. Two valid shapes — pick per function:

1. **Return a result struct.** Best for one-shot async loaders. The view assigns the result into individual `@State`s in one place.
   ```swift
   func loadResults(query: String, repo: Repository) async throws -> LoadedResults
   ```
2. **Take individual `Binding`s and mutate.** Best for interactive callbacks that touch multiple fields plus side channels (storage, analytics, navigation).
   ```swift
   func selectStation(_ s: Station, selected: Binding<Station>, destination: Binding<Destination?>, storage: StorageCapability)
   ```

Group properties into a struct only when they share real domain meaning — a result payload, a capability of related bindings, a `Repository` of operations against the same backend. Never because they happen to coexist on one screen. UI is composable; today's screen is tomorrow's two screens.

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

**Use the struct + extension split ONLY when the struct carries its own data** — at minimum one stored property, `@State`, `@Binding`, or `@Environment` field that makes it meaningfully a model. If the view has no data of its own (composes pure subviews, derives everything from `@Environment`, or just lays out static content), declare `View` conformance inline instead. An empty `struct Foo {}` followed by `extension Foo: View { ... }` is pure ceremony — it implies a model/view split that does not exist and is strictly worse than a single declaration.

```swift
// ❌ empty struct + extension — no model, no reason to split
struct HomeScreen {}

extension HomeScreen: View {
    var body: some View { /* ... */ }
}

// ✅ no data of its own — declare View inline
struct HomeScreen: View {
    var body: some View { /* ... */ }
}

// ✅ has its own state — the extension split now means something
struct HomeScreen {
    @State var selection: Tab = .feed
}

extension HomeScreen: View {
    var body: some View { /* ... */ }
}
```

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

## View init: data plumbing only

When a `View` struct needs an explicit `init`, that init exists to copy arguments into stored properties. Nothing else.

- ❌ No side effects — no network calls, persistence, analytics, logging, `Task` kick-offs, notifications.
- ❌ No initialisation of heavy collaborators or external resources (no `URLSession`, no opening files, no reading `UserDefaults`).
- ❌ No business logic — no transformations, no branching that decides what the view *is*.
- ✅ Evaluating a closure argument into a stored property is fine when the API requires it (`@ViewBuilder` content captured into a `let`, a default value computed from another arg). That *is* the data plumbing.

The struct exists to maintain the dataflow graph, nothing else. SwiftUI re-creates view structs freely on every parent re-render — anything you put in `init` therefore runs at unpredictable times and unbounded frequency. Side-effecting work belongs in `.task`, `.onAppear`, `.onChange(of:)`, a capability closure invoked from an event, or a custom `DynamicProperty.update()`.

```swift
// ❌ init is doing work
struct ProfileScreen: View {
    let user: User

    init(userID: String) {
        self.user = UserService.shared.fetchUserSync(userID)  // network in init
        Analytics.track("profile_opened", id: userID)         // side effect in init
    }

    var body: some View { /* … */ }
}

// ✅ init only assigns; work happens in .task
struct ProfileScreen: View {
    let userID: String
    @State private var user: User?
    @Environment(\.userService) private var userService

    var body: some View {
        Group {
            if let user {
                ProfileBody(user: user)
            } else {
                ProgressView()
            }
        }
        .task {
            user = try? await userService.fetchUser(userID)
        }
    }
}
```

## File organization

> **One file per view. One folder per feature. Shared code in `Common/`.**

Folders mirror the product (what the user sees on screen), not the compiler (Models/Views/ViewModels). A new developer should be able to navigate to any screen by following the UI hierarchy alone.

```
MyApp/
├── MyAppApp.swift       ← @main App struct
├── ContentView.swift    ← root TabView / NavigationStack
├── Home/                ← HomeScreen.swift + its subviews + its local models
├── Search/
├── Profile/
└── Common/              ← components/extensions/utilities used by ≥2 features
```

Hard rules:
- **Never** create `Models/`, `Views/`, or `ViewModels/` at the top level.
- A subview used by only one screen lives in that screen's folder, not in `Common/`.
- A `private struct` inside a screen file that grows → extract to its own file in the same folder, drop `private`.
- **Never create empty files.** Don't scaffold placeholder files in advance — create a file only when there is real content to put in it (a type definition, a real implementation). No empty `// TODO` files, no `Foo.swift` with just `import SwiftUI`.

See `references/file-organization.md` for the decision table, naming conventions, sub-folder rules, and a worked three-tab app example.

## App entry point

Start every new SwiftUI project with `@main struct App: App` and nothing else — no `AppDelegate.swift`, no `SceneDelegate.swift`, no `Main.storyboard`.

```swift
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}
```

Only add an `AppDelegate` when a third-party SDK or system API genuinely needs `UIApplicationDelegate` callbacks (push notifications, background fetch). Wire it in via `@UIApplicationDelegateAdaptor` — never make the AppDelegate the `@main` entry point. `SceneDelegate` is almost never needed; `Scene` handles multi-window natively.

See `references/app-entry-point.md` for the `@UIApplicationDelegateAdaptor` snippet and the cleanup steps when Xcode generates a UIKit-lifecycle project.

## Reusable components

**Reuse first. Fork last.** Before creating a new view, check whether an existing component already handles the case — even when "only the look is different." Forking is the most common source of UI duplication in SwiftUI codebases.

**Components are purely UI** — no business logic, no service dependencies, no `@Environment` reads of capabilities. They take data and closures in, render, and emit events out. A component that imports `UserService` is a screen wearing a component's clothes. Test: if you can't construct it in a `#Preview` from inline literals alone, it isn't a component.

The decision is mechanical:

| Difference between two usages | What to build |
|---|---|
| Same data, different structure/layout | Two separate components |
| Same structure, different data | One component, parameterise the data |
| Same structure and data, different visual treatment | One component, expose a **style** |

Most "we need a different look here" requests fall on the third row. Do not duplicate the component — give it a styling API following Apple's pattern (configuration struct + style protocol + environment + shorthand). The component stays one type; new looks become new styles.

### When NOT to fork

- ❌ "We need a smaller version" → `controlSize` parameter or honour `@Environment(\.controlSize)`.
- ❌ "We need it in red here" → `.tint(_:)`, or a style that reads a stored colour.
- ❌ "We need it without the icon" → `LabelStyle` (e.g. `.titleOnly`) or a configuration flag.
- ❌ "We need a different background" → custom style; component stays the same.
- ❌ "We need it laid out vertically" → custom style; the layout lives in `makeBody`, not in a fork.

Forking is the right answer only when the **structure** itself changes — different child views, fundamentally different data shape, different interaction model — not when the visuals change.

### When to extract

Extract a component to `Common/` (or a feature sub-folder when scoped) once **two or more screens** need the same combination of structure + behaviour. A component used in one place stays where it is used. Premature extraction forces an API before the second use case has revealed what is actually variable.

See `references/custom-component-styling.md` for the full styleable-component pattern: the five pieces (configuration / protocol / environment key / modifier / shorthand), the `DynamicProperty` + `resolve(_:)` trick that lets styles read `@Environment`, default style, file layout, and modal-propagation caveats.

## Reusable styles (built-in views)

For built-in styleable views (`Button`, `Toggle`, `Label`, `GroupBox`, `ProgressView`, etc.), encapsulate visual treatment as a named `*Style` value — the same way fonts or colours are reused. Inline modifier chains do not scale.

**Extract rule:** if a styleable control has more than one modifier on it or its label inline, that's a style waiting to be named. Move the chain into a `makeBody` implementation.

```swift
// ❌ intent buried under modifiers
Button(label) {}
    .font(.subheadline)
    .foregroundStyle(.indigo)
    .opacity(isPressed ? 0.7 : 1)

// ✅ intent is the name
Button(label) {}
    .buttonStyle(.link)
```

**File layout:** every custom style lives in `Common/Styles/<Element>/<Name>Style.swift`, organised by the protocol it implements:

```
Common/Styles/
├── Button/   ← CardButtonStyle, ChipButtonStyle, LinkButtonStyle
├── Toggle/   ← AppToggleStyle
└── Label/    ← IconLabelStyle
```

**File contents (in order):** doc comment → style struct → constrained `static var`/`static func` shorthand on the protocol so call sites read `.myStyle`, not `MyStyle()`. Use `static func` when the style takes parameters, `static var` otherwise. **This enum-like dot syntax is Apple's official recommendation** (since Swift 5.5 / SE-0299) — always prefer it to constructing the style type directly.

```swift
// ❌ type initializer (pre-Swift 5.5 style)
.textFieldStyle(RoundedBorderTextFieldStyle())

// ✅ enum-like shorthand (Apple's recommendation)
.textFieldStyle(.roundedBorder)
```

See `references/style-protocols.md` for the full list of customisable style protocols (`ButtonStyle`, `ToggleStyle`, …), the list of non-customisable styles (where you can only choose between Apple's concrete styles, not write your own), the `MyLabelStyle` worked example showing both `static var` and `static func` shorthands, and the rules for using `@Environment` inside a built-in style.

This skill matches what stock SwiftUI does — single `makeBody` styles applied via dot syntax. **Style composition** (modifying an existing style by wrapping it) is intentionally not part of this skill, because it isn't a pattern Apple uses in stock SwiftUI.

## Code style

Six mechanical formatting rules apply to every SwiftUI file you write or review. Read `references/code-style.md` once before editing — the rules are short but the examples matter.

1. **Always write the explicit type name** — never `.init(…)`. Type is part of the readability of every call site.
2. **Every modifier on its own line.** No `Divider().padding(...)` — the modifier goes on the next indented line.
3. **Empty line between sibling views** in any container (`VStack`/`HStack`/`ZStack`/`ForEach`). Modifiers don't count as siblings.
4. **Never inline block content.** Trailing closures always expand to multiple lines. Empty `{}` is the only exception.
5. **One parameter per line when ≥2 arguments are labelled** — initializers, modifiers, anything with multiple labelled args. Single-arg calls and purely positional calls (`min(a, b)`, `max(x, y)`, `zip(a, b)`, `print(x, y, z)`) stay on one line.
6. **Use shape aliases at modifier call sites** — `.clipShape(.capsule)`, not `.clipShape(Capsule())`. Standalone shape views in `ZStack` still use explicit types.

When writing or refactoring SwiftUI, mentally check each rule on every change. The diff should be quiet enough to scan.

## Previews

Every `View` you create has a working `#Preview` block. No exceptions.

- Use the `#Preview` macro (Xcode 15+ / iOS 17+) — never the legacy `PreviewProvider` protocol.
- The preview must compile and render without crashing or requiring runtime data the AI can't provide. Stub state, capabilities, and environment values inline.
- If the view has multiple visual variants — selected/unselected, loading/loaded/empty/error, light/dark, control sizes, locales — write **one `#Preview` per variant** and name each preview after the variant it covers.

```swift
#Preview("Default") {
    ChipButton(title: "Filter", isSelected: false) { }
}

#Preview("Selected") {
    ChipButton(title: "Filter", isSelected: true) { }
}

#Preview("Loading") {
    ActivityFeed(state: .loading)
}

#Preview("Empty") {
    ActivityFeed(state: .loaded([]))
}

#Preview("Error") {
    ActivityFeed(state: .failed(SampleError.network))
}
```

Previews are the unit of iteration: build a view by writing the preview first, watch it render, refine. A view without a preview — or with a preview that doesn't compile — is half-finished work.

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

## When classes ARE allowed

Classes (including `@Observable`) are allowed **only** for the two cases below. If neither applies, the answer is a struct, a `DynamicProperty`, or a capability — not a class.

1. **Bridging UIKit / AppKit.** `UIViewControllerRepresentable`, `UIViewRepresentable`, coordinators, `NSObject` subclasses required by delegate protocols, things that have to inherit from a Cocoa class.
2. **Features that genuinely cannot be expressed any other way.** System bridges and frameworks that hand you a reference-type API: `CLLocationManager` delegate, `AVPlayer`, `MTKView`, `WKWebView`, Combine publishers from a UIKit world, third-party SDKs whose API is class-based.

`@Observable` carries an extra constraint on top of the two conditions above. Use `@Observable` only when **all three** hold:
1. One of the two conditions above applies (UIKit bridge OR unavoidable reference-type API).
2. The class plays the role of a **model** — a data container — not a controller, coordinator, view-model, or delegate.
3. You need a lifecycle hook a struct can't give you — almost always automatic cleanup on `deinit` (releasing system resources, cancelling subscriptions, stopping a `CLLocationManager`, tearing down a `WKWebView`).

"I want auto-tracked properties" is not a reason — structs + `@State` give you that natively. If `deinit` doesn't matter, neither does `@Observable`. A class that's only a class because some Cocoa protocol forces it (e.g., a delegate) does not need `@Observable` either; mark it as a plain class and observe whatever values it produces from a struct upstream.

When a class is unavoidable, host it inside SwiftUI via `@State var x = MyClass()` (with `@Observable`, when condition 3 applies) — not via `@StateObject`. That keeps the lifecycle owned by SwiftUI's graph even though the value itself is a reference.

Never reach for a class to "organise logic", "share state across views", "make it testable", or "separate concerns". Those are all solved better — and natively — by structs, `@Environment`, custom `DynamicProperty`, or a capability struct.

### Bridging callback / delegate APIs to capabilities

The cases above (delegate protocols, completion-handler SDKs, `NotificationCenter`, frameworks like `CLLocationManager` or `CBCentralManager`) hand you class-shaped, callback-shaped APIs. Bridge them before they reach the rest of the app:

1. Wrap one-shot callbacks with `withCheckedThrowingContinuation` to expose an `async throws` method.
2. Wrap event streams with `AsyncStream` / `AsyncThrowingStream`, produced from delegate callbacks.
3. Hide the class behind a capability struct that exposes those async surfaces as closures:

```swift
struct LocationCapability {
    var requestAuthorization: () async -> CLAuthorizationStatus
    var updates: () -> AsyncStream<CLLocation>
}
```

The class is sealed off — it talks to one capability struct and nothing else. Views, other capabilities, and feature code only ever import the struct. This is what keeps the rest of the codebase value-typed and pure-MV even when one corner has to be a class because Apple says so. Choose isolation by the actual concurrency shape (`@MainActor` for UI-bound work, `actor` for genuinely concurrent inputs) — don't default to either reflexively.

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

**Architecture & state:**
- `references/state-ownership-decisions.md` — Decision tree for where to put state: `@State`, `@Binding`, `@Environment`, custom `DynamicProperty`, or a hosting class of last resort.
- `references/custom-dynamic-properties.md` — Building custom sources of truth (the `BrightnessWrapper` pattern, mutable storage in immutable structs, `update()` semantics).
- `references/swiftdata-query.md` — Dependency-keyed recreation of `@Query` (and any `DynamicProperty` whose init does real work). The `IndexedSotView` boundary that decouples source-of-truth recreation from parent recompute frequency.
- `references/dependency-injection.md` — `@Environment` + `EnvironmentKey` vs protocol/mock vs capability struct. Covers previews, tests, and production wiring.
- `references/capability-pattern.md` — Closures-as-data in depth, composition, ad-hoc polymorphism, when to use a protocol instead.
- `references/declarative-requests-networking.md` — Layering `declarative-requests-swift` into MV: Repository as a struct of `@RequestBuilder` closures, domain capability composing Repository + network closure + decoder, per-layer testing.

**Migration:**
- `references/refactoring-from-mvvm.md` — Step-by-step migrations with before/after code (counter, form, list/detail, networking-backed screen, navigation).
- `references/why-not-tca.md` — The TCA critique, mapping TCA concepts to MV equivalents, and what you lose vs gain.

**Testing:**
- `references/testing-views.md` — Testing SwiftUI views natively (hosting in a test `App`, body-evaluation assertions, ViewInspector usage) — without ViewModels.

**Project hygiene (read once per project):**
- `references/file-organization.md` — Folder layout, naming, decision table, three-tab worked example.
- `references/app-entry-point.md` — `@main App` default, `@UIApplicationDelegateAdaptor`, Xcode-template cleanup.
- `references/code-style.md` — Six mechanical formatting rules. **Read before writing or reviewing any SwiftUI code.**

**Components & styling:**
- `references/style-protocols.md` — Reusable styles for built-in views (`ButtonStyle`, `ToggleStyle`, `LabelStyle`, …): folder layout, implementation template, parameterised styles, environment access.
- `references/custom-component-styling.md` — Making your own components styleable like Apple's: five-piece pattern (configuration / protocol / environment key / modifier / shorthand), `DynamicProperty` + `resolve(_:)` trick, default style, file layout, modal-propagation caveat. Stock-SwiftUI shape only — no composable styles.
