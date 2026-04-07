# Hummingbird Patterns

Hummingbird 2 framework patterns, routing, contexts, middleware, and decision guide for choosing between Vapor and Hummingbird.

## Vapor vs Hummingbird Decision Tree

```
Start
├─ Need batteries-included (ORM, auth, WebSocket built-in)?
│  ├─ Yes → Vapor
│  └─ No → Continue
├─ Need pure structured concurrency (no EventLoopFuture)?
│  ├─ Yes → Hummingbird 2
│  └─ No → Continue
├─ Need lightweight/minimal footprint?
│  ├─ Yes → Hummingbird 2
│  └─ No → Continue
├─ Need large community/ecosystem?
│  ├─ Yes → Vapor
│  └─ No → Continue
├─ Building microservice or Lambda?
│  ├─ Yes → Hummingbird 2
│  └─ No → Vapor (safer default)
```

### Comparison Table

| Feature | Vapor 4 | Hummingbird 2 |
|---------|---------|----------------|
| **Concurrency** | async/await + EventLoopFuture legacy | Pure structured concurrency |
| **ORM** | Fluent (built-in) | Bring your own (PostgresNIO, SQLiteNIO) |
| **Auth** | Built-in middleware | hummingbird-auth package |
| **WebSocket** | Built-in | hummingbird-websocket package |
| **Binary Size** | ~50MB | ~15MB |
| **Startup Time** | ~200ms | ~50ms |
| **Swift Version** | 5.9+ | 5.9+ (designed for 6.0) |
| **Lambda Support** | Via adapter | First-class |
| **Learning Curve** | Lower (more docs) | Higher (fewer docs) |
| **Community** | Large (10k+ GitHub stars) | Growing (1k+ stars) |

## Package.swift

### Minimal Hummingbird Setup

```swift
// swift-tools-version:5.9
import PackageDescription

let package = Package(
    name: "MyApp",
    platforms: [.macOS(.v14)],
    dependencies: [
        .package(url: "https://github.com/hummingbird-project/hummingbird.git", from: "2.0.0"),
    ],
    targets: [
        .executableTarget(
            name: "App",
            dependencies: [
                .product(name: "Hummingbird", package: "hummingbird"),
            ]
        ),
        .testTarget(
            name: "AppTests",
            dependencies: [
                .target(name: "App"),
                .product(name: "HummingbirdTesting", package: "hummingbird"),
            ]
        ),
    ]
)
```

### With Database and Auth

```swift
dependencies: [
    .package(url: "https://github.com/hummingbird-project/hummingbird.git", from: "2.0.0"),
    .package(url: "https://github.com/hummingbird-project/hummingbird-auth.git", from: "2.0.0"),
    .package(url: "https://github.com/hummingbird-project/hummingbird-fluent.git", from: "2.0.0"),
    .package(url: "https://github.com/vapor/fluent-postgres-driver.git", from: "2.8.0"),
],
```

## Project Structure

```
Sources/App/
├── Controllers/
│   ├── TodoController.swift
│   └── UserController.swift
├── Models/
│   ├── Todo.swift
│   └── User.swift
├── Middleware/
│   └── LoggingMiddleware.swift
├── Application+build.swift      # App configuration
└── App.swift                    # Entry point
```

## Entry Point

### App.swift

```swift
import Hummingbird
import Logging

@main
struct App {
    static func main() async throws {
        let app = try await buildApplication(
            configuration: .init(
                address: .hostname("0.0.0.0", port: 8080),
                serverName: "MyApp"
            )
        )
        try await app.runService()
    }
}
```

## Application Configuration

### Application+build.swift

```swift
import Hummingbird
import Logging

func buildApplication(
    configuration: ApplicationConfiguration
) async throws -> some ApplicationProtocol {
    let logger = Logger(label: "MyApp")

    let router = Router()

    // Middleware
    router.middlewares.add(LogRequestsMiddleware(.info))
    router.middlewares.add(CORSMiddleware(
        allowOrigin: .originBased,
        allowHeaders: [.contentType, .authorization],
        allowMethods: [.get, .post, .put, .delete, .patch]
    ))

    // Health check
    router.get("health") { _, _ in
        HTTPResponse.Status.ok
    }

    // API routes
    let api = router.group("api/v1")
    TodoController.addRoutes(to: api.group("todos"))

    return Application(
        router: router,
        configuration: configuration,
        logger: logger
    )
}
```

## Routing

### Basic Routes

