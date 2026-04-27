# Style protocols for built-in SwiftUI views

Use this file when extracting visual treatment for a built-in styleable view (`Button`, `Toggle`, `Label`, etc.) into a reusable `*Style` type. For making your own custom components styleable, see `custom-component-styling.md` instead.

This skill matches what stock SwiftUI does: one `makeBody`, applied through enum-like dot syntax. Style **composition** (wrapping an existing style with a modifier) is intentionally not part of this skill — Apple does not use that pattern in stock SwiftUI.

## When to extract

Extract to `Common/Styles/` as soon as a visual treatment:
- Is used in more than one place, OR
- Has more than one modifier on the view it styles, OR
- Obscures what the control *is* rather than showing what it *does* (a chain of five modifiers tacked on a `Button` is visual noise; a named style is self-documenting).

```swift
// ❌ intent buried under modifiers — what kind of button is this?
Button(label) {}
    .font(.subheadline)
    .foregroundStyle(.indigo)
    .opacity(isPressed ? 0.7 : 1)

// ✅ intent is the name
Button(label) {}
    .buttonStyle(.link)
```

## Where styles live

Styles are grouped by element type, not alphabetically or by feature. A new `ToggleStyle` always goes into `Toggle/`, regardless of which screen it first appears on.

```
Common/
└── Styles/
    ├── Button/
    │   ├── CardButtonStyle.swift
    │   ├── ChipButtonStyle.swift
    │   └── LinkButtonStyle.swift
    ├── Toggle/
    │   └── AppToggleStyle.swift
    └── Label/
        └── IconLabelStyle.swift
```

## Customisable styles (with `makeBody`)

These protocols expose `makeBody(configuration:)`. Conform a struct to one of them to write your own style.

- `ButtonStyle`
- `ControlGroupStyle`
- `DatePickerStyle`
- `DisclosureGroupStyle`
- `FormStyle`
- `GaugeStyle`
- `GroupBoxStyle`
- `LabeledContentStyle`
- `LabelStyle`
- `MenuStyle`
- `NavigationSplitViewStyle`
- `PrimitiveButtonStyle`
- `ProductViewStyle`
- `ProgressViewStyle`
- `SubscriptionStoreControlStyle`
- `TableStyle`
- `TextEditorStyle`
- `ToggleStyle`

## Non-customisable styles (no `makeBody`)

These protocols **do not expose `makeBody`** — you cannot write your own style for these views. Choose between Apple's concrete styles (e.g. `.plain`, `.bordered`, `.inset`).

- `AccessibilityQuickActionStyle`
- `IndexViewStyle`
- `ListStyle`
- `MenuBarExtraStyle`
- `MenuButtonStyle`
- `NavigationViewStyle`
- `PickerStyle`
- `ShapeStyle`
- `SubscriptionOptionGroupStyle`
- `TabViewStyle`
- `TextFieldStyle`
- `WindowStyle`
- `WindowToolbarStyle`

If a design ask requires customising one of these (e.g. a fully custom list cell layout), the answer is to compose your own view from scratch — not to fight the protocol.

## Enum-like style shorthand (Apple's official recommendation)

Since Swift 5.5 (introduced at WWDC '21), styles use [SE-0299: Extending Static Member Lookup in Generic Contexts](https://github.com/apple/swift-evolution/blob/main/proposals/0299-extend-generic-static-member-lookup.md) to expose enum-like dot syntax. **The official Apple documentation explicitly recommends using these enum-like styles instead of constructing the style type directly.**

```swift
struct ExampleView: View {
    @State var name = ""

    var body: some View {
        // ❌ Before Swift 5.5 — type initializer
        TextField("", text: $name)
            .textFieldStyle(RoundedBorderTextFieldStyle())

        // ✅ Starting with Swift 5.5 — enum-like shorthand
        TextField("", text: $name)
            .textFieldStyle(.roundedBorder)
    }
}
```

Because the change lives in the language, enum-like styles work on any OS that already has the underlying style type — they are not gated by deployment target.

Apply the same pattern to your own styles:
- **`static var`** for styles that take no parameters.
- **`static func`** for styles that take parameters.

## Implementation pattern

Every style file contains three things in this order:

1. A doc comment explaining what the style does and when to use it.
2. The style struct conforming to the protocol.
3. Constrained `static var` and/or `static func` extensions on the protocol so the style can be used with dot syntax.

```swift
// Common/Styles/Label/MyLabelStyle.swift
import SwiftUI

/// Vertical label with a coloured circular background.
/// Use for badge-style labels in icon grids.
struct MyLabelStyle: LabelStyle {
    let color: Color

    init(color: Color = .green) {
        self.color = color
    }

    func makeBody(configuration: Configuration) -> some View {
        VStack(spacing: 10) {
            configuration.title
                .minimumScaleFactor(0.2)
                .lineLimit(1)

            configuration.icon
        }
        .frame(width: 100, height: 100)
        .padding(20)
        .background(Circle().fill(color))
    }
}

extension LabelStyle where Self == MyLabelStyle {
    /// Default `MyLabelStyle` (green background).
    static var myLabelStyle: MyLabelStyle {
        MyLabelStyle()
    }

    /// `MyLabelStyle` with a custom background colour.
    static func myLabelStyle(_ color: Color) -> MyLabelStyle {
        MyLabelStyle(color: color)
    }
}
```

Call sites use the dot syntax for both forms:

```swift
HStack {
    // ✅ enum-like, no parameter
    Label("Club", systemImage: "suit.club.fill")
        .labelStyle(.myLabelStyle)

    // ✅ enum-like, with parameter
    Label("Spade", systemImage: "suit.spade.fill")
        .labelStyle(.myLabelStyle(.yellow))
}
```

The constrained extension (`where Self == MyLabelStyle`) is required so Swift resolves the dot syntax without ambiguity. Apply this same shape to every style protocol — `ButtonStyle`, `ToggleStyle`, `GroupBoxStyle`, etc.

## Reading the environment inside a style

Styles applied to **built-in** SwiftUI views (the customisable protocols listed above) can use `@Environment` directly inside `makeBody` — Apple installs the style as a `View` internally, so property wrappers resolve correctly.

```swift
struct FadeWhenDisabledButtonStyle: ButtonStyle {
    @Environment(\.isEnabled) var isEnabled

    func makeBody(configuration: Configuration) -> some View {
        configuration.label
            .opacity(isEnabled ? 1 : 0.4)
    }
}
```

This is a critical difference from custom components: when YOU build a styleable component, you must use the `DynamicProperty` + `resolve(_:)` trick from `custom-component-styling.md` for environment values to work. Built-in styles do not need it.
