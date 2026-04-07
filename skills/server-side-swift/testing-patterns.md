# Testing Patterns

VaporTesting, Hummingbird testing, Swift Testing integration, and integration test patterns for server-side Swift.

## Vapor Testing Setup

### Test Target Configuration

```swift
// Package.swift
.testTarget(
    name: "AppTests",
    dependencies: [
        .target(name: "App"),
        .product(name: "VaporTesting", package: "vapor"),
    ]
),
```

### Basic Test Structure (Swift Testing)

```swift
import Testing
import VaporTesting
@testable import App

@Suite("Todo API Tests", .serialized)
struct TodoTests {
    @Test("Health check returns 200")
    func healthCheck() async throws {
        try await withApp(configure: configure) { app in
            try await app.testing().test(.GET, "health") { res async throws in
                #expect(res.status == .ok)
            }
        }
    }

    @Test("Create todo returns created todo")
    func createTodo() async throws {
        try await withApp(configure: configure) { app in
            let input = CreateTodoRequest(title: "Test Todo")
            try await app.testing().test(.POST, "api/v1/todos",
                beforeRequest: { req in
                    try req.content.encode(input)
                },
                afterResponse: { res async throws in
                    #expect(res.status == .ok)
                    let todo = try res.content.decode(TodoDTO.self)
                    #expect(todo.title == "Test Todo")
                    #expect(todo.isComplete == false)
                    #expect(todo.id != nil)
                }
            )
        }
    }

    @Test("List todos returns array")
    func listTodos() async throws {
        try await withApp(configure: configure) { app in
            // Create test data
            try await app.testing().test(.POST, "api/v1/todos",
                beforeRequest: { req in
                    try req.content.encode(CreateTodoRequest(title: "Todo 1"))
                }
            ) { _ in }

            // List
            try await app.testing().test(.GET, "api/v1/todos") { res async throws in
                #expect(res.status == .ok)
                let todos = try res.content.decode([TodoDTO].self)
                #expect(!todos.isEmpty)
            }
        }
    }
}
```

### Test with Authentication

```swift
@Suite("Protected API Tests", .serialized)
struct ProtectedTodoTests {
    @Test("Unauthenticated request returns 401")
    func unauthenticatedRequest() async throws {
        try await withApp(configure: configure) { app in
            try await app.testing().test(.GET, "api/v1/todos") { res async throws in
                #expect(res.status == .unauthorized)
            }
        }
    }

    @Test("Authenticated request returns todos")
    func authenticatedRequest() async throws {
        try await withApp(configure: configure) { app in
            // Register user first
            let registerInput = RegisterRequest(
                name: "Test User",
                email: "test@example.com",
                password: "password123"
            )
            var token: String = ""

            try await app.testing().test(.POST, "api/v1/auth/register",
                beforeRequest: { req in
                    try req.content.encode(registerInput)
                },
                afterResponse: { res async throws in
                    #expect(res.status == .ok)
                    let tokenResponse = try res.content.decode(TokenResponse.self)
                    token = tokenResponse.accessToken
                }
            )

            // Use token for authenticated request
            try await app.testing().test(.GET, "api/v1/todos",
                beforeRequest: { req in
                    req.headers.bearerAuthorization = BearerAuthorization(token: token)
                },
                afterResponse: { res async throws in
                    #expect(res.status == .ok)
                }
            )
        }
    }
}
```

### Test Database Configuration

```swift
// TestApp+configure.swift
import Vapor
import Fluent
import FluentSQLiteDriver

func configureForTesting(_ app: Application) async throws {
    // Use in-memory SQLite for tests
    app.databases.use(.sqlite(.memory), as: .sqlite)

    // Run migrations
    app.migrations.add(CreateUser())
    app.migrations.add(CreateTodo())
    try await app.autoMigrate()

    // Register routes
    try routes(app)
}

// In tests:
@Test("Test with isolated database")
func testWithIsolatedDB() async throws {
    try await withApp(configure: configureForTesting) { app in
        // Each test gets a fresh in-memory database
        try await app.testing().test(.GET, "api/v1/todos") { res async throws in
            #expect(res.status == .ok)
            let todos = try res.content.decode([TodoDTO].self)
            #expect(todos.isEmpty)  // Fresh DB, no data
        }
    }
}
```

## Hummingbird Testing

### Basic Hummingbird Test

