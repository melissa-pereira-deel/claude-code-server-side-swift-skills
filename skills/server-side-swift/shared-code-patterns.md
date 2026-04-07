# Shared Code Patterns

SharedModels package, DTOs for client↔server communication, API endpoint protocols, and APNSwift push notifications.

## SharedModels Package

### Package Structure

```
Workspace/
├── MyApp/                    # iOS app
│   ├── MyApp.xcodeproj
│   └── Sources/
├── Server/                   # Server app
│   ├── Package.swift
│   └── Sources/App/
└── SharedModels/             # Shared package
    ├── Package.swift
    └── Sources/SharedModels/
        ├── DTOs/
        │   ├── UserDTO.swift
        │   ├── TodoDTO.swift
        │   └── AuthDTOs.swift
        ├── Endpoints/
        │   ├── APIEndpoint.swift
        │   └── TodoEndpoints.swift
        └── Errors/
            └── APIError.swift
```

### SharedModels Package.swift

```swift
// swift-tools-version:5.9
import PackageDescription

let package = Package(
    name: "SharedModels",
    platforms: [.iOS(.v17), .macOS(.v14)],
    products: [
        .library(name: "SharedModels", targets: ["SharedModels"]),
    ],
    targets: [
        .target(name: "SharedModels"),
        .testTarget(name: "SharedModelsTests", dependencies: ["SharedModels"]),
    ]
)
```

### Server Package.swift (depends on SharedModels)

```swift
// swift-tools-version:5.9
import PackageDescription

let package = Package(
    name: "Server",
    platforms: [.macOS(.v14)],
    dependencies: [
        .package(url: "https://github.com/vapor/vapor.git", from: "4.89.0"),
        .package(path: "../SharedModels"),  // Local dependency
    ],
    targets: [
        .executableTarget(
            name: "App",
            dependencies: [
                .product(name: "Vapor", package: "vapor"),
                .product(name: "SharedModels", package: "SharedModels"),
            ]
        ),
    ]
)
```

### iOS App (depends on SharedModels)

In Xcode: File → Add Package Dependencies → Add Local → Select `SharedModels/`

## Shared DTOs

### User DTOs

```swift
// SharedModels/Sources/SharedModels/DTOs/UserDTO.swift
import Foundation

public struct UserDTO: Codable, Sendable, Identifiable, Hashable {
    public let id: UUID?
    public let name: String
    public let email: String
    public let createdAt: Date?

    public init(id: UUID? = nil, name: String, email: String, createdAt: Date? = nil) {
        self.id = id
        self.name = name
        self.email = email
        self.createdAt = createdAt
    }
}

public struct CreateUserRequest: Codable, Sendable {
    public let name: String
    public let email: String
    public let password: String

    public init(name: String, email: String, password: String) {
        self.name = name
        self.email = email
        self.password = password
    }
}

public struct LoginRequest: Codable, Sendable {
    public let email: String
    public let password: String

    public init(email: String, password: String) {
        self.email = email
        self.password = password
    }
}

public struct TokenResponse: Codable, Sendable {
    public let accessToken: String
    public let refreshToken: String
    public let expiresIn: Int

    public init(accessToken: String, refreshToken: String, expiresIn: Int) {
        self.accessToken = accessToken
        self.refreshToken = refreshToken
        self.expiresIn = expiresIn
    }
}
```

### Todo DTOs

```swift
// SharedModels/Sources/SharedModels/DTOs/TodoDTO.swift
import Foundation

public struct TodoDTO: Codable, Sendable, Identifiable, Hashable {
    public let id: UUID?
    public let title: String
    public let isComplete: Bool
    public let createdAt: Date?

    public init(id: UUID? = nil, title: String, isComplete: Bool, createdAt: Date? = nil) {
        self.id = id
        self.title = title
        self.isComplete = isComplete
        self.createdAt = createdAt
    }
}

public struct CreateTodoRequest: Codable, Sendable {
    public let title: String

    public init(title: String) {
        self.title = title
    }
}

public struct UpdateTodoRequest: Codable, Sendable {
    public let title: String?
    public let isComplete: Bool?

    public init(title: String? = nil, isComplete: Bool? = nil) {
        self.title = title
        self.isComplete = isComplete
    }
}
```