```swift
let router = Router()

// Simple response
router.get("hello") { _, _ -> String in
    "Hello, World!"
}

// Path parameters
router.get("users/:userID") { _, context -> String in
    let userID = try context.parameters.require("userID")
    return "User: \(userID)"
}

// JSON response
router.get("status") { _, _ -> ServerStatus in
    ServerStatus(version: "1.0.0", uptime: ProcessInfo.processInfo.systemUptime)
}

struct ServerStatus: ResponseCodable {
    let version: String
    let uptime: TimeInterval
}
```

### Route Groups

```swift
let api = router.group("api/v1")

// Public routes
let publicRoutes = api.group("auth")
publicRoutes.post("login", use: AuthController.login)
publicRoutes.post("register", use: AuthController.register)

// Protected routes
let protected = api.group()
    .add(middleware: AuthMiddleware())
protected.group("todos") { todos in
    todos.get(use: TodoController.index)
    todos.post(use: TodoController.create)
    todos.get(":id", use: TodoController.show)
    todos.put(":id", use: TodoController.update)
    todos.delete(":id", use: TodoController.delete)
}
```

## Controllers

### Controller Pattern

```swift
import Hummingbird

struct TodoController {
    typealias Context = BasicRequestContext

    static func addRoutes(to group: RouterGroup<Context>) {
        group.get(use: index)
        group.post(use: create)
        group.get(":id", use: show)
        group.put(":id", use: update)
        group.delete(":id", use: delete)
    }

    @Sendable
    static func index(_ request: Request, context: Context) async throws -> [TodoDTO] {
        // Fetch from repository
        try await TodoRepository.shared.findAll()
    }

    @Sendable
    static func create(_ request: Request, context: Context) async throws -> EditedResponse<TodoDTO> {
        let input = try await request.decode(as: CreateTodoRequest.self, context: context)
        let todo = try await TodoRepository.shared.create(title: input.title)
        return EditedResponse(status: .created, response: todo.toDTO)
    }

    @Sendable
    static func show(_ request: Request, context: Context) async throws -> TodoDTO {
        let id = try context.parameters.require("id", as: UUID.self)
        guard let todo = try await TodoRepository.shared.find(id: id) else {
            throw HTTPError(.notFound, message: "Todo not found")
        }
        return todo.toDTO
    }

    @Sendable
    static func update(_ request: Request, context: Context) async throws -> TodoDTO {
        let id = try context.parameters.require("id", as: UUID.self)
        let input = try await request.decode(as: UpdateTodoRequest.self, context: context)
        guard let todo = try await TodoRepository.shared.update(id: id, input: input) else {
            throw HTTPError(.notFound, message: "Todo not found")
        }
        return todo.toDTO
    }

    @Sendable
    static func delete(_ request: Request, context: Context) async throws -> HTTPResponse.Status {
        let id = try context.parameters.require("id", as: UUID.self)
        try await TodoRepository.shared.delete(id: id)
        return .noContent
    }
}
```

## Request Context

### Custom Request Context

```swift
import Hummingbird
import HummingbirdAuth

struct AppRequestContext: AuthRequestContext {
    var coreContext: CoreRequestContextStorage
    var auth: LoginCache

    init(source: Source) {
        self.coreContext = .init(source: source)
        self.auth = .init()
    }

    // Custom properties
    var currentUser: User? {
        auth.get(User.self)
    }
}
```

### Using Custom Context

```swift
func buildApplication(
    configuration: ApplicationConfiguration
) async throws -> some ApplicationProtocol {
    let router = Router(context: AppRequestContext.self)

    router.get("profile") { _, context -> UserDTO in
        guard let user = context.currentUser else {
            throw HTTPError(.unauthorized)
        }
        return user.toDTO
    }

    return Application(router: router, configuration: configuration)
}
```

## Middleware

### Custom Logging Middleware

```swift
import Hummingbird

struct TimingMiddleware<Context: RequestContext>: RouterMiddleware {
    func handle(
        _ request: Request,
        context: Context,
        next: (Request, Context) async throws -> Response
    ) async throws -> Response {
        let start = ContinuousClock.now
        let response = try await next(request, context)
        let elapsed = ContinuousClock.now - start
        context.logger.info(
            "\(request.method) \(request.uri.path) - \(response.status.code) (\(elapsed))"
        )
        return response
    }
}
```

### Authentication Middleware

