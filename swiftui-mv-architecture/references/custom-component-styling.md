# Custom component styling

Apple's built-in styleable views (`Button`, `Toggle`, `Label`, `GroupBox`, `ProgressView`, `LabeledContent`, `DisclosureGroup`, `Form`, …) all share the same architecture. When you build your own component that has visual variants, mirror that architecture exactly — do not invent a different shape. The result is a component that:

- Is set with one modifier (`.myComponentStyle(_:)`), the same way `.buttonStyle(_:)` works.
- Can be styled at the screen, feature, or app level and propagates down the view hierarchy.
- Is locally overridable — any subtree can swap to a different style.
- Can read `@Environment` values inside the style (`isEnabled`, `controlSize`, `colorScheme`, `tint`, …).
- Composes with other styles (modify an existing style instead of rewriting it).

This file covers the full pattern: configuration, protocol, environment, modifier, shorthand, the `DynamicProperty` resolution trick, configuration initialisers, composable styles, and modal-presentation propagation.

The running example is a `RangeSlider<Label>` component. The same pattern applies to any custom view.

---

## 1. The five pieces

Every styleable component has five pieces. They are always shaped the same way — only the names change.

### 1.1 Style configuration

A struct holding the parts the style needs to render: bound values, parameters, and a type-erased label/content.

```swift
struct RangeSliderStyleConfiguration {
    @Binding var range: ClosedRange<Double>
    let bounds: ClosedRange<Double>?
    let label: Label

    /// Type-erased label so styles do not see the component's `Label` generic.
    struct Label: View {
        let underlying: AnyView
        init(_ label: some View) { underlying = AnyView(label) }
        var body: some View { underlying }
    }
}
```

**Rules:**
- Anything the style varies must be in the configuration (current value, bounds, role).
- Anything available from the environment (`isEnabled`, `colorScheme`, `controlSize`, `tint`) does **not** belong in the configuration — let the style read it directly.
- Type-erase generic content (`label`, `content`) into a nested `View` struct that wraps an `AnyView`. A nested view type — not a bare `AnyView` — leaves room to add convenience properties later (e.g. a placeholder string for a `TextField` configuration).
- Bindings stay bindings. Use `@Binding` in the configuration so styles can mutate the value (e.g. a `Toggle` style flipping `isOn`).

### 1.2 Style protocol

```swift
protocol RangeSliderStyle: DynamicProperty {
    associatedtype Body: View
    @ViewBuilder func makeBody(configuration: Configuration) -> Body
    typealias Configuration = RangeSliderStyleConfiguration
}
```

**Rules:**
- Always conform to `DynamicProperty`. This is what lets styles use `@Environment`, `@State`, or any other property wrapper inside the style itself. (See section 3.)
- `Configuration` is a `typealias` to the configuration type — same convention as `ButtonStyle.Configuration`.
- `makeBody` is `@ViewBuilder` so styles can use `if`/`switch` directly.

### 1.3 Environment key

The current style is propagated through the view tree as an environment value, exactly like `.buttonStyle` propagates a button style.

```swift
private struct RangeSliderStyleKey: EnvironmentKey {
    static var defaultValue: any RangeSliderStyle = DefaultRangeSliderStyle()
}

extension EnvironmentValues {
    var rangeSliderStyle: any RangeSliderStyle {
        get { self[RangeSliderStyleKey.self] }
        set { self[RangeSliderStyleKey.self] = newValue }
    }
}
```

The default value defines what the component looks like when no style is set — every styleable component must have one.

### 1.4 View modifier

```swift
extension View {
    func rangeSliderStyle(_ style: some RangeSliderStyle) -> some View {
        environment(\.rangeSliderStyle, style)
    }
}
```

`some RangeSliderStyle` lets callers pass any concrete style. The modifier writes it into the environment; the component reads it back.

### 1.5 Shorthand

A static member on the protocol so call sites read `.vertical`, not `VerticalRangeSliderStyle()`.

```swift
extension RangeSliderStyle where Self == VerticalRangeSliderStyle {
    static var vertical: Self { VerticalRangeSliderStyle() }
}

// Parameterised variant
extension RangeSliderStyle where Self == TaggedRangeSliderStyle {
    static func tagged(_ tag: String) -> Self {
        TaggedRangeSliderStyle(tag: tag)
    }
}
```

**Rule:** constrain `Self == ConcreteType`. Without the constraint, dot syntax cannot resolve the static member.

---

## 2. Wiring the component

The component reads the style from the environment, builds a configuration from its inputs, and renders the style's body.

