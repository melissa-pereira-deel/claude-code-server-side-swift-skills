# Vapor Patterns

Routing, controllers, middleware, configuration, and project structure for Vapor 4+.

## Project Structure

### Standard Vapor Layout

```
Sources/App/
├── Controllers/           # Route collections
│   ├── TodoController.swift
│   └── UserController.swift
├── DTOs/                  # Request/response objects
│   ├── TodoDTO.swift
│   └── UserDTO.swift
├── Migrations/            # Database migrations
│   ├── CreateTodo.swift
│   └── CreateUser.swift
├── Models/                # Fluent models
│   ├── Todo.swift
│   └── User.swift
├── Middleware/             # Custom middleware
│   └── CORSConfiguration.swift
├── configure.swift        # App configuration
├── entrypoint.swift       # Entry point
└── routes.swift           # Route registration
```

## Package.swift

### Minimal Vapor Setup

```swift
// swift-tools-version:5.9
import PackageDescription

let package = Package(
    name: "MyApp",
    platforms: [.macOS(.v13)],
    dependencies: [
        .package(url: "https://github.com/vapor/vapor.git", from: "4.89.0"),
        .package(url: "https://github.com/vapor/fluent.git", from: "4.9.0"),
        .package(url: "https://github.com/vapor/fluent-postgres-driver.git", from: "2.8.0"),
    ],
    targets: [
        .executableTarget(
            name: "App",
            dependencies: [
                .product(name: "Vapor", package: "vapor"),
                .product(name: "Fluent", package: "fluent"),
                .product(name: "FluentPostgresDriver", package: "fluent-postgres-driver"),
            ]
        ),
        .testTarget(
            name: "AppTests",
            dependencies: [
                .target(name: "App"),
                .product(name: "VaporTesting", package: "vapor"),
            ]
        ),
    ]
)
```

### With Authentication and JWT

```swift
dependencies: [
    .package(url: "https://github.com/vapor/vapor.git", from: "4.89.0"),
    .package(url: "https://github.com/vapor/fluent.git", from: "4.9.0"),
    .package(url: "https://github.com/vapor/fluent-postgres-driver.git", from: "2.8.0"),
    .package(url: "https://github.com/vapor/jwt.git", from: "4.2.0"),
    .package(url: "https://github.com/vapor/redis.git", from: "4.10.0"),
],
```

## Entry Point

### Modern Entry Point (Vapor 4.89+)

```swift
import Vapor
import Logging

@main
enum Entrypoint {
    static func main() async throws {
        var env = try Environment.detect()
        try LoggingSystem.bootstrap(from: &env)

        let app = try await Application.make(env)
        defer { Task { try await app.asyncShutdown() } }

        try await configure(app)
        try await app.execute()
    }
}
```

## Configuration

### configure.swift

```swift
import Vapor
import Fluent
import FluentPostgresDriver

func configure(_ app: Application) async throws {
    // MARK: - Server Configuration
    app.http.server.configuration.hostname = "0.0.0.0"
    app.http.server.configuration.port = Environment.get("PORT").flatMap(Int.init) ?? 8080

    // MARK: - Database
    app.databases.use(
        .postgres(
            configuration: .init(
                hostname: Environment.get("DB_HOST") ?? "localhost",
                port: Environment.get("DB_PORT").flatMap(Int.init) ?? 5432,
                username: Environment.get("DB_USER") ?? "vapor",
                password: Environment.get("DB_PASS") ?? "vapor",
                database: Environment.get("DB_NAME") ?? "vapor_db",
                tls: .prefer(try .init(configuration: .clientDefault))
            )
        ),
        as: .psql
    )

    // MARK: - Migrations
    app.migrations.add(CreateUser())
    app.migrations.add(CreateTodo())

    // MARK: - Middleware
    app.middleware.use(FileMiddleware(publicDirectory: app.directory.publicDirectory))
    app.middleware.use(CORSMiddleware.default())

    // MARK: - Routes
    try routes(app)
}
```

### CORS Middleware Configuration

```swift
extension CORSMiddleware {
    static func `default`() -> CORSMiddleware {
        CORSMiddleware(configuration: .init(
            allowedOrigin: .any(
                Environment.get("ALLOWED_ORIGINS")?
                    .split(separator: ",")
                    .map(String.init) ?? ["http://localhost:3000"]
            ),
            allowedMethods: [.GET, .POST, .PUT, .DELETE, .PATCH, .OPTIONS],
            allowedHeaders: [
                .accept, .authorization, .contentType, .origin,
                .xRequestedWith, .init("X-API-Key")
            ],
            allowCredentials: true
        ))
    }
}
```

## Routing

### Basic Route Registration

```swift
// routes.swift
func routes(_ app: Application) throws {
    // Health check
    app.get("health") { req async -> HTTPStatus in
        .ok
    }

    // API versioning
    let api = app.grouped("api", "v1")

    // Register controllers
    try api.register(collection: TodoController())
    try api.register(collection: UserController())
}
```

### Route Groups and Middleware

