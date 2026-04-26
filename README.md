# SwiftUI MV Architecture

A Claude Code skill for writing SwiftUI the way it was actually designed — as a reactive dataflow graph of value types, not an OOP UI framework.

Based on the work of [Lazar Otasevic](https://github.com/sisoje) (lead iOS engineer, Emirates Group).

---

## What this is

MVVM and TCA are patterns imported from other platforms and ecosystems. They work against SwiftUI's native primitives, not with them. The result is broken `@Environment` injection inside ViewModels, `[weak self]` capture lists, extra re-renders, and test scaffolding that exists only to compensate for the architecture.

This skill encodes a different approach: **MV — Model as View**.

The premise is simple. `SwiftUI.View` is not a UI object — it's a protocol that describes a node in a reactive dependency graph. State flows down as values. Events flow up as closures. The framework handles the rest. When you work with that model instead of against it, a surprising number of hard problems disappear entirely.

---

## What you get

- `@State` with plain structs instead of `@Observable` classes — one rule, no lifecycle surprises
- Logic as stateless structs of closures (capabilities) — composable, trivially testable, no mocking framework needed
- `@Environment` for dependency injection — compile-time, hierarchical, works in previews for free
- Custom `DynamicProperty` for external state — stays inside SwiftUI's graph so every property wrapper keeps working
- Apple-style styleable components — encode visual variants as styles, not forks; one component, many looks
- Mechanical code-style rules and a hard `#Preview` requirement on every view — uniform output across humans and LLMs
- A narrow, explicit carve-out for when a class (or `@Observable`) is genuinely the right tool — and when it isn't
- No ViewModels. No reducers. No action enums. No `[weak self]`.

---

## Why it matters now

Architecture choices have a new dimension in 2026: **how well does AI work in this codebase?**

LLMs are good at pattern-completion over local context and bad at tracking long-range mutable state across non-local effects. OOP with class-based ViewModels is specifically the pattern that requires tracking long-range mutable state across non-local effects.

Purely functional SwiftUI has low entropy of valid programs — two developers solving the same problem converge to the same solution, the primitives mean the same thing at every scale, and there are very few ways to be wrong. That property makes AI assistance genuinely useful rather than requiring line-by-line supervision.

---

## What's in the skill

| File | Contents |
|---|---|
| `SKILL.md` | Mental model, "scan first" router, default patterns, view-init purity, file/styling rules, anti-patterns, class carve-out, triage |
| `references/state-ownership-decisions.md` | Decision tree for `@State` vs `@Binding` vs `@Environment` vs custom `DynamicProperty` |
| `references/custom-dynamic-properties.md` | Building custom sources of truth inside SwiftUI's graph |
| `references/dependency-injection.md` | `@Environment` + capability structs vs protocol/mock |
| `references/capability-pattern.md` | Closures as data — composition, ad-hoc polymorphism, when to use a protocol |
| `references/refactoring-from-mvvm.md` | Step-by-step migrations with before/after code |
| `references/why-not-tca.md` | TCA critique and full concept mapping to MV equivalents |
| `references/testing-views.md` | Testing SwiftUI views natively, without ViewModels |
| `references/file-organization.md` | One file per view, one folder per feature, `Common/` for shared — with worked example |
| `references/app-entry-point.md` | `@main App` defaults and `@UIApplicationDelegateAdaptor` cleanup |
| `references/code-style.md` | Six mechanical formatting rules — read before writing or reviewing any SwiftUI code |
| `references/style-protocols.md` | Reusable styles for built-in views (`ButtonStyle`, `ToggleStyle`, …) |
| `references/custom-component-styling.md` | Making your own components styleable like Apple's `Button`/`Toggle`/`GroupBox` |

See [CHANGELOG.md](CHANGELOG.md) for release history.

---

## Installation

Download the latest `swiftui-mv-architecture.skill` from [Releases](../../releases) and install it into Claude Code.

---

## Credits

Architecture and articles by [Lazar Otasevic](https://github.com/sisoje) — [Medium](https://medium.com/@redhotbits). Skill packaging by [Mirko Tomic](https://github.com/mirkokg).