### Paginated Response

```swift
// SharedModels/Sources/SharedModels/DTOs/PaginatedResponse.swift
import Foundation

public struct PaginatedResponse<T: Codable & Sendable>: Codable, Sendable {
    public let items: [T]
    public let metadata: PageMetadata

    public init(items: [T], metadata: PageMetadata) {
        self.items = items
        self.metadata = metadata
    }
}

public struct PageMetadata: Codable, Sendable {
    public let page: Int
    public let per: Int
    public let total: Int
    public let pageCount: Int

    public init(page: Int, per: Int, total: Int) {
        self.page = page
        self.per = per
        self.total = total
        self.pageCount = max(1, Int(ceil(Double(total) / Double(per))))
    }
}
```

## API Endpoint Protocol

### Endpoint Definition

```swift
// SharedModels/Sources/SharedModels/Endpoints/APIEndpoint.swift
import Foundation

public enum HTTPMethod: String, Codable, Sendable {
    case get = "GET"
    case post = "POST"
    case put = "PUT"
    case patch = "PATCH"
    case delete = "DELETE"
}

public protocol APIEndpoint: Sendable {
    associatedtype Response: Codable & Sendable
    associatedtype Body: Codable & Sendable

    var path: String { get }
    var method: HTTPMethod { get }
    var body: Body? { get }
    var queryItems: [URLQueryItem]? { get }
    var requiresAuth: Bool { get }
}

extension APIEndpoint {
    public var body: Never? { nil }
    public var queryItems: [URLQueryItem]? { nil }
    public var requiresAuth: Bool { true }
}

// For endpoints with no body
extension Never: Codable {
    public init(from decoder: Decoder) throws {
        fatalError("Never should not be decoded")
    }
    public func encode(to encoder: Encoder) throws {
        fatalError("Never should not be encoded")
    }
}
```

### Endpoint Definitions

```swift
// SharedModels/Sources/SharedModels/Endpoints/TodoEndpoints.swift
import Foundation

public enum TodoEndpoints {
    public struct List: APIEndpoint {
        public typealias Response = PaginatedResponse<TodoDTO>
        public typealias Body = Never

        public let path = "/api/v1/todos"
        public let method: HTTPMethod = .get
        public let queryItems: [URLQueryItem]?

        public init(page: Int = 1, per: Int = 20) {
            self.queryItems = [
                URLQueryItem(name: "page", value: "\(page)"),
                URLQueryItem(name: "per", value: "\(per)")
            ]
        }
    }

    public struct Get: APIEndpoint {
        public typealias Response = TodoDTO
        public typealias Body = Never

        public let path: String
        public let method: HTTPMethod = .get

        public init(id: UUID) {
            self.path = "/api/v1/todos/\(id)"
        }
    }

    public struct Create: APIEndpoint {
        public typealias Response = TodoDTO

        public let path = "/api/v1/todos"
        public let method: HTTPMethod = .post
        public let body: CreateTodoRequest?

        public init(title: String) {
            self.body = CreateTodoRequest(title: title)
        }
    }

    public struct Update: APIEndpoint {
        public typealias Response = TodoDTO

        public let path: String
        public let method: HTTPMethod = .put
        public let body: UpdateTodoRequest?

        public init(id: UUID, title: String? = nil, isComplete: Bool? = nil) {
            self.path = "/api/v1/todos/\(id)"
            self.body = UpdateTodoRequest(title: title, isComplete: isComplete)
        }
    }

    public struct Delete: APIEndpoint {
        public typealias Response = EmptyResponse
        public typealias Body = Never

        public let path: String
        public let method: HTTPMethod = .delete

        public init(id: UUID) {
            self.path = "/api/v1/todos/\(id)"
        }
    }
}

public struct EmptyResponse: Codable, Sendable {}
```

## iOS API Client

### Generic API Client

