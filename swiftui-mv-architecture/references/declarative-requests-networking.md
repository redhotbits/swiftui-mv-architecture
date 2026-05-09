# Networking with DeclarativeRequests

When the codebase depends on [`declarative-requests-swift`](https://github.com/sisoje/declarative-requests-swift), it slots into the capability pattern as one specific layer: **request construction**. Execution, decoding, and domain capabilities stay as ordinary capability structs around it.

This file is about the **layering** in MV terms. For the DSL itself (block reference, `@RequestBuilder` rules, edge cases), see the `declarative-requests-swift` skill.

## The four layers

```
View
  │ depends on
  ▼
Domain capability        struct UserService { var fetch: (UserID) async throws -> User }
  │ composes
  ├── Repository         struct UserRepository { @RequestBuilder var getUser: (UserID) -> any RequestBuildable; ... }
  ├── Network closure    (URLRequest) async throws -> (Data, URLResponse)
  └── Decoder            JSONDecoder
```

Each layer is a value. Each is independently testable. Each is replaceable at the composition root.

## Repository — a struct of `@RequestBuilder` closures

This is the shape from the library's own `repositoryExample` test, lifted into MV:

```swift
struct UserRepository {
    @RequestBuilder var getUser:    (UserID) -> any RequestBuildable
    @RequestBuilder var refresh:    (String) -> any RequestBuildable
    @RequestBuilder var deleteUser: (UserID) -> any RequestBuildable
}

extension UserRepository {
    static func live(bearerToken: @escaping () -> String?) -> UserRepository {
        UserRepository(
            getUser: { id in
                Method.GET
                BaseURL("https://api.example.com")
                Endpoint("/users/\(id)")

                if let token = bearerToken() {
                    Authorization(bearer: token)
                }
            },
            refresh: { token in
                Method.POST
                BaseURL("https://api.example.com")
                Endpoint("/auth/refresh")
                JSONBody(["token": token])
            },
            deleteUser: { id in
                Method.DELETE
                BaseURL("https://api.example.com")
                Endpoint("/users/\(id)")

                if let token = bearerToken() {
                    Authorization(bearer: token)
                }
            }
        )
    }
}
```

It is a capability struct — same MV shape as `UserService` — except the closures return `RequestBuildable` values, not network results. No `async`, no execution. The throw happens later, when a caller reads `.request`.

## Domain capability composes Repository + network + decoder

```swift
typealias NetworkClosure = (URLRequest) async throws -> (Data, URLResponse)

struct UserService {
    var fetch:   (UserID) async throws -> User
    var delete:  (UserID) async throws -> Void
    var refresh: (String) async throws -> Void
}

extension UserService {
    static func live(
        repository: UserRepository,
        network: @escaping NetworkClosure,
        decoder: JSONDecoder = .init()
    ) -> UserService {
        UserService(
            fetch: { id in
                let request = try repository.getUser(id).request
                let (data, _) = try await network(request)
                return try decoder.decode(User.self, from: data)
            },
            delete: { id in
                let request = try repository.deleteUser(id).request
                _ = try await network(request)
            },
            refresh: { token in
                let request = try repository.refresh(token).request
                _ = try await network(request)
            }
        )
    }
}
```

`UserService` is what views see (injected via `@Environment`). `UserRepository`, the `NetworkClosure`, and the decoder are implementation details bolted together at the composition root.

## Why split the Repository from the network capability

Two distinct concerns:

1. **Request *shape*** — URL, headers, body, auth. Deterministic. Authoritative.
2. **Request *execution*** — network call, response decoding, error mapping. Side-effecting. Environment-dependent.

Collapsing them into a single `(UserID) async throws -> User` closure means you can only test by mocking the whole pipeline. Splitting buys three independent test surfaces:

- **Repository**: assert directly on the `URLRequest` — no `URLSession`, no async.
- **Network closure**: stub it with a closure that returns canned `Data`.
- **Domain capability**: compose the real repository with a stub network closure to verify decoding and error paths end-to-end without touching the wire.

## Testing each layer

**Repository — pure URL/header assertions:**

```swift
@Test func userRepository_getUser_buildsCorrectRequest() throws {
    let repository = UserRepository.live(bearerToken: { "abc" })
    let request = try repository.getUser(UserID("42")).request

    #expect(request.httpMethod == "GET")
    #expect(request.url?.absoluteString == "https://api.example.com/users/42")
    #expect(request.value(forHTTPHeaderField: "Authorization") == "Bearer abc")
}
```

**Domain capability — stub network only:**

```swift
@Test func userService_fetch_decodesUser() async throws {
    let stubResponse = try JSONEncoder().encode(User.fixture)
    let service = UserService.live(
        repository: .live(bearerToken: { "abc" }),
        network: { _ in (stubResponse, .okResponse) }
    )

    let user = try await service.fetch(UserID("42"))

    #expect(user == .fixture)
}
```

The real repository runs (so the test implicitly verifies request construction). Only the network is stubbed.

**View — stub the domain capability:**

The view never knows the repository or network closure exist. Tests construct `UserService(fetch: { _ in .fixture }, ...)` inline.

## Network closure — also a capability

Treat the executor as a capability struct (or a typealias for the closure):

```swift
typealias NetworkClosure = (URLRequest) async throws -> (Data, URLResponse)

extension NetworkClosure where Self == NetworkClosure {
    static var live: NetworkClosure {
        { try await URLSession.shared.data(for: $0) }
    }
}
```

Cross-cutting concerns (auth refresh, retries, logging, caching) compose by closure wrapping — see `capability-pattern.md` for `AuthComposer` and `retrying(times:)`. The DSL is orthogonal: it only produces `URLRequest` values, which any of these wrappers consume unchanged.

## Where each layer lives

```
Common/
└── Networking/
    ├── NetworkClosure.swift               ← typealias + .live / .preview / .unimplemented
    ├── ComposableNetworking.swift         ← auth/retry/logging composers
    └── Repositories/
        ├── UserRepository.swift           ← struct + .live(...)
        ├── OrderRepository.swift
        └── ...

Common/Services/
├── UserService.swift                      ← struct + .live(...) + .preview + .unimplemented + EnvironmentKey
└── ...
```

Hard rule: views inject the **domain capability** via `@Environment`, never the repository or the network closure. The latter two are wired up once at the composition root.

## When NOT to split

For one-off requests with no reuse — a health check, a debug-screen fetch — the inline form is fine:

```swift
let request = try RequestBlock {
    Method.GET
    BaseURL("https://api.example.com")
    Endpoint("/health")
}.request

let (data, _) = try await network(request)
```

The Repository struct earns its keep when (a) a base URL or auth header is shared across endpoints, (b) the same endpoint is called from multiple capabilities or screens, or (c) you want to assert request shape in isolation. None of those apply for a one-shot health check.

## Available blocks (quick orientation)

For full DSL details consult the `declarative-requests-swift` skill. The complete block list:

| Block | Use |
|---|---|
| `Method.GET` / `.POST` / `.PUT` / `.DELETE` / `.PATCH` / `.HEAD` / `.OPTIONS` / `.CONNECT` / `.TRACE` / `.custom("…")` | HTTP method |
| `BaseURL(URL)` / `BaseURL(String)` | Base URL — required before `Endpoint` resolves |
| `Endpoint("/path")` | Path appended to BaseURL |
| `Query("name", "value")` / `Query(encodable)` | URL query items (encodable flattens by key) |
| `JSONBody(encodable)` | JSON body — also sets `Content-Type: application/json` |
| `URLEncodedBody("k", "v")` / `URLEncodedBody(encodable)` / `URLEncodedBody([:])` | Form-encoded body; repeated keys allowed |
| `StreamBody(InputStream)` | Body from a stream |
| `Header.contentType.setValue("…")` / `.addValue("…")` / `Header.custom("X-Trace-Id").setValue("…")` | Headers (typed common cases + custom) |
| `Authorization(bearer: token)` / `Authorization(username:password:)` | `Authorization` header (Bearer or Basic) |
| `Cookie("key", "value")` | Accumulates into the `Cookie` header |
| `ContentType.JSON` / `.URLEncoded` / image/audio/video/font MIME types | Standalone Content-Type setter |
| `Timeout(30)` | `request.timeoutInterval` |
| `AllowAccess.cellular(true)` / `.expensiveNetwork(false)` / `.constrainedNetwork(false)` / `.ultraConstrainedNetwork(false)` | Per-request access toggles |
| `RequestBlock { … }` | Group blocks (useful inside conditional branches that produce more than one block) |

Result-builder control flow works inside any builder closure: `if` / `else`, `for`-`in`, etc.

## Checklist

- [ ] Repository is a `struct` of `@RequestBuilder` closures returning `any RequestBuildable`.
- [ ] Repository closures contain only block expressions — no side effects, no execution.
- [ ] Domain capabilities (`UserService` etc.) compose Repository + `NetworkClosure` + decoder, exposing `async throws -> Domain` closures.
- [ ] Views inject only the domain capability via `@Environment`. Repository and network closure live at the composition root.
- [ ] Tests: assert `URLRequest` shape directly on the repository; stub `NetworkClosure` for end-to-end behavior; stub the domain capability for views.