```swift
import Testing
import Hummingbird
import HummingbirdTesting
@testable import App

@Suite("Hummingbird API Tests", .serialized)
struct HBTodoTests {
    @Test("Health check")
    func healthCheck() async throws {
        let app = try await buildApplication(
            configuration: .init(address: .hostname("localhost", port: 0))
        )

        try await app.test(.router) { client in
            try await client.execute(uri: "/health", method: .get) { response in
                #expect(response.status == .ok)
            }
        }
    }

    @Test("Create todo")
    func createTodo() async throws {
        let app = try await buildApplication(
            configuration: .init(address: .hostname("localhost", port: 0))
        )

        try await app.test(.router) { client in
            let input = CreateTodoRequest(title: "Test")
            let body = try JSONEncoder.api.encode(input)

            try await client.execute(
                uri: "/api/v1/todos",
                method: .post,
                headers: [.contentType: "application/json"],
                body: ByteBuffer(data: body)
            ) { response in
                #expect(response.status == .created)
                let todo = try JSONDecoder.api.decode(
                    TodoDTO.self,
                    from: response.body
                )
                #expect(todo.title == "Test")
            }
        }
    }
}
```

## Test Helpers

### Factory Methods

```swift
// TestHelpers.swift
import Foundation
@testable import App

enum TestFactory {
    static func makeCreateTodoRequest(
        title: String = "Test Todo"
    ) -> CreateTodoRequest {
        CreateTodoRequest(title: title)
    }

    static func makeRegisterRequest(
        name: String = "Test User",
        email: String = "test\(UUID())@example.com",
        password: String = "password123"
    ) -> RegisterRequest {
        RegisterRequest(name: name, email: email, password: password)
    }
}
```

### Authenticated Test Helper

```swift
// TestHelpers+Auth.swift
import Vapor
import VaporTesting
@testable import App

extension Application {
    func loginTestUser(
        name: String = "Test User",
        email: String = "test@example.com",
        password: String = "password123"
    ) async throws -> String {
        var token = ""

        let registerInput = RegisterRequest(name: name, email: email, password: password)
        try await self.testing().test(.POST, "api/v1/auth/register",
            beforeRequest: { req in
                try req.content.encode(registerInput)
            },
            afterResponse: { res async throws in
                let response = try res.content.decode(TokenResponse.self)
                token = response.accessToken
            }
        )

        return token
    }
}

// Usage in tests:
@Test("Protected endpoint")
func protectedEndpoint() async throws {
    try await withApp(configure: configureForTesting) { app in
        let token = try await app.loginTestUser()

        try await app.testing().test(.GET, "api/v1/todos",
            beforeRequest: { req in
                req.headers.bearerAuthorization = BearerAuthorization(token: token)
            },
            afterResponse: { res async throws in
                #expect(res.status == .ok)
            }
        )
    }
}
```

### Response Assertion Helpers

```swift
extension HTTPHeaders {
    func expectContentType(_ expected: HTTPMediaType) {
        #expect(self.contentType == expected)
    }
}

func expectJSON<T: Decodable & Equatable>(
    _ response: Response,
    status: HTTPResponseStatus = .ok,
    body expectedBody: T
) throws {
    #expect(response.status == status)
    let decoded = try response.content.decode(T.self)
    #expect(decoded == expectedBody)
}
```

## Integration Test Patterns

### Full CRUD Cycle Test

```swift
@Test("Full todo CRUD lifecycle")
func crudLifecycle() async throws {
    try await withApp(configure: configureForTesting) { app in
        var todoID: UUID?

        // CREATE
        try await app.testing().test(.POST, "api/v1/todos",
            beforeRequest: { req in
                try req.content.encode(CreateTodoRequest(title: "New Todo"))
            },
            afterResponse: { res async throws in
                #expect(res.status == .ok)
                let todo = try res.content.decode(TodoDTO.self)
                todoID = todo.id
                #expect(todo.title == "New Todo")
                #expect(todo.isComplete == false)
            }
        )

        let id = try #require(todoID)

        // READ
        try await app.testing().test(.GET, "api/v1/todos/\(id)") { res async throws in
            #expect(res.status == .ok)
            let todo = try res.content.decode(TodoDTO.self)
            #expect(todo.title == "New Todo")
        }

        // UPDATE
        try await app.testing().test(.PUT, "api/v1/todos/\(id)",
            beforeRequest: { req in
                try req.content.encode(UpdateTodoRequest(title: "Updated", isComplete: true))
            },
            afterResponse: { res async throws in
                #expect(res.status == .ok)
                let todo = try res.content.decode(TodoDTO.self)
                #expect(todo.title == "Updated")
                #expect(todo.isComplete == true)
            }
        )

        // DELETE
        try await app.testing().test(.DELETE, "api/v1/todos/\(id)") { res async throws in
            #expect(res.status == .noContent)
        }

        // VERIFY DELETED
        try await app.testing().test(.GET, "api/v1/todos/\(id)") { res async throws in
            #expect(res.status == .notFound)
        }
    }
}
```