```swift
// iOS App — APIClient.swift
import Foundation
import SharedModels

actor APIClient {
    private let baseURL: URL
    private let session: URLSession
    private var accessToken: String?

    init(baseURL: URL, session: URLSession = .shared) {
        self.baseURL = baseURL
        self.session = session
    }

    func setToken(_ token: String?) {
        self.accessToken = token
    }

    func request<E: APIEndpoint>(_ endpoint: E) async throws -> E.Response {
        var url = baseURL.appendingPathComponent(endpoint.path)

        if let queryItems = endpoint.queryItems {
            var components = URLComponents(url: url, resolvingAgainstBaseURL: false)!
            components.queryItems = queryItems
            url = components.url!
        }

        var request = URLRequest(url: url)
        request.httpMethod = endpoint.method.rawValue
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")

        if endpoint.requiresAuth, let token = accessToken {
            request.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
        }

        if let body = endpoint.body {
            request.httpBody = try JSONEncoder.api.encode(body)
        }

        let (data, response) = try await session.data(for: request)

        guard let httpResponse = response as? HTTPURLResponse else {
            throw APIError.invalidResponse
        }

        guard (200...299).contains(httpResponse.statusCode) else {
            if let errorResponse = try? JSONDecoder.api.decode(ErrorResponse.self, from: data) {
                throw APIError.server(statusCode: httpResponse.statusCode, message: errorResponse.reason)
            }
            throw APIError.httpError(statusCode: httpResponse.statusCode)
        }

        // Handle empty response
        if E.Response.self == EmptyResponse.self {
            return EmptyResponse() as! E.Response
        }

        return try JSONDecoder.api.decode(E.Response.self, from: data)
    }
}
```

### JSON Encoder/Decoder Configuration

```swift
// SharedModels/Sources/SharedModels/JSONCoding.swift
import Foundation

extension JSONEncoder {
    public static let api: JSONEncoder = {
        let encoder = JSONEncoder()
        encoder.dateEncodingStrategy = .iso8601
        encoder.keyEncodingStrategy = .convertToSnakeCase
        return encoder
    }()
}

extension JSONDecoder {
    public static let api: JSONDecoder = {
        let decoder = JSONDecoder()
        decoder.dateDecodingStrategy = .iso8601
        decoder.keyDecodingStrategy = .convertFromSnakeCase
        return decoder
    }()
}
```

### Usage in SwiftUI

```swift
// iOS App — TodoListView.swift
import SwiftUI
import SharedModels

@Observable
class TodoListViewModel {
    var todos: [TodoDTO] = []
    var isLoading = false
    var error: String?

    private let client: APIClient

    init(client: APIClient) {
        self.client = client
    }

    func loadTodos() async {
        isLoading = true
        defer { isLoading = false }

        do {
            let response = try await client.request(TodoEndpoints.List())
            todos = response.items
        } catch {
            self.error = error.localizedDescription
        }
    }

    func createTodo(title: String) async {
        do {
            let todo = try await client.request(TodoEndpoints.Create(title: title))
            todos.insert(todo, at: 0)
        } catch {
            self.error = error.localizedDescription
        }
    }

    func toggleComplete(_ todo: TodoDTO) async {
        guard let id = todo.id else { return }
        do {
            let updated = try await client.request(
                TodoEndpoints.Update(id: id, isComplete: !todo.isComplete)
            )
            if let index = todos.firstIndex(where: { $0.id == id }) {
                todos[index] = updated
            }
        } catch {
            self.error = error.localizedDescription
        }
    }
}
```

## APNSwift — Push Notifications

### Server Setup

```swift
// Package.swift dependency
.package(url: "https://github.com/swift-server-community/APNSwift.git", from: "5.0.0"),
// Target:
.product(name: "APNSwift", package: "APNSwift"),
```

### APNs Configuration

```swift
import APNSwift
import Vapor

func configureAPNs(_ app: Application) async throws {
    let apnsConfig = APNSClientConfiguration(
        authenticationMethod: .jwt(
            privateKey: try .loadFrom(filePath: Environment.get("APNS_KEY_PATH")!),
            keyIdentifier: Environment.get("APNS_KEY_ID")!,
            teamIdentifier: Environment.get("APNS_TEAM_ID")!
        ),
        environment: app.environment == .production ? .production : .sandbox
    )

    app.apns.containers.use(
        apnsConfig,
        eventLoopGroupProvider: .shared(app.eventLoopGroup),
        responseDecoder: JSONDecoder(),
        requestEncoder: JSONEncoder(),
        as: .default
    )
}
```

