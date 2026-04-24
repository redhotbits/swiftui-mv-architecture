# Custom `DynamicProperty` — Building Sources of Truth

`DynamicProperty` is the framework's extension point for adding new kinds of state to SwiftUI. If you ever feel the need to put state in a class, ask first: *can this be a `DynamicProperty` instead?* Almost always, yes.

## Why `DynamicProperty`, not a class

A `DynamicProperty` is a struct that SwiftUI installs into its dependency graph. That means:

- SwiftUI owns its lifecycle (initialization, retention, teardown).
- You can use `@State`, `@Environment`, `@AppStorage`, and other `DynamicProperty`s **inside it** — they all work because you are still in a `DynamicProperty` context.
- It can expose a `wrappedValue` (what `self.x` gives you) and a `projectedValue` (what `$x` gives you — typically a `Binding`, but can be any pure-data abstraction).
- The `update()` method is called by the framework before each evaluation — this is your hook for doing setup that depends on the current graph state.

A class, in contrast, sits *outside* this system. You manage its lifetime. `@Environment` inside it will not resolve. `@AppStorage` inside it will not persist. You gain nothing and lose everything.

## Anatomy of a custom `DynamicProperty`

Three ingredients:

1. **`@State`** (or another `DynamicProperty`) to hold the actual data — because the outer struct is immutable per-evaluation, you need a mutable cell.
2. **A class for side-effect resources** — subscriptions, timers, anything with reference identity that must survive body recomputation. This is the *only* legitimate use of a class in this pattern, and it holds no user-facing state.
3. **`update()`** — idempotent setup. Called on every evaluation. Must no-op after the first successful setup.

### Template

```swift
@propertyWrapper
struct MySource: DynamicProperty {
    @State private var value: MyValue = .initial
    private let storage = Storage()

    mutating func update() {
        storage.setupIfNeeded(binding: $value)
    }

    var wrappedValue: MyValue { value }

    var projectedValue: Binding<MyValue> {
        Binding(
            get: { value },
            set: { newValue in
                value = newValue
                // forward to the external system here if needed
            }
        )
    }

    private final class Storage {
        private var cancellable: AnyCancellable?

        func setupIfNeeded(binding: Binding<MyValue>) {
            guard cancellable == nil else { return }
            cancellable = externalPublisher.sink { newValue in
                binding.wrappedValue = newValue
            }
        }
    }
}
```

The `Storage` class is a deliberately narrow escape hatch: SwiftUI structs are re-created all the time, but `@State`-backed properties are stable across re-creations, and so is anything you hang off a `@State`-held reference. That is why `Storage` can safely own a subscription.

## Worked example — two-way bound brightness

External system (`ExternalDisplay`) has `currentBrightness`, `setBrightness(_:)`, and a `brightnessPublisher`. We want a SwiftUI-native source of truth that:

- Gives views a `Binding<Double>` they can wire to a `Slider`.
- Writes back to the external system when the view mutates the binding.
- Reflects external changes (e.g., system slider moved elsewhere) automatically.

```swift
@propertyWrapper
struct BrightnessWrapper: DynamicProperty {
    @State private var brightness: Double = ExternalDisplay.currentBrightness
    private let storage = Storage()

    mutating func update() {
        storage.setupIfNeeded(binding: $brightness)
    }

    var wrappedValue: Double { brightness }

    var projectedValue: Binding<Double> {
        Binding(
            get: { brightness },
            set: { newValue in
                brightness = newValue
                ExternalDisplay.setBrightness(newValue)
            }
        )
    }

    private final class Storage {
        private var cancellable: AnyCancellable?
        func setupIfNeeded(binding: Binding<Double>) {
            guard cancellable == nil else { return }
            cancellable = ExternalDisplay.brightnessPublisher.sink { newValue in
                binding.wrappedValue = newValue
            }
        }
    }
}

struct BrightnessSlider: View {
    @BrightnessWrapper var brightness
    var body: some View {
        VStack {
            Text("Brightness: \(Int(brightness * 100))%")
            Slider(value: $brightness, in: 0...1)
        }
    }
}
```

Note:

- The view uses `brightness` (wrapped) to read and `$brightness` (projected `Binding`) to write. It never touches `BrightnessWrapper` as a whole — it only depends on the *abstraction* (`Binding<Double>`), not the implementation.
- There is no `didSet`, no observer chain, no bidirectional sync logic scattered across a `ViewModel`. The getter/setter inside `projectedValue` is the *entire* mutation surface.
- `Storage` is the only class — and it contains zero user-facing state. Replace it with any other reference-type resource (`NSObject` KVO, `NotificationCenter` token, async task, etc.) as needed.

## Views should depend on the projection, not the wrapper

`@BrightnessWrapper var brightness` is an implementation detail. A reusable sub-view should not require a specific property wrapper — it should accept the *projection*:

```swift
// Reusable, testable, preview-friendly
struct BrightnessSliderView: View {
    @Binding var brightness: Double
    var body: some View {
        VStack {
            Text("Brightness: \(Int(brightness * 100))%")
            Slider(value: $brightness, in: 0...1)
        }
    }
}

// Composition root wires the real wrapper in
struct RootView: View {
    @BrightnessWrapper var brightness
    var body: some View { BrightnessSliderView(brightness: $brightness) }
}

// Previews/tests inject a plain binding
#Preview {
    @Previewable @State var b = 0.5
    BrightnessSliderView(brightness: $b)
}
```

This is the `Binding<T>` analogue of dependency injection: the sub-view depends on an abstraction (the projection), and any source of truth — real, preview, or test — can supply it.

## Checklist for a new `DynamicProperty`

- [ ] Struct, not class.
- [ ] Storage cell is `@State` (or another `DynamicProperty`), not a property of a class.
- [ ] Any side-effect resources live in a nested private class; that class holds *no* user-facing state.
- [ ] `update()` is idempotent (uses `guard … == nil` or similar).
- [ ] `projectedValue` is a value type — usually `Binding`, sometimes a struct of closures, never an object reference.
- [ ] Consumers depend on the projection, not the wrapper itself.
- [ ] Works in previews without extra scaffolding (the wrapper either supplies a sensible default or the view accepts the projection directly).
