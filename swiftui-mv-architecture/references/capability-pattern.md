# The Capability Pattern

**Capability = a closure (or a struct of closures) that represents behavior as data.**

Closures are first-class values in Swift. They can be passed, stored, composed, and substituted. Unlike classes they carry no hidden state; unlike enums they execute directly without an interpreter; unlike protocols they need no conformance ceremony.

The capability pattern is the natural way to represent behavior in a value-oriented, SwiftUI-friendly codebase. It is what makes logic *composable*.

## The two shapes

### Single closure (one capability)

```swift
typealias FetchUser = (UserID) async throws -> User

struct UserDetailView: View {
    let userID: UserID
    let fetchUser: FetchUser

    @State private var user: User?

    var body: some View {
        Group { if let u = user { UserBody(user: u) } else { ProgressView() } }
            .task { user = try? await fetchUser(userID) }
    }
}
```

### Struct of closures (grouped capabilities)

When several related capabilities travel together:

```swift
struct UserService {
    var fetch:  (UserID) async throws -> User
    var save:   (User)   async throws -> Void
    var delete: (UserID) async throws -> Void
}
```

This is often called a "client" in other codebases. Same idea, no ceremony.

## Why this beats protocol + class

| Concern                | Protocol + class               | Capability struct          |
| ---------------------- | ------------------------------ | -------------------------- |
| Substitution           | Need a conforming type         | Construct with new closures |
| Mocking in tests       | Subclass or mock framework     | Pass stub closures          |
| Composition            | Wrapper types / decorator      | Wrap a closure in a closure |
| Partial override       | Subclass + override one method | Construct with some reused, some replaced |
| Existential cost       | Boxing overhead                | Value semantics            |
| Lines of code          | N                              | ~N/3                       |
| Swift idiomaticity     | Java-in-Swift                  | Idiomatic Swift            |

## Composition

Capabilities compose by closure wrapping — a higher-order capability takes one capability and returns another, usually adding a cross-cutting concern.

### Example: adding logging

```swift
extension UserService {
    func logged(_ log: @escaping (String) -> Void) -> UserService {
        UserService(
            fetch: { id in
                log("fetch(\(id))")
                return try await self.fetch(id)
            },
            save: { user in
                log("save(\(user.id))")
                try await self.save(user)
            },
            delete: { id in
                log("delete(\(id))")
                try await self.delete(id)
            }
        )
    }
}

// At the composition root
let service = UserService.live.logged { print("[UserService]", $0) }
```

### Example: adding retry

```swift
extension UserService {
    func retrying(times: Int) -> UserService {
        UserService(
            fetch: { id in
                var lastError: Error?
                for _ in 0..<times {
                    do { return try await self.fetch(id) } catch { lastError = error }
                }
                throw lastError!
            },
            save: self.save,
            delete: self.delete
        )
    }
}
```

Note: only `fetch` is wrapped; `save` and `delete` are passed through unchanged. This is *partial override* without subclassing.

### Example: composing capabilities from smaller ones

`ComposableNetworking` from Lazar's networking article is this pattern at app-lifecycle scale:

```swift
typealias NetworkClosureType = (URLRequest) async throws -> (Data, URLResponse)

struct AuthComposer {
    @Binding var bearerToken: String?
    let baseFunction: NetworkClosureType
    let authorizeRequest: (URLRequest, String) -> URLRequest

    var composition: NetworkClosureType {
        { request in
            guard let token = bearerToken else {
                throw ComposableNetworkingError.noBearerToken
            }
            return try await baseFunction(authorizeRequest(request, token))
        }
    }
}

struct ComposableNetworking {
    @Binding var bearerToken: String?
    let authComposer: AuthComposer
    let baseFunction: NetworkClosureType

    var composition: NetworkClosureType {
        bearerToken != nil ? authComposer.composition : baseFunction
    }
}
```

Everything — including the *networking layer itself* — is a value (`struct` holding closures). State (`bearerToken`) is injected as a `@Binding`. No class, no framework. The final `composition` is just a function `(URLRequest) async throws -> (Data, URLResponse)` — the same shape as `URLSession.shared.data(for:)` — that any layer above can consume without knowing about auth.

## Canned instances

For each capability, provide canned instances:

```swift
extension UserService {
    static let live: UserService = UserService(
        fetch:  { id in /* real network */ },
        save:   { user in /* real network */ },
        delete: { id in /* real network */ }
    )

    static let preview: UserService = UserService(
        fetch:  { _ in .preview },
        save:   { _ in },
        delete: { _ in }
    )

    static let unimplemented: UserService = UserService(
        fetch:  { _ in fatalError("unimplemented") },
        save:   { _ in fatalError("unimplemented") },
        delete: { _ in fatalError("unimplemented") }
    )
}
```

`unimplemented` is useful in tests: the test passes in stubs *only* for the capabilities it actually exercises, and any unexpected call is a loud test failure rather than silent success.

## When to use a protocol instead

Capability structs are the default. Reach for a protocol only when:

1. **You need conformance to be discoverable by the compiler across a module boundary** — e.g., a plugin system where external callers provide types.
2. **Objective-C interop** requires a class type.
3. **The capability genuinely has many methods (10+) with shared default implementations.** A struct of closures grows awkward, and protocol extensions give cheap defaults.

For the 95% case — a handful of methods, one or two live implementations plus test/preview variants — the capability struct wins.

## Testability

Because capabilities are plain values, tests construct them inline:

```swift
func test_checkout_tracksEvent() async {
    var tracked: [String] = []
    let analytics = Analytics(track: { event, _ in tracked.append(event) })
    let sut = CheckoutLogic(analytics: analytics)

    await sut.buy()

    XCTAssertEqual(tracked, ["buy_tapped"])
}
```

No mock framework. No `expect(...).to(receive(...))`. Just a closure that records into a local variable. The test reads like the specification.

## Ad-hoc polymorphism

With closures you can substitute behavior for a *single call site* without defining a new type:

```swift
let aggressiveRetry = UserService.live.retrying(times: 5)
let normalRetry    = UserService.live.retrying(times: 2)

ProductListView(service: normalRetry)     // normal
AdminDashboardView(service: aggressiveRetry) // critical
```

Two different behaviors, one underlying service, zero new types. This is very hard to do cleanly with protocols.