### Send Push Notification

```swift
import APNSwift

struct PushNotificationService {
    let app: Application

    func sendPush(to deviceToken: String, title: String, body: String) async throws {
        let alert = APNSAlertNotification(
            alert: .init(title: .raw(title), body: .raw(body)),
            expiration: .immediately,
            priority: .immediately,
            topic: Environment.get("APNS_TOPIC")!
        )

        try await app.apns.client.sendAlertNotification(
            alert,
            deviceToken: deviceToken
        )
    }

    func sendSilentPush(to deviceToken: String, data: [String: String]) async throws {
        let notification = APNSBackgroundNotification(
            expiration: .immediately,
            topic: Environment.get("APNS_TOPIC")!,
            payload: data
        )

        try await app.apns.client.sendBackgroundNotification(
            notification,
            deviceToken: deviceToken
        )
    }
}
```

### Device Token Storage

```swift
// Shared DTO
public struct RegisterDeviceRequest: Codable, Sendable {
    public let token: String
    public let platform: String  // "ios", "macos"

    public init(token: String, platform: String) {
        self.token = token
        self.platform = platform
    }
}

// Server endpoint
app.post("api", "v1", "devices") { req async throws -> HTTPStatus in
    let user = try req.auth.require(UserPayload.self)
    let input = try req.content.decode(RegisterDeviceRequest.self)

    // Upsert device token
    if let existing = try await DeviceToken.query(on: req.db)
        .filter(\.$token == input.token)
        .first() {
        existing.$user.id = user.userID
        try await existing.save(on: req.db)
    } else {
        let device = DeviceToken(
            token: input.token,
            platform: input.platform,
            userID: user.userID
        )
        try await device.save(on: req.db)
    }

    return .ok
}
```

## Shared API Error

```swift
// SharedModels/Sources/SharedModels/Errors/APIError.swift
import Foundation

public enum APIError: LocalizedError, Sendable {
    case invalidResponse
    case httpError(statusCode: Int)
    case server(statusCode: Int, message: String)
    case decodingError(String)
    case networkError(String)

    public var errorDescription: String? {
        switch self {
        case .invalidResponse:
            return "Invalid server response"
        case .httpError(let statusCode):
            return "HTTP error: \(statusCode)"
        case .server(_, let message):
            return message
        case .decodingError(let message):
            return "Decoding error: \(message)"
        case .networkError(let message):
            return "Network error: \(message)"
        }
    }
}

public struct ErrorResponse: Codable, Sendable {
    public let error: Bool
    public let reason: String
    public let code: UInt?

    public init(error: Bool, reason: String, code: UInt? = nil) {
        self.error = error
        self.reason = reason
        self.code = code
    }
}
```

## Anti-Patterns

### ❌ Exposing Database Models to Client

```swift
// ❌ Bad — Client knows about Fluent internals
import Fluent
public final class Todo: Model { ... }

// ✅ Good — Shared lightweight DTOs only
public struct TodoDTO: Codable, Sendable { ... }
```

### ❌ Different Coding Strategies Client vs Server

```swift
// ❌ Bad — Server uses snake_case, client uses camelCase
// This causes silent decode failures

// ✅ Good — Use shared JSONEncoder/JSONDecoder configuration
// Define once in SharedModels, use everywhere
```

### ❌ Non-Sendable Shared Types

```swift
// ❌ Bad — Not Sendable, causes issues with Swift 6
public class UserDTO { ... }

// ✅ Good — All shared types are structs and Sendable
public struct UserDTO: Codable, Sendable { ... }
```

## References

- [APNSwift Documentation](https://github.com/swift-server-community/APNSwift)
- [Swift Package Manager](https://www.swift.org/documentation/package-manager/)
- **vapor-patterns.md** — Server-side Vapor setup
- **auth-patterns.md** — Token management