```swift
struct RangeSlider<Label: View>: View {
    @Binding var range: ClosedRange<Double>
    var bounds: ClosedRange<Double>?
    var label: Label

    @Environment(\.rangeSliderStyle) private var style

    init(
        range: Binding<ClosedRange<Double>>,
        in bounds: ClosedRange<Double>? = nil,
        @ViewBuilder label: () -> Label
    ) {
        self._range = range
        self.bounds = bounds
        self.label = label()
    }

    var body: some View {
        let configuration = RangeSliderStyleConfiguration(
            range: $range,
            bounds: bounds,
            label: .init(label)
        )

        AnyView(style.resolve(configuration: configuration))
    }
}
```

`AnyView` is unavoidable here: the environment value has type `any RangeSliderStyle`, so the body's return type cannot be opaque. Accept it — the component is one type, written once. The `AnyView` cost is paid once per render of one view, not propagated.

---

## 3. The `DynamicProperty` trick — why styles need `resolve(_:)`

Styles routinely need to react to environment values: a disabled state should desaturate the control; the tint should colour the fill; a small `controlSize` should reduce padding.

The naive code looks correct:

```swift
struct DefaultRangeSliderStyle: RangeSliderStyle {
    @Environment(\.isEnabled) var isEnabled  // ← appears to work

    func makeBody(configuration: Configuration) -> some View {
        // …
        .saturation(isEnabled ? 1 : 0)
    }
}
```

It does not. SwiftUI logs a runtime warning: *"Accessing Environment's value outside of being installed on a View. This will always read the default value and will not update."* The style is not a `View`, so SwiftUI never injects environment values into it.

**Fix:** funnel the style through a tiny intermediate `View` that is generic over the concrete style type. Because the wrapper is generic (not `any RangeSliderStyle`), SwiftUI can resolve property wrappers on the style — including `@Environment`, `@State`, `@FocusState`, etc.

```swift
struct ResolvedRangeSliderStyle<Style: RangeSliderStyle>: View {
    var configuration: RangeSliderStyleConfiguration
    var style: Style

    var body: some View {
        style.makeBody(configuration: configuration)
    }
}

extension RangeSliderStyle {
    func resolve(configuration: Configuration) -> some View {
        ResolvedRangeSliderStyle(configuration: configuration, style: self)
    }
}
```

The component now calls `style.resolve(configuration:)` instead of `style.makeBody(configuration:)` directly. From the call site `self` is the concrete style type, so `ResolvedRangeSliderStyle<ConcreteStyle>` gets the property-wrapper resolution for free.

This is why every style protocol in this codebase conforms to `DynamicProperty` and every component renders styles via `resolve(_:)` — without it, `@Environment` in styles silently breaks.

---

## 4. Default style

Every component needs a default. Implement it in the same file as the protocol or in its own file under `Common/Styles/<Component>/`.

```swift
struct DefaultRangeSliderStyle: RangeSliderStyle {
    @Environment(\.isEnabled) var isEnabled

    func makeBody(configuration: Configuration) -> some View {
        GeometryReader { proxy in
            ZStack {
                Capsule()
                    .fill(.regularMaterial)
                    .frame(height: 4)

                // … track + thumbs, mutating configuration.range via $range
            }
        }
        .frame(height: 27)
        .saturation(isEnabled ? 1 : 0)
    }
}
```

---

## 5. Composable styles — modifying an existing style

Often a new visual variant is "the existing style, plus one more thing" (a different font, a custom border, a shimmer). Do not copy the existing style — modify it.

The mechanism is a **configuration initialiser** on the component itself, mirroring `Button(_:)` taking a `PrimitiveButtonStyleConfiguration`. The new style instantiates the component from its configuration and applies the standard style modifier on top:

```swift
extension RangeSlider where Label == RangeSliderStyleConfiguration.Label {
    init(_ configuration: RangeSliderStyleConfiguration) {
        self._range = configuration.$range
        self.bounds = configuration.bounds
        self.label = configuration.label
    }
}

struct CardRangeSliderStyle: RangeSliderStyle {
    func makeBody(configuration: Configuration) -> some View {
        RangeSlider(configuration)
            .padding()
            .background(.regularMaterial, in: .rect(cornerRadius: 16))
    }
}
```

Used like Apple's modified styles:

```swift
RangeSlider(range: $range, in: 0...100) { Text("Range") }
    .rangeSliderStyle(CardRangeSliderStyle())
    .rangeSliderStyle(.vertical)
```

The card style wraps the vertical style. The order matters — the *inner* (lower in the modifier chain) style runs first.

