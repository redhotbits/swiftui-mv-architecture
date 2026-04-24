# Testing SwiftUI Views Natively

A common MVVM justification is "views can't be tested, so we extract logic into a ViewModel class". In SwiftUI this is false twice over:

1. The pure-data **State** and stateless **Logic** structs are *easier* to test than any class — they are value types with no lifecycle, no mocking, and no setup.
2. The **View itself** can be tested natively by hosting it in a test `App` and observing body evaluations. No third-party library required.

This reference covers both.

## Level 1 — Test the logic, not the view

In the MV architecture, behavior lives on stateless `Logic` structs that consume `@Binding var state` plus injected closures. These test directly:

```swift
func test_submit_setsErrorOnFailure() async {
    var state = LoginState(email: "a@b.com", password: "wrong")
    let logic = LoginLogic(
        state: Binding(get: { state }, set: { state = $0 }),
        login: { _, _ in throw AuthError.invalidCredentials }
    )

    await logic.submit()

    XCTAssertEqual(state.error, "Invalid credentials")
    XCTAssertFalse(state.isSubmitting)
}
```

Everything needed:
- Construct `State` inline.
- Construct `Logic` inline with a `Binding` backed by a local var and stub closures.
- Invoke the capability.
- Assert on `state` afterward.

No mocking framework. No `XCTestCase` subclass tree. No `@MainActor` dance. No `TestStore`.

### Testing derived state

Derived state is a computed property on `State`. Test it as a pure function:

```swift
func test_canSubmit_requiresValidEmail() {
    var state = LoginState(email: "not-an-email", password: "12345678")
    XCTAssertFalse(state.canSubmit)

    state.email = "a@b.com"
    XCTAssertTrue(state.canSubmit)
}
```

### Testing `async` flows

`async throws` functions as capabilities are trivially testable because you control the stub:

```swift
func test_submit_happyPath() async throws {
    var state = LoginState(email: "a@b.com", password: "correct!")
    var loginCallCount = 0
    let logic = LoginLogic(
        state: Binding(get: { state }, set: { state = $0 }),
        login: { email, pwd in
            loginCallCount += 1
            XCTAssertEqual(email, "a@b.com")
            return User.sample
        }
    )

    await logic.submit()

    XCTAssertEqual(loginCallCount, 1)
    XCTAssertNil(state.error)
    XCTAssertFalse(state.isSubmitting)
}
```

## Level 2 — Test the view itself

You can test what the view's `body` actually produces by hosting it in a test-only `App` and observing body evaluations with `async`/`await`. The whole scaffolding is ~30 lines.

### The test infrastructure

```swift
import SwiftUI

// Test-only App that can host any view and share state with tests.
struct TestApp: App {
    static var shared: Self!
    @State var view: any View = EmptyView()

    var body: some Scene {
        let _ = Self.shared = self
        WindowGroup { AnyView(view) }
    }
}

// Tick mechanism: a continuation you can await on each body evaluation.
final class BodyTicker {
    private var continuation: AsyncStream<(Int, Any)>.Continuation?
    private(set) lazy var stream: AsyncStream<(Int, Any)> = AsyncStream { c in
        self.continuation = c
    }
    private var index = 0

    func tick<V>(_ view: V) {
        continuation?.yield((index, view))
        index += 1
    }
}

extension View {
    static func bodyEvaluations() -> AsyncStream<(Int, Self)> {
        // ...returns a stream parameterized by Self; see the repo for full code
        fatalError("Hook this up per the GitHub reference implementation")
    }
}
```

The full infrastructure is ~30 lines and available at: https://github.com/sisoje/testable-view-swiftui

### The "body assertion" hook

Inside the view you want to test, add a single line as the first line of `body`:

```swift
struct CounterScreen: View {
    @State var count = 0
    var body: some View {
        let _ = assert(bodyAssertion)   // fires every body evaluation in tests
        VStack {
            Text("Count: \(count)")
            Button("+") { count += 1 }
        }
    }
}
```

In release builds, `assert` is compiled out — zero runtime cost. In tests, it triggers a notification to the test harness that the body evaluated, along with the current view value.

### The test

```swift
func test_counter_increments() async throws {
    TestApp.shared.view = CounterScreen()
    for await (index, view) in CounterScreen.bodyEvaluations().prefix(2) {
        switch index {
        case 0:
            XCTAssertEqual(view.count, 0)
            view.increment()   // mutate through captured value
        case 1:
            XCTAssertEqual(view.count, 1)
        default: break
        }
    }
}
```

What this proves:
- The view renders with `count == 0` on first evaluation.
- After calling the mutating method, the view re-evaluates with `count == 1`.
- This is end-to-end: state change → graph invalidation → body re-run. Nothing mocked.

### Combining with ViewInspector (optional)

For asserting on the *output* of `body` (text content, button taps), ViewInspector slots in naturally:

```swift
switch index {
case 0:
    _ = try view.inspect().find(text: "Count: 0")
    try view.inspect().find(button: "+").tap()
case 1:
    _ = try view.inspect().find(text: "Count: 1")
default: break
}
```

## Why this is better than testing ViewModels

A typical MVVM test:

```swift
func testIncrement() {
    let vm = CounterViewModel()
    vm.increment()
    XCTAssertEqual(vm.count, 1)
}
```

This test proves the ViewModel's internal state mutated. It proves nothing about whether the view would render that mutation, whether `@Published` fires, whether the binding forwards correctly, or whether a property wrapper inside the ViewModel (e.g., `@AppStorage`) works — because those things *do not work* inside ViewModels.

In MV:
- Logic tests prove the state mutation is correct (same as VM test, minus the class).
- View tests prove the end-to-end render pipeline works (impossible in MVVM without UI tests).

Both without leaving unit tests.

## Common pitfalls

- **Don't use `@MainActor` reflexively on test types**. If your logic and state are pure value types, tests can run off the main actor. Only mark test helpers `@MainActor` when they touch `TestApp.shared` or anything SwiftUI-owned.
- **Avoid sharing state across tests**. `TestApp.shared` is a global convenience. Reset `TestApp.shared.view = EmptyView()` between tests (or use per-test `TestApp` instances if the framework allows).
- **Prefer the async-stream approach over `XCTestExpectation`**. Expectations are a UIKit-era pattern. `for await … in prefix(N)` reads better and is deterministic.
- **Do not test implementation details**. Do not assert "body was evaluated exactly 2 times". The number of evaluations can change with SwiftUI internals. Test *semantic* things — state values, rendered text, button availability.

## Summary

- Unit-test `State` and `Logic` structs as plain Swift values.
- Integration-test `View`s by hosting in `TestApp` and awaiting body evaluations.
- Neither approach needs `XCTestExpectation`, a mocking library, or a "TestStore"-style DSL.
- The infrastructure is small enough to paste into a project and own.