```swift
func routes(_ app: Application) throws {
    let api = app.grouped("api", "v1")

    // Public routes
    let publicRoutes = api.grouped("public")
    try publicRoutes.register(collection: AuthController())

    // Protected routes (require authentication)
    let protected = api.grouped(UserToken.authenticator(), UserToken.guardMiddleware())
    try protected.register(collection: TodoController())
    try protected.register(collection: ProfileController())

    // Admin routes
    let admin = protected.grouped(AdminMiddleware())
    try admin.register(collection: AdminController())
}
```

## Controllers (RouteCollection)

### Standard Controller Pattern

```swift
import Vapor
import Fluent

struct TodoController: RouteCollection {
    func boot(routes: RoutesBuilder) throws {
        let todos = routes.grouped("todos")

        todos.get(use: index)
        todos.post(use: create)
        todos.group(":todoID") { todo in
            todo.get(use: show)
            todo.put(use: update)
            todo.delete(use: delete)
        }
    }

    // MARK: - CRUD Operations

    @Sendable
    func index(req: Request) async throws -> [TodoDTO] {
        try await Todo.query(on: req.db)
            .sort(\.$createdAt, .descending)
            .all()
            .map(\.toDTO)
    }

    @Sendable
    func create(req: Request) async throws -> TodoDTO {
        let input = try req.content.decode(CreateTodoRequest.self)
        try CreateTodoRequest.validate(content: req)

        let todo = Todo(title: input.title, isComplete: false)
        try await todo.save(on: req.db)
        return todo.toDTO
    }

    @Sendable
    func show(req: Request) async throws -> TodoDTO {
        guard let todo = try await Todo.find(req.parameters.get("todoID"), on: req.db) else {
            throw Abort(.notFound, reason: "Todo not found")
        }
        return todo.toDTO
    }

    @Sendable
    func update(req: Request) async throws -> TodoDTO {
        guard let todo = try await Todo.find(req.parameters.get("todoID"), on: req.db) else {
            throw Abort(.notFound, reason: "Todo not found")
        }

        let input = try req.content.decode(UpdateTodoRequest.self)
        todo.title = input.title ?? todo.title
        todo.isComplete = input.isComplete ?? todo.isComplete
        try await todo.save(on: req.db)
        return todo.toDTO
    }

    @Sendable
    func delete(req: Request) async throws -> HTTPStatus {
        guard let todo = try await Todo.find(req.parameters.get("todoID"), on: req.db) else {
            throw Abort(.notFound, reason: "Todo not found")
        }
        try await todo.delete(on: req.db)
        return .noContent
    }
}
```

### Paginated Controller

```swift
@Sendable
func index(req: Request) async throws -> Page<TodoDTO> {
    try await Todo.query(on: req.db)
        .sort(\.$createdAt, .descending)
        .paginate(for: req)
        .map(\.toDTO)
}
// GET /todos?page=2&per=20
```

## DTOs and Content

### Request/Response DTOs

```swift
// ✅ Good — Separate DTOs from database models
struct TodoDTO: Content, Sendable {
    let id: UUID?
    let title: String
    let isComplete: Bool
    let createdAt: Date?
}

struct CreateTodoRequest: Content, Validatable {
    let title: String

    static func validations(_ validations: inout Validations) {
        validations.add("title", as: String.self, is: !.empty && .count(1...255))
    }
}

struct UpdateTodoRequest: Content {
    let title: String?
    let isComplete: Bool?
}
```

```swift
// ❌ Bad — Exposing database model directly as response
app.get("todos") { req async throws -> [Todo] in
    try await Todo.query(on: req.db).all()  // Leaks internal fields
}
```

### Model to DTO Conversion

```swift
extension Todo {
    var toDTO: TodoDTO {
        TodoDTO(
            id: id,
            title: title,
            isComplete: isComplete,
            createdAt: createdAt
        )
    }
}
```

## Middleware

### Custom Timing Middleware

```swift
struct RequestTimingMiddleware: AsyncMiddleware {
    func respond(to request: Request, chainingTo next: AsyncResponder) async throws -> Response {
        let start = Date()
        let response = try await next.respond(to: request)
        let elapsed = Date().timeIntervalSince(start)
        response.headers.add(name: "X-Response-Time", value: "\(elapsed)ms")
        request.logger.info("[\(request.method)] \(request.url.path) — \(elapsed)ms")
        return response
    }
}
```

### API Key Guard Middleware

```swift
struct APIKeyMiddleware: AsyncMiddleware {
    func respond(to request: Request, chainingTo next: AsyncResponder) async throws -> Response {
        guard let apiKey = request.headers.first(name: "X-API-Key"),
              apiKey == Environment.get("API_KEY") else {
            throw Abort(.unauthorized, reason: "Invalid API key")
        }
        return try await next.respond(to: request)
    }
}
```

### Rate Limiting Middleware

