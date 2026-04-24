# State Ownership — A Decision Tree

"Where does this state live?" is the first question in any SwiftUI architecture decision. Get this right and the rest follows.

## The question to ask

For every piece of state, ask: **what determines when this state is created and destroyed?**

The answer tells you where it belongs.

## Decision tree

```
Is this state a function of other state I already have?
├── YES → Don't store it. Make it a computed property.
└── NO → continue

Does exactly one view need it, and only while that view exists?
├── YES → @State on that view. Done.
└── NO → continue

Does a parent own it, and a child need read/write access?
├── YES → Parent has @State, child has @Binding. Done.
└── NO → continue

Does a subtree of views need it (not just one child)?
├── YES → @Environment with an EnvironmentKey. Done.
└── NO → continue

Does it come from an external system (disk, network, device state, system framework)?
├── YES → Custom DynamicProperty wrapping the system. Done.
└── NO → continue

Is it genuinely global for the whole app lifetime?
├── YES → @State var x = SomeObservableClass() at the root, injected down.
└── NO → You probably have one of the above and didn't realize it.
```

## Case by case

### Computed, not stored

```swift
struct State {
    var firstName: String = ""
    var lastName: String = ""
    // DON'T: var fullName: String = ""  (stored, can desync)
    // DO:
    var fullName: String { "\(firstName) \(lastName)".trimmingCharacters(in: .whitespaces) }
}
```

Stored state is a liability — it can get out of sync. If something is derivable, derive it.

### `@State` for view-local

```swift
struct EditProfile: View {
    @State private var draft = ProfileDraft()   // lives for this screen
    // ...
}
```

`@State`:
- Created once when the view first appears in the graph.
- Preserved across body re-evaluations.
- Torn down when the view is removed from the graph.
- Each instance of the view has its own.

This is the default. Start here.

### `@State` + `@Binding` for parent/child

```swift
struct Parent: View {
    @State private var draft = ProfileDraft()
    var body: some View {
        ChildEditor(draft: $draft)
    }
}

struct ChildEditor: View {
    @Binding var draft: ProfileDraft
    var body: some View {
        TextField("Name", text: $draft.name)
    }
}
```

The parent owns the state. The child gets read/write access via the `Binding`. The child has no `@State` for `draft` — if it did, there would be two sources of truth.

### `@Environment` for shared-in-subtree

```swift
private struct ThemeKey: EnvironmentKey { static let defaultValue = Theme.default }
extension EnvironmentValues {
    var theme: Theme { get { self[ThemeKey.self] } set { self[ThemeKey.self] = newValue } }
}

// Inject at the subtree root
SettingsFlow().environment(\.theme, .dark)

// Consume anywhere inside
struct AnyDeepChild: View {
    @Environment(\.theme) private var theme
    var body: some View { Text("hi").foregroundStyle(theme.foreground) }
}
```

`@Environment` scales with the tree: override at any node, consume at any descendant, with zero plumbing in between.

### Custom `DynamicProperty` for external systems

See `custom-dynamic-properties.md`. Use when:
- State reflects something outside your app (device brightness, network reachability, Bluetooth state).
- You want a `Binding` that writes back through to the external system.
- You need `@Environment` and other wrappers to keep working inside your state container.

### `@Observable` class at the root for app-wide

```swift
@Observable
final class AppSession {
    var user: User?
    var cart: [Product] = []
}

@main
struct MyApp: App {
    @State private var session = AppSession()   // hosted by @State at root
    var body: some Scene {
        WindowGroup {
            RootView().environment(session)
        }
    }
}
```

Rules for this escape hatch:

1. **One at most per concern** (e.g., one `AppSession`, not one per screen). If you find yourself writing `FeatureXSession`, `FeatureYSession`, you have confused "app-wide" with "screen-scoped" — those are `@State`s.
2. **Host with `@State`** at the composition root so SwiftUI manages lifecycle. Do not use `@StateObject`. Do not construct it as a singleton and pass the singleton down.
3. **Inject via `.environment(session)`** (modern) or as an init parameter. Do not use `@EnvironmentObject` (legacy, pre-`@Observable`).
4. **Only state, no behavior.** Put stored properties on the class; put mutating logic in separate stateless `Logic` structs that take the session as a dependency. This keeps the class as a pure source of truth, not a "god ViewModel".
5. **Prefer `@Environment` values** (structs of closures with safe defaults) for cross-cutting *services* (auth, analytics, API). Reserve the `@Observable` root class for cross-cutting *state*.

## Anti-patterns in state ownership

### ❌ `@State var vm = SomeObservableClass()` per screen

You have reinvented `@StateObject`/VM ownership. Each screen now owns a class with hidden state and lifecycle. Use `@State var state = SomeStruct()` instead, or if you genuinely need a class, think hard about whether it should be at the root instead.

### ❌ Multiple sources of truth for the same value

```swift
// BROKEN
struct Parent: View {
    @State private var text = ""
    var body: some View {
        Child()   // child also has its own @State var text
    }
}
```

If both parent and child hold `@State text`, they desynchronize on first interaction. Use `@Binding` or lift to environment.

### ❌ Global mutable singleton

```swift
class Cart { static let shared = Cart(); var items: [Product] = [] }
```

- Not preview-friendly (can't override per-preview).
- Not test-friendly (shared across tests).
- No compile-time enforcement that callers provide a cart.
- Not observable by SwiftUI without adding `@Observable` — at which point you might as well host with `@State` at the root.

### ❌ State in the view struct as a stored (non-wrapped) property

```swift
// BROKEN - this doesn't persist
struct MyView: View {
    var counter = 0   // reset on every body evaluation
    var body: some View { Button("\(counter)") { counter += 1 } }  // won't even compile
}
```

Views are re-created constantly. Only `DynamicProperty`-wrapped properties survive. Stored non-wrapped properties are *inputs* (configuration passed in), not state.

## The value-types-first principle

Prefer, in order:
1. Nothing (derive it).
2. `@State` holding a struct.
3. `@Binding`.
4. `@Environment` value.
5. Custom `DynamicProperty`.
6. `@State` holding an `@Observable` class at the composition root.

The further you go down this list, the more reasons you need. "I need a shared thing" is usually solved at level 3 or 4, not level 6.