```swift
import Hummingbird
import HummingbirdAuth

struct JWTAuthMiddleware: RouterMiddleware {
    typealias Context = AppRequestContext

    func handle(
        _ request: Request,
        context: Context,
        next: (Request, Context) async throws -> Response
    ) async throws -> Response {
        guard let bearer = request.headers.bearer else {
            throw HTTPError(.unauthorized, message: "Missing authorization header")
        }

        let payload = try await JWTService.verify(token: bearer.token)
        let user = try await UserRepository.find(id: payload.userID)

        var context = context
        context.auth.login(user)

        return try await next(request, context)
    }
}
```

## Error Handling

### HTTPError Usage

```swift
// ✅ Hummingbird uses HTTPError (not Abort like Vapor)
throw HTTPError(.notFound, message: "Resource not found")
throw HTTPError(.badRequest, message: "Invalid input")
throw HTTPError(.unauthorized, message: "Authentication required")
throw HTTPError(.forbidden, message: "Insufficient permissions")
```

### Custom Error Response

```swift
struct AppError: HTTPResponseError {
    var status: HTTPResponse.Status
    var message: String
    var code: String

    var headers: HTTPFields { [:] }

    func body(allocator: ByteBufferAllocator) -> ByteBuffer? {
        let response = ErrorResponse(error: true, code: code, message: message)
        return try? JSONEncoder().encodeAsByteBuffer(response, allocator: allocator)
    }
}

struct ErrorResponse: Codable {
    let error: Bool
    let code: String
    let message: String
}
```

## Fluent Integration (via hummingbird-fluent)

### Setup with Fluent

```swift
import Hummingbird
import HummingbirdFluent
import FluentPostgresDriver

func buildApplication(
    configuration: ApplicationConfiguration
) async throws -> some ApplicationProtocol {
    let fluent = Fluent(logger: Logger(label: "Fluent"))
    fluent.databases.use(
        .postgres(
            configuration: .init(
                hostname: Environment.get("DB_HOST") ?? "localhost",
                username: Environment.get("DB_USER") ?? "vapor",
                password: Environment.get("DB_PASS") ?? "vapor",
                database: Environment.get("DB_NAME") ?? "app_db",
                tls: .prefer(try .init(configuration: .clientDefault))
            )
        ),
        as: .psql
    )

    // Add migrations
    await fluent.migrations.add(CreateTodo())

    let router = Router()

    // Pass fluent to controllers via context or dependency injection
    TodoController.addRoutes(to: router.group("api/v1/todos"), fluent: fluent)

    var app = Application(router: router, configuration: configuration)
    app.addServices(fluent)
    return app
}
```

## AWS Lambda Deployment

### Lambda Handler

```swift
import AWSLambdaRuntime
import Hummingbird
import HummingbirdLambda

@main
struct MyLambda: APIGatewayV2LambdaFunction {
    typealias Context = BasicLambdaRequestContext<APIGatewayV2Request>

    init(context: LambdaInitializationContext) async throws {}

    func buildResponder() async throws -> some HTTPResponder<Context> {
        let router = Router(context: Context.self)

        router.get("hello") { _, _ -> String in
            "Hello from Lambda!"
        }

        return router.buildResponder()
    }
}
```

## Anti-Patterns

### ❌ Using Vapor Patterns in Hummingbird

```swift
// ❌ Bad — Vapor's Abort doesn't exist in Hummingbird
throw Abort(.notFound)

// ✅ Good — Use Hummingbird's HTTPError
throw HTTPError(.notFound, message: "Not found")
```

### ❌ Ignoring Structured Concurrency

```swift
// ❌ Bad — Creating detached tasks
Task.detached {
    await heavyComputation()
}

// ✅ Good — Use TaskGroup or structured approach
try await withThrowingTaskGroup(of: Void.self) { group in
    group.addTask { await heavyComputation() }
}
```

### ❌ Mutable Shared State Without Actor

```swift
// ❌ Bad — Shared mutable state
var requestCount = 0
router.get("count") { _, _ -> String in
    requestCount += 1  // Data race!
    return "\(requestCount)"
}

// ✅ Good — Use actor for shared state
actor RequestCounter {
    private var count = 0
    func increment() -> Int {
        count += 1
        return count
    }
}
```

## References

- [Hummingbird Documentation](https://docs.hummingbird.codes/)
- [Hummingbird GitHub](https://github.com/hummingbird-project/hummingbird)
- [Hummingbird Examples](https://github.com/hummingbird-project/hummingbird-examples)
- **vapor-patterns.md** — Vapor comparison
- **database-patterns.md** — Database integration
