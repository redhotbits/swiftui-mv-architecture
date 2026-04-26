# SwiftUI code style

Mechanical formatting rules. Apply when writing or reviewing any SwiftUI code. None of these are subjective — every rule has a single right answer.

## Rule index

1. Always write the explicit type name — never `.init`
2. Every modifier on its own line
3. Empty line between sibling views
4. Never inline block content
5. One parameter per line for multi-argument calls
6. Use shape aliases — never instantiate shape types directly in modifiers

---

## 1. Always write the explicit type name — never `.init`

Use the full type name when constructing instances. Never use `.init(...)` shorthand.

```swift
// ❌ .init hides what type is being created
private let settings: [SettingsRow] = [
    .init(icon: "bell.fill", title: "Notifications", ...),
    .init(icon: "lock.fill", title: "Privacy", ...),
]

// ✅ type is immediately readable at each call site
private let settings: [SettingsRow] = [
    SettingsRow(icon: "bell.fill", title: "Notifications", ...),
    SettingsRow(icon: "lock.fill", title: "Privacy", ...),
]
```

This matters most when:
- Multiple types appear in the same file (e.g. `SettingsRow` and `BadgeItem` both initialized in `ProfileScreen`)
- The array type annotation is far from the initializer call
- Reading a diff or code review where the surrounding type context isn't visible

The only acceptable exception is a `let` binding where the type is written explicitly right beside it:

```swift
let header: SectionHeader = .init(title: "Recent Activity", actionLabel: "See all")
//                           ^^^^ type is right there — readable
```

Even then, prefer `SectionHeader(title: ...)` — it is strictly more informative.

## 2. Every modifier on its own line

Never chain a modifier on the same line as the view it modifies. Put each modifier on the next line, indented one level relative to the view.

```swift
// ❌ modifier buried on the same line
Divider().padding(.leading, 62)
Capsule().fill(.white.opacity(0.25))

// ✅ modifier on its own line
Divider()
    .padding(.leading, 62)

Capsule()
    .fill(.white.opacity(0.25))
```

This applies everywhere: inside `ForEach` closures, `ZStack` layers, conditional branches — no exceptions.

## 3. Empty line between sibling views

Always put one empty line between distinct view components that are siblings inside a container (`VStack`, `HStack`, `ZStack`, `ForEach`, etc.). Modifiers are part of the view above them — they do not count as a sibling.

```swift
// ❌ siblings run together, hard to scan
VStack {
    Text("Good morning,")
        .font(.callout)
        .foregroundStyle(.secondary)
    Text("Mirko")
        .font(.largeTitle.bold())
    Text("3 tasks pending today")
        .font(.subheadline)
        .foregroundStyle(.secondary)
}

// ✅ each component is visually distinct
VStack {
    Text("Good morning,")
        .font(.callout)
        .foregroundStyle(.secondary)

    Text("Mirko")
        .font(.largeTitle.bold())

    Text("3 tasks pending today")
        .font(.subheadline)
        .foregroundStyle(.secondary)
}
```

The same rule applies inside `HStack` and `ZStack`:

```swift
ZStack {
    Circle()
        .fill(Color.indigo)
        .frame(width: 90, height: 90)

    Text("MT")
        .font(.title.bold())
        .foregroundStyle(.white)
}
```

## 4. Never inline block content

A trailing closure or block body must always expand to multiple lines — the opening brace on the same line as the call, the content indented, the closing brace on its own line. This applies everywhere: `body`, `.tabItem`, `Button`, `ForEach`, `VStack`, modifiers, anything that takes a closure.

```swift
// ❌ block content crammed onto one line
.tabItem { Label("Search", systemImage: "magnifyingglass") }

Button("Sign Out") { viewModel.signOut() }

// ✅ always expand
.tabItem {
    Label("Search", systemImage: "magnifyingglass")
}

Button("Sign Out") {
    viewModel.signOut()
}
```

Empty blocks are the only exception — `{}` on one line is fine when the body is intentionally empty (e.g., a no-op button placeholder).

## 5. One parameter per line when ≥2 arguments are labelled

When a call has **two or more labelled** arguments, put each argument on its own line. The closing `)` goes on its own line at the original indentation level.

Single-argument calls and **positional / unlabelled calls stay on one line** — line-breaking unlabelled args removes the only visual cue to argument order and makes the call harder, not easier, to read.

```swift
// ❌ multiple labelled arguments crammed on one line
SettingsRow(icon: "bell.fill", title: "Notifications", subtitle: "Push, email", iconColor: .red)

.shadow(color: .black.opacity(0.05), radius: 10, x: 0, y: 2)

// ✅ labelled — one argument per line
SettingsRow(
    icon: "bell.fill",
    title: "Notifications",
    subtitle: "Push, email",
    iconColor: .red
)

.shadow(
    color: .black.opacity(0.05),
    radius: 10,
    x: 0,
    y: 2
)

// ✅ positional / unlabelled — stays on one line
min(width, maxWidth)
max(value, 0)
zip(items, indices)
print("Loaded", items.count, "items")
Image("logo")
```

The rule applies to any call with two or more labelled args — `Label`, `LinearGradient`, `Image(systemName:)` paired with other labelled args, view modifiers, plain initializers. It does not apply to free functions whose API is purely positional (`min`, `max`, `zip`, `print`, etc.).

## 6. Use shape aliases — never instantiate shape types directly in modifiers

SwiftUI 17+ exposes static shorthand members for the built-in shapes. Use them in any modifier that accepts a `Shape` (`.clipShape`, `.contentShape`, etc.). Never construct `Capsule()`, `Rectangle()`, `RoundedRectangle(...)`, or `Circle()` at call sites where an alias is available.

```swift
// ❌ verbose, noisy
.clipShape(Capsule())
.contentShape(Rectangle())
.clipShape(RoundedRectangle(cornerRadius: 16))
.clipShape(Circle())

// ✅ concise aliases
.clipShape(.capsule)
.contentShape(.rect)
.clipShape(.rect(cornerRadius: 16))
.clipShape(.circle)
```

This rule applies only to modifier call sites. Standalone shape views used as drawing primitives inside `ZStack` or as `View` conformers still use explicit types (`Circle()`, `RoundedRectangle(cornerRadius: x)`).
