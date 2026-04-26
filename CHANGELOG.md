# Changelog

All notable changes to the `swiftui-mv-architecture` skill are documented in this file.

This project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html) and the format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [1.1.0] - 2026-04-26

### Added

- **"Scan first" router** at the top of `SKILL.md` so the AI can route to the relevant section/reference instead of reading the file end-to-end on every load.
- **"Reusable components"** section: reuse-first rule, decision table (data / structure / visuals → component vs parameter vs style), explicit "when NOT to fork" list.
- **"Reusable styles"** section: extract rule, `Common/Styles/` folder layout, full list of built-in styleable view protocols (`ButtonStyle`, `ToggleStyle`, …).
- **"Code style"** section: six mechanical formatting rules — explicit type names (no `.init`), modifier-per-line, empty line between sibling views, no inline block content, labelled-args line breaking, shape aliases at modifier call sites.
- **"Previews"** section: every `View` must have a working `#Preview` block (no legacy `PreviewProvider`); one preview per visual variant, named after the variant.
- **"When classes ARE allowed"** section: narrow carve-out for UIKit/AppKit bridging and unavoidable reference-type APIs. `@Observable` carries an additional constraint — only when the class plays a model role *and* needs a struct-impossible lifecycle hook (typically `deinit` cleanup).
- **"View init: data plumbing only"** section: the explicit `init` of a `View` struct must only copy arguments to stored properties. No side effects, no logic, no eager initialisation, no resource opening. SwiftUI re-creates view structs freely.
- **"Never create empty files"** rule added to the file organization hard rules.
- `references/code-style.md` — full formatting rules with worked examples.
- `references/app-entry-point.md` — `@main App` defaults, `@UIApplicationDelegateAdaptor` snippet, and Xcode-template cleanup checklist.
- `references/style-protocols.md` — implementation template for built-in style protocols, parameterised-style pattern, environment-access rules.
- `references/custom-component-styling.md` — five-piece pattern for making custom components styleable like Apple's (`Button`/`Toggle`/`GroupBox`): configuration, protocol, environment key, modifier, shorthand. Covers the `DynamicProperty` + `resolve(_:)` trick that lets styles read `@Environment`, configuration-initialiser composition, cross-component `ModifiedStyle` reuse, and modal-propagation caveats.

### Changed

- `SKILL.md` trimmed from 780 to ~470 lines by moving formatting/patterns details to references and keeping summaries + pointers inline. Trigger cost roughly halved on every SwiftUI-related conversation.
- `references/file-organization.md` consolidated to absorb the content previously duplicated in `SKILL.md`. The skill now states the rule and points; the reference owns the details.
- Pattern 3 ("Model conforms to `View`") now requires the struct to carry its own data (≥1 stored property, `@State`, `@Binding`, or `@Environment`). Empty `struct Foo {}` + `extension Foo: View` is explicitly forbidden — pure ceremony, strictly worse than a single inline declaration.
- Rule "one parameter per line for multi-argument calls" refined to fire only when **≥2 arguments are labelled**. Positional/unlabelled calls (`min(a, b)`, `max(x, y)`, `zip(a, b)`, `print(x, y)`) now correctly stay on one line.
- Skill `description` trigger keywords expanded to cover component reuse, custom styles, and styleable components.
- Reference-files index regrouped by intent: Architecture & state / Migration / Testing / Project hygiene / Components & styling. `code-style.md` flagged as **read before writing or reviewing any SwiftUI code**.

## [1.0.0] - 2026-04-24

### Added

- Initial release of the `swiftui-mv-architecture` skill.
- Core mental model: SwiftUI is a reactive dataflow system of dependency nodes; `View` is a protocol any value can conform to; the view struct has no lifecycle.
- The rule: decouple state from logic — keep state as pure data owned by the framework, logic as a stateless value exposing closures.
- Default patterns 1–6: inline `@State`, state/logic split, model-as-view, custom `DynamicProperty`, `@Environment` injection, capability pattern.
- Anti-patterns: ViewModels with `@Observable`/`ObservableObject`, `@StateObject`/`@ObservedObject` in new code, TCA-style enum-action reducers, "where do I put the business logic?", protocol + mock for every dependency.
- Triage and communication style guidance.
- Reference files: `custom-dynamic-properties.md`, `dependency-injection.md`, `capability-pattern.md`, `refactoring-from-mvvm.md`, `why-not-tca.md`, `testing-views.md`, `state-ownership-decisions.md`, `file-organization.md`.
- GitHub Actions release workflow that zips the skill on every `v*` tag push.

[1.1.0]: https://github.com/redhotbits/swiftui-mv-architecture/releases/tag/v1.1.0
[1.0.0]: https://github.com/redhotbits/swiftui-mv-architecture/releases/tag/v1.0.0
