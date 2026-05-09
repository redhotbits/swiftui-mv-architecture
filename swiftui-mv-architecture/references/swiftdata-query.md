# SwiftData `@Query` — Dependency-Keyed Recreation

`@Query` is a source of truth. So are the parameters that built it (filter, sort, animation). SwiftData enforces an awkward consequence: once a `Query` is constructed, you do not mutate its filter/sort in place — you recreate it. That makes `@Query`-backed views a dependency-flow problem, not just a data-flow problem.

The same shape applies to any `DynamicProperty` whose `init` does real work and depends on parent state — paginated fetchers, configured publishers, subscription wrappers. `@Query` is the canonical example because SwiftData is strict about it.

Based on Lazar Otasevic, *SwiftUI data flow pattern: The (only) proper way to use SwiftData Query*.

## When you actually need this pattern

This document is about the **non-trivial parent** case. If your screen has one `@Query` and a stable parent, the inline form is correct — do not reach for the helpers below:

```swift
struct ItemsView: View {
    @Query(sort: \.id) private var items: [Item]
    var body: some View { RawItems(items: items) }
}
```

You earn the pattern when one or more apply:

- The parent recomputes for many unrelated reasons (large screen with shared state).
- The `Query` parameters depend on parent state (counter, filter chip, accepted snapshot).
- Constructing the `Query` is genuinely expensive — it wires a fetch pipeline; recreation can mean real SQL work.

If none of those apply, write `@Query` inline and move on.

## The wrong shape

The instinct is to parameterise a child's `init` with parent state and build the Query there:

```swift
// ❌ couples Query construction to parent's recompute shape
struct ItemsView: View {
    @Query private var items: [Item]
    init(counter: Int) {
        _items = Query(
            filter: #Predicate<Item> { $0.id <= "\(counter)" },
            sort: \.id
        )
    }
    var body: some View { RawItems(items: items) }
}

// parent
ItemsView(counter: counter)
```

This works in a counter app and breaks in a real screen. Every parent recompute reconstructs the child, which reconstructs the Query, which re-wires the fetch pipeline — for *any* parent recompute, not just changes to `counter`. The boundary is wrong: Query recreation is gated by parent evaluation shape instead of by the dependency that actually changes the result.

## The pattern: gate by an explicit equatable key

Three small helper views compose into a boundary that recreates the source of truth only when its dependencies change.

### 1. Host the property

```swift
struct PropertyHostView<Property, Content: View>: View {
    let property: Property

    @ViewBuilder let content: (Property) -> Content

    var body: some View {
        content(property)
    }
}
```

A generic shell that holds a value and hands it to a builder closure.

### 2. Gate by an `Equatable` index

```swift
struct EquatableByParameterView<Index: Equatable, Content: View>: View, Equatable {
    static func == (lhs: Self, rhs: Self) -> Bool { lhs.index == rhs.index }

    let index: Index

    @ViewBuilder let content: () -> Content

    var body: some View {
        content()
    }
}
```

The trick: `View` conformance + `Equatable` + `.equatable()` modifier tells SwiftUI to skip body re-evaluation while `index` is unchanged. The `content` closure runs only when `index` changes.

### 3. Compose into a Query boundary

```swift
struct IndexedSotView<Index: Equatable, Property, Content: View>: View {
    let index: Index
    let property: () -> Property

    @ViewBuilder let content: (Property) -> Content

    var body: some View {
        EquatableByParameterView(index: index) {
            PropertyHostView(property: property(), content: content)
        }
        .equatable()
    }
}
```

The shape that matters: `property()` runs again **only when `index` changes**. Source-of-truth recreation is now gated by one explicit dependency, not by parent evaluation frequency.

## Real usage

Static query (no dependencies):

```swift
IndexedSotView(index: 0) {
    Query<Item, [Item]>(
        sort: \.id,
        animation: .easeInOut
    )
} content: { query in
    QueryCapableView(items: query.wrappedValue)
}
```

Query keyed by `counter`:

```swift
IndexedSotView(index: counter) {
    Query<Item, [Item]>(
        filter: #Predicate { $0.id <= "\(counter)" },
        sort: \.id,
        animation: .easeInOut
    )
} content: { query in
    QueryCapableView(items: query.wrappedValue)
}
```

Query keyed by an accepted snapshot, conditionally rendered:

```swift
if let accepted {
    IndexedSotView(index: accepted) {
        Query<Item, [Item]>(
            filter: #Predicate { $0.id <= accepted },
            sort: \.id,
            animation: .easeInOut
        )
    } content: { query in
        QueryCapableView(items: query.wrappedValue)
    }
}
```

`QueryCapableView` is a leaf that takes the resolved `[Item]` — never the `Query` itself. Pure-data, preview-friendly, testable without SwiftData.

## Gotcha — the index must cover all closure dependencies

`index` is the invalidation key for the **entire** subtree, not just for Query construction. If the `content` closure reads parent-level state that isn't in `index`, the subtree may be considered equal and won't refresh when that state changes.

```swift
// ❌ theme is captured by content but not in index — subtree won't re-render on theme change
IndexedSotView(index: accepted) {
    Query<Item, [Item]>(filter: #Predicate { $0.id <= accepted })
} content: { query in
    ChildView(items: query.wrappedValue, theme: theme)
}

// ✅ aggregate every captured dependency into the index
struct IndexKey: Equatable {
    let accepted: AcceptedSnapshot
    let theme: Theme
}

IndexedSotView(index: IndexKey(accepted: accepted, theme: theme)) {
    Query<Item, [Item]>(filter: #Predicate { $0.id <= accepted })
} content: { query in
    ChildView(items: query.wrappedValue, theme: theme)
}
```

Rule: every value the closures capture from the parent must appear in `index` (directly or via an aggregate `Equatable` key). If a dependency cannot be expressed as `Equatable`, it doesn't belong in this subtree — push it down through `@Environment` instead, where SwiftUI's own change-tracking handles it.

## Why this is the right flow

- The parent can recompute often without thrashing the fetch pipeline.
- Query creation is tied to one explicit source-of-truth parameter.
- No fake mutability tricks (no rebuilding `_items` in `init`, no observed wrappers around `Query`).
- Invalidation is predictable and local — you can read the dependency by looking at `index`.

If params are SOT and `Query` is SOT, dependency-keyed recreation isn't a stylistic preference. It's the data-flow model that matches how SwiftData actually works.

## Generalising beyond `@Query`

The pattern is not SwiftData-specific. It applies to any `DynamicProperty` (or property factory) whose construction:

- Wires real machinery (subscriptions, fetch pipelines, caches, decoded resources).
- Depends on parent state that should drive recreation.
- Cannot mutate its inputs in place after construction.

In those cases, `IndexedSotView` (or a custom equivalent) is the boundary that decouples *construct* from *parent re-evaluates*. For trivial sources where construction is a no-op assignment — `@State`, `@Binding` projection, plain `@Environment` reads — none of this is needed.

## Checklist

- [ ] You actually need this — parent recomputes for unrelated reasons, OR Query depends on parent state, OR construction is expensive.
- [ ] `index` is `Equatable` and includes every parent value the closures capture.
- [ ] `content` receives the resolved data (`[Item]`) where possible, not the `Query` itself.
- [ ] Leaf views are pure-data and testable without SwiftData.
- [ ] The `Query` factory closure doesn't read state outside `index`.