### Style stack

A naive single-environment-value implementation breaks composition: the second `.rangeSliderStyle(...)` overwrites the first, and `RangeSlider(configuration)` inside the card style would re-read the same card style and infinite-loop.

The fix is to maintain a stack of styles in the environment, push on `.rangeSliderStyle`, and pop after `makeBody` runs. Skip this until the component genuinely needs composable styles — most do not.

```swift
private struct RangeSliderStyleStackKey: EnvironmentKey {
    static var defaultValue: [any RangeSliderStyle] = []
}

extension EnvironmentValues {
    var rangeSliderStyleStack: [any RangeSliderStyle] {
        get { self[RangeSliderStyleStackKey.self] }
        set { self[RangeSliderStyleStackKey.self] = newValue }
    }

    var rangeSliderStyle: any RangeSliderStyle {
        rangeSliderStyleStack.last ?? DefaultRangeSliderStyle()
    }
}

extension View {
    func rangeSliderStyle(_ style: some RangeSliderStyle) -> some View {
        transformEnvironment(\.rangeSliderStyleStack) { $0.append(style) }
    }
}
```

In the component, pop after rendering:

```swift
AnyView(style.resolve(configuration: configuration))
    .transformEnvironment(\.rangeSliderStyleStack) { stack in
        if !stack.isEmpty { stack.removeLast() }
    }
```

This is exactly how Apple's built-in styles compose. Read the *Composable Styles* technique once, then use this skeleton when needed.

---

## 6. Reusable style modifiers across components

A modification that applies to multiple components (font override, label-style override, shimmer effect, …) belongs in a `ViewModifier`, then attached via a generic `ModifiedStyle` wrapper conforming to each style protocol it supports.

```swift
struct ModifiedStyle<Style, Modifier: ViewModifier>: DynamicProperty {
    var style: Style
    var modifier: Modifier
}

extension ModifiedStyle: RangeSliderStyle where Style: RangeSliderStyle {
    func makeBody(configuration: RangeSliderStyleConfiguration) -> some View {
        RangeSlider(configuration)
            .rangeSliderStyle(style)
            .modifier(modifier)
    }
}

extension RangeSliderStyle {
    func modifier(_ modifier: some ViewModifier) -> some RangeSliderStyle {
        ModifiedStyle(style: self, modifier: modifier)
    }
}
```

Then a single `ViewModifier` (e.g. `FontModifier(font: …)`) can be applied to any styleable component via `.someStyle.modifier(FontModifier(font: .rounded))`.

---

## 7. Style propagation into sheets and modals

SwiftUI does not propagate style environment values across all modal presentation boundaries (`.sheet`, `.fullScreenCover`, `.popover` outside a `NavigationStack`). On those platforms, restate the styles inside the presented view, or wrap the presenter in a `UIHostingController`-backed `UIViewControllerRepresentable` that re-injects them.

For most apps, the practical answer is: apply styles at the root (`ContentView`) **and** restate them on each sheet's root view if that sheet uses the styled components. Treat sheet content as a fresh top of the tree.

---

## 8. File layout

Custom-component styling files live next to the component or under `Common/Styles/`, mirroring the rules in the main skill:

```
Common/
├── RangeSlider/
│   ├── RangeSlider.swift                    ← component
│   ├── RangeSliderStyleConfiguration.swift  ← configuration + Label
│   ├── RangeSliderStyle.swift               ← protocol + environment + modifier + resolve
│   └── Styles/
│       ├── DefaultRangeSliderStyle.swift
│       ├── VerticalRangeSliderStyle.swift
│       └── CardRangeSliderStyle.swift
```

Shorthand `static var`/`static func` extensions live in the same file as the concrete style they expose — never in a shared "shorthands" file. The rule is: the file that defines `VerticalRangeSliderStyle` is also the file that defines `.vertical`.

---

## 9. Decision checklist

Before adding a new component or style, walk the checklist:

1. Is there an existing component with the same data and structure? → Add a style, do not fork.
2. Is the difference reachable through `@Environment` (`controlSize`, `tint`, `colorScheme`)? → Honour the environment, do not add a parameter.
3. Does the new style modify an existing style (font tweak, wrapper, decoration)? → Configuration initialiser + composable style, not a copy.
4. Is the modification useful across multiple components (font, label style, shimmer)? → `ViewModifier` + `ModifiedStyle` wrapper, not per-component duplication.
5. Is the component used in only one place today? → Keep it inline; extract when the second use lands.

If a fork still feels right after walking the checklist, the structural difference is real — fork the component, not the style.
