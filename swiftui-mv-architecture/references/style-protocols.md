# Style protocols for built-in SwiftUI views

Use this file when extracting visual treatment for a built-in styleable view (`Button`, `Toggle`, `Label`, etc.) into a reusable `*Style` type. For making your own custom components styleable, see `custom-component-styling.md` instead.

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

## Style protocols with a customisable `makeBody`

| Protocol | Customises |
|---|---|
| `ButtonStyle` | Pressable controls with a label |
| `PrimitiveButtonStyle` | Full gesture control (custom trigger logic) |
| `ToggleStyle` | On/off controls |
| `LabelStyle` | Icon + title pairs |
| `ProgressViewStyle` | Determinate and indeterminate progress |
| `GaugeStyle` | Scalar value indicators |
| `GroupBoxStyle` | Labelled container boxes |
| `DisclosureGroupStyle` | Expandable sections |
| `MenuStyle` | Drop-down menus |
| `ControlGroupStyle` | Related control clusters |
| `FormStyle` | Form layout containers |
| `LabeledContentStyle` | Label + content pairs |
| `NavigationSplitViewStyle` | Sidebar/detail split |
| `DatePickerStyle` | Date and time pickers |
| `TextEditorStyle` | Multi-line text editors |
| `TableStyle` | Data tables |
| `ProductViewStyle` | StoreKit product views |
| `SubscriptionStoreControlStyle` | StoreKit subscription controls |

## Implementation pattern

Every style file contains three things in this order:

1. A doc comment explaining what the style does and when to use it
2. The style struct conforming to the protocol
3. A shorthand static extension so the style can be used with dot syntax — the same way built-in styles like `.plain` or `.bordered` work

```swift
// Common/Styles/Button/CardButtonStyle.swift
import SwiftUI

/// Full-width card button: white background, rounded corners, subtle shadow.
/// Use for standalone actions that sit at the bottom of a scroll view
/// (e.g. Sign Out, Save, Continue).
struct CardButtonStyle: ButtonStyle {
    func makeBody(configuration: Configuration) -> some View {
        configuration.label
            .frame(maxWidth: .infinity)
            .background(Color(.systemBackground))
            .clipShape(.rect(cornerRadius: 16))
            .shadow(
                color: .black.opacity(0.04),
                radius: 6,
                x: 0,
                y: 2
            )
            .opacity(configuration.isPressed ? 0.7 : 1)
    }
}

extension ButtonStyle where Self == CardButtonStyle {
    /// Full-width card button. See ``CardButtonStyle``.
    static var card: CardButtonStyle { CardButtonStyle() }
}
```

The shorthand extension constrains `Self` to the concrete style type so Swift resolves the dot syntax without ambiguity. The call site becomes:

```swift
Button { } label: {
    Text("Sign Out")
        .font(.subheadline.bold())
        .foregroundStyle(.red)
        .padding(.vertical, 16)
}
.buttonStyle(.card)
```

Apply the same pattern to every other style protocol — `ToggleStyle`, `LabelStyle`, etc. — with a `static var` or `static func` (when the style takes parameters) on a constrained extension of the protocol.

## Parameterised styles — `static func`, not `static var`

When the visual treatment depends on runtime data (e.g. a selected/unselected chip), pass the data as a stored property and expose it via `static func`:

```swift
struct ChipButtonStyle: ButtonStyle {
    let isSelected: Bool

    func makeBody(configuration: Configuration) -> some View {
        configuration.label
            .padding(.horizontal, 12)
            .padding(.vertical, 6)
            .background(isSelected ? Color.accentColor : .clear)
            .foregroundStyle(isSelected ? .white : .primary)
            .clipShape(.capsule)
    }
}

// parameterised shorthand — static func, not static var
extension ButtonStyle where Self == ChipButtonStyle {
    static func chip(isSelected: Bool) -> ChipButtonStyle {
        ChipButtonStyle(isSelected: isSelected)
    }
}

// call site reads like natural language
.buttonStyle(.chip(isSelected: isSelected))
```

## Reading the environment inside a style

Styles applied to **built-in** SwiftUI views (the ones in the table above) can use `@Environment` directly inside `makeBody` — Apple installs the style as a `View` internally, so property wrappers resolve correctly.

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