### Validation Test

```swift
@Test("Create todo rejects empty title")
func emptyTitleRejected() async throws {
    try await withApp(configure: configureForTesting) { app in
        try await app.testing().test(.POST, "api/v1/todos",
            beforeRequest: { req in
                try req.content.encode(["title": ""])
            },
            afterResponse: { res async throws in
                #expect(res.status == .badRequest)
            }
        )
    }
}

@Test("Create todo rejects missing title")
func missingTitleRejected() async throws {
    try await withApp(configure: configureForTesting) { app in
        try await app.testing().test(.POST, "api/v1/todos",
            beforeRequest: { req in
                try req.content.encode(["invalid": "data"])
            },
            afterResponse: { res async throws in
                #expect(res.status == .badRequest)
            }
        )
    }
}
```

### Error Response Test

```swift
@Test("Not found returns proper error body")
func notFoundError() async throws {
    try await withApp(configure: configureForTesting) { app in
        let fakeID = UUID()
        try await app.testing().test(.GET, "api/v1/todos/\(fakeID)") { res async throws in
            #expect(res.status == .notFound)
            let error = try res.content.decode(ErrorResponse.self)
            #expect(error.error == true)
            #expect(error.reason.contains("not found"))
        }
    }
}
```

## Performance Testing

```swift
@Test("List todos responds within 100ms")
func listPerformance() async throws {
    try await withApp(configure: configureForTesting) { app in
        // Seed data
        for i in 1...100 {
            try await app.testing().test(.POST, "api/v1/todos",
                beforeRequest: { req in
                    try req.content.encode(CreateTodoRequest(title: "Todo \(i)"))
                }
            ) { _ in }
        }

        // Measure
        let start = ContinuousClock.now
        try await app.testing().test(.GET, "api/v1/todos?per=20") { res async throws in
            #expect(res.status == .ok)
        }
        let elapsed = ContinuousClock.now - start

        #expect(elapsed < .milliseconds(100), "Response too slow: \(elapsed)")
    }
}
```

## Anti-Patterns

### ❌ Shared State Between Tests

```swift
// ❌ Bad — Tests share database state
static var app: Application!

@Test func test1() async throws {
    // Creates data that leaks to test2
}

@Test func test2() async throws {
    // Depends on data from test1
}

// ✅ Good — Each test gets fresh app/database
@Test func test1() async throws {
    try await withApp(configure: configureForTesting) { app in
        // Isolated database
    }
}
```

### ❌ Testing Implementation Instead of Behavior

```swift
// ❌ Bad — Testing internal database state
@Test func createTodo() async throws {
    let todo = Todo(title: "Test")
    try await todo.save(on: db)
    let count = try await Todo.query(on: db).count()
    #expect(count == 1)
}

// ✅ Good — Testing API behavior
@Test func createTodo() async throws {
    try await withApp(configure: configureForTesting) { app in
        try await app.testing().test(.POST, "api/v1/todos",
            beforeRequest: { req in
                try req.content.encode(CreateTodoRequest(title: "Test"))
            },
            afterResponse: { res async throws in
                #expect(res.status == .ok)
                let todo = try res.content.decode(TodoDTO.self)
                #expect(todo.title == "Test")
            }
        )
    }
}
```

### ❌ Hardcoded Test Data

```swift
// ❌ Bad — Hardcoded IDs can collide
let id = UUID(uuidString: "12345678-1234-1234-1234-123456789012")!

// ✅ Good — Generate unique data per test
let id = UUID()  // Random UUID
let email = "test\(UUID().uuidString.prefix(8))@example.com"  // Unique email
```

## Running Tests

```bash
# Run all tests
swift test

# Run specific test suite
swift test --filter TodoTests

# Run with verbose output
swift test --verbose

# Run tests in parallel (default in Swift Testing)
swift test --parallel

# Run with Docker (matching production environment)
docker-compose run --rm app swift test
```

## References

- [VaporTesting Documentation](https://docs.vapor.codes/basics/testing/)
- [Swift Testing Framework](https://developer.apple.com/documentation/testing)
- [HummingbirdTesting](https://docs.hummingbird.codes/2.0/documentation/hummingbirdtesting/)
- **vapor-patterns.md** — Controller patterns to test
- **auth-patterns.md** — Auth testing patterns
