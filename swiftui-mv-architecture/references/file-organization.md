# File organization

SwiftUI projects should be organized by **visual and behavioral hierarchy**, not by code type. The folder structure is a map of the product, not the codebase.

## The single rule

> One file per view. One folder per feature. Shared code in `Common/`.

A new team member should be able to open Finder, navigate into `Home/`, and immediately understand what the Home tab contains — without reading any Swift.

## Root structure

The source root contains only the app entry point and the root scene — everything else is in a feature folder:

```
MyApp/
├── MyAppApp.swift       ← @main App struct
├── ContentView.swift    ← root TabView / NavigationStack
├── Home/
├── Search/
├── Profile/
└── Common/             ← reusable components, extensions, utilities
```

## Feature folders mirror the visual hierarchy

Each feature folder owns its screen struct, its subviews, and its local models:

```
Home/
├── HomeScreen.swift     ← screen entry point (the struct conforming to View)
├── FeaturedCard.swift   ← subview used only inside HomeScreen
├── ActivityRow.swift    ← subview used only inside HomeScreen
├── StatCard.swift       ← subview used only inside HomeScreen
├── ActivityItem.swift   ← data model used only by Home
└── StatItem.swift       ← data model used only by Home
```

If a screen section grows to contain its own set of subviews, give it a sub-folder:

```
Home/
├── HomeScreen.swift
├── FeaturedCard.swift
└── ActivityFeed/
    ├── ActivityFeedSection.swift
    └── ActivityRow.swift
```

## Common/

Anything used by more than one feature lives in `Common/`. This includes:
- Reusable view components (`SectionHeader`, `EmptyState`, `LoadingCard`)
- App-wide extensions (`Color+Theme.swift`, `Font+App.swift`)
- Shared utilities and helpers
- Custom-component infrastructure (`Common/RangeSlider/`, `Common/Styles/`)

```
Common/
├── SectionHeader.swift
└── Extensions/
    └── Color+Theme.swift
```

## Three-tab app: worked example

A dashboard app with Home, Search, and Profile tabs:

```
DashboardApp/
├── DashboardApp.swift       ← @main
├── ContentView.swift        ← TabView with three tabs
│
├── Home/
│   ├── HomeScreen.swift     ← struct HomeScreen (entry point)
│   ├── FeaturedCard.swift   ← struct FeaturedCard: View
│   ├── ActivityRow.swift    ← struct ActivityRow: View
│   ├── StatCard.swift       ← struct StatCard: View
│   ├── ActivityItem.swift   ← struct ActivityItem + enum ActivityStatus + .samples
│   └── StatItem.swift       ← struct StatItem + .samples
│
├── Search/
│   ├── SearchScreen.swift   ← struct SearchScreen (@State search + category)
│   ├── CategoryChips.swift  ← struct CategoryChips: View (reusable within Search)
│   ├── SearchItemRow.swift  ← struct SearchItemRow: View
│   └── SearchItem.swift     ← struct SearchItem + .samples
│
├── Profile/
│   ├── ProfileScreen.swift  ← struct ProfileScreen
│   ├── ProfileHeader.swift  ← struct ProfileHeader: View (avatar + stats)
│   ├── BadgeCard.swift      ← struct BadgeCard: View
│   ├── SettingsRowView.swift ← struct SettingsRowView: View
│   ├── BadgeItem.swift      ← struct BadgeItem (value type)
│   └── SettingsRow.swift    ← struct SettingsRow (value type)
│
└── Common/
    └── SectionHeader.swift  ← struct SectionHeader: View (title + optional action)
```

## Decision: where does a file go?

| Situation | Location |
|-----------|----------|
| View used only in Screen A | `FeatureA/` |
| View used in Screen A and B | `Common/` |
| Model used only by Screen A | `FeatureA/` |
| Model used by multiple screens | `Common/` |
| App-wide extension (`Color+`, `Font+`) | `Common/Extensions/` |
| Screen that grows into its own sub-hierarchy | Sub-folder inside the feature folder |

## Naming

- Screen entry points end in `Screen`: `HomeScreen`, `SearchScreen`
- Subview files are named after what they display: `ActivityRow`, `StatCard`, `SectionHeader`
- Model files are named after the type: `ActivityItem`, `SettingsRow`
- No `View` suffix unless disambiguation is necessary (`SettingsRowView` vs the model `SettingsRow`)

## Hard rules

- **One file per view.** Never group sibling views into the same file.
- **One folder per feature.** The folder name matches the tab or navigation destination.
- **Models live with their feature.** `ActivityItem.swift` belongs in `Home/`, not a global `Models/` folder, unless it is shared across features.
- **`private` structs that grow** → extract to their own file inside the same folder; drop `private`, rely on folder co-location for conceptual grouping.
- **Never create `Models/`, `Views/`, or `ViewModels/` at the top level.** Those reflect code types, not product structure.

## Access control

Files in the same folder are in the same Swift module — `internal` access (the default) is sufficient for co-located types to see each other. Only reach for `private` or `fileprivate` on helpers that are truly one-off utilities within a single file.

When a `private struct` inside a screen file starts growing, that is the signal to extract it to its own file in the same folder and drop `private`.

## Anti-patterns

❌ All subviews in one file:
```swift
// HomeScreen.swift — 300 lines, multiple private structs — DON'T DO THIS
struct HomeScreen: View { ... }
private struct FeaturedCard: View { ... }
private struct ActivityRow: View { ... }
private struct StatCard: View { ... }
```

❌ Type-based top-level folders:
```
MyApp/
├── Models/       ← reflects code type, not product structure
├── Views/        ← same problem
└── Utilities/    ← split utilities by feature instead
```

✅ Behavior/hierarchy-based folders:
```
MyApp/
├── Home/
├── Search/
├── Profile/
└── Common/
```

## What this is NOT

This is not Clean Architecture layers, not feature modules, not SPM packages. It is a **file naming convention** — simple, zero ceremony, immediately readable. Large apps may eventually graduate to separate Swift packages per feature, but that is an independent decision and this folder convention composes with it cleanly.