```swift
import Vapor
import Redis

struct RateLimitMiddleware: AsyncMiddleware {
    let maxRequests: Int
    let windowSeconds: Int

    func respond(to request: Request, chainingTo next: AsyncResponder) async throws -> Response {
        let key = RedisKey("rate_limit:\(request.peerAddress?.description ?? "unknown")")
        let count = try await request.redis.increment(key).get()

        if count == 1 {
            _ = try await request.redis.expire(key, after: .seconds(Int64(windowSeconds))).get()
        }

        guard count <= maxRequests else {
            throw Abort(.tooManyRequests, reason: "Rate limit exceeded")
        }

        let response = try await next.respond(to: request)
        response.headers.add(name: "X-RateLimit-Limit", value: "\(maxRequests)")
        response.headers.add(name: "X-RateLimit-Remaining", value: "\(max(0, maxRequests - count))")
        return response
    }
}
```

## Error Handling

### Structured Error Responses

```swift
struct AppError: AbortError, DebuggableError {
    var status: HTTPResponseStatus
    var reason: String
    var identifier: String
    var source: ErrorSource?

    static func notFound(_ item: String) -> AppError {
        AppError(
            status: .notFound,
            reason: "\(item) not found",
            identifier: "not_found"
        )
    }

    static func validation(_ message: String) -> AppError {
        AppError(
            status: .badRequest,
            reason: message,
            identifier: "validation_error"
        )
    }

    static func unauthorized(_ message: String = "Unauthorized") -> AppError {
        AppError(
            status: .unauthorized,
            reason: message,
            identifier: "unauthorized"
        )
    }
}
```

### Custom Error Middleware

```swift
struct CustomErrorMiddleware: AsyncMiddleware {
    func respond(to request: Request, chainingTo next: AsyncResponder) async throws -> Response {
        do {
            return try await next.respond(to: request)
        } catch let abort as AbortError {
            let body = ErrorResponse(
                error: true,
                reason: abort.reason,
                code: abort.status.code
            )
            let response = Response(status: abort.status)
            try response.content.encode(body)
            return response
        } catch {
            request.logger.error("Unhandled error: \(error)")
            let body = ErrorResponse(
                error: true,
                reason: "Internal server error",
                code: 500
            )
            let response = Response(status: .internalServerError)
            try response.content.encode(body)
            return response
        }
    }
}

struct ErrorResponse: Content {
    let error: Bool
    let reason: String
    let code: UInt
}
```

## WebSocket Support

### Basic WebSocket Route

```swift
app.webSocket("ws", "chat") { req, ws in
    ws.onText { ws, text in
        // Broadcast to all connected clients
        ws.send("Echo: \(text)")
    }

    ws.onClose.whenComplete { _ in
        print("WebSocket closed")
    }
}
```

### WebSocket with Channel Management

```swift
actor WebSocketStore {
    private var clients: [UUID: WebSocket] = [:]

    func add(_ ws: WebSocket) -> UUID {
        let id = UUID()
        clients[id] = ws
        return id
    }

    func remove(_ id: UUID) {
        clients.removeValue(forKey: id)
    }

    func broadcast(_ message: String) async {
        for (id, ws) in clients {
            guard !ws.isClosed else {
                clients.removeValue(forKey: id)
                continue
            }
            ws.send(message)
        }
    }
}
```

## Anti-Patterns

### ❌ Blocking the Event Loop

```swift
// ❌ Bad — Synchronous file I/O on event loop
app.get("file") { req -> String in
    let data = try Data(contentsOf: URL(fileURLWithPath: "/large/file"))
    return String(data: data, encoding: .utf8)!
}

// ✅ Good — Use NIO file I/O
app.get("file") { req async throws -> Response in
    req.fileio.streamFile(at: "/large/file")
}
```

### ❌ Hardcoded Configuration

```swift
// ❌ Bad — Hardcoded values
app.databases.use(.postgres(hostname: "localhost", username: "root"), as: .psql)

// ✅ Good — Environment variables
app.databases.use(.postgres(
    hostname: Environment.get("DB_HOST") ?? "localhost",
    username: Environment.get("DB_USER") ?? "vapor"
), as: .psql)
```

### ❌ Missing Validation

```swift
// ❌ Bad — No validation
func create(req: Request) async throws -> Todo {
    let todo = try req.content.decode(Todo.self)
    try await todo.save(on: req.db)
    return todo
}

// ✅ Good — Validate input
func create(req: Request) async throws -> TodoDTO {
    try CreateTodoRequest.validate(content: req)
    let input = try req.content.decode(CreateTodoRequest.self)
    let todo = Todo(title: input.title, isComplete: false)
    try await todo.save(on: req.db)
    return todo.toDTO
}
```

## Environment Configuration

### .env File Template

```
# Server
PORT=8080
LOG_LEVEL=info

# Database
DB_HOST=localhost
DB_PORT=5432
DB_USER=vapor
DB_PASS=vapor
DB_NAME=vapor_db

# Authentication
JWT_SECRET=your-secret-key-change-in-production
API_KEY=your-api-key

# CORS
ALLOWED_ORIGINS=http://localhost:3000,https://myapp.com

# Redis (optional)
REDIS_URL=redis://localhost:6379
```

## References

- [Vapor Documentation](https://docs.vapor.codes/)
- [Vapor GitHub](https://github.com/vapor/vapor)
- **database-patterns.md** — Fluent ORM details
- **auth-patterns.md** — Authentication patterns
- **deployment-patterns.md** — Docker and cloud deployment
