# Authentication Patterns

JWT, sessions, Bearer tokens, and Sign in with Apple server-side validation.

## JWT Authentication (Recommended for Mobile APIs)

### Package Setup

```swift
// Package.swift
.package(url: "https://github.com/vapor/jwt.git", from: "4.2.0"),
// In target dependencies:
.product(name: "JWT", package: "jwt"),
```

### JWT Configuration

```swift
// configure.swift
import JWT

func configure(_ app: Application) async throws {
    // Symmetric key (HMAC)
    await app.jwt.keys.add(hmac: HMACKey(from: Environment.get("JWT_SECRET") ?? "dev-secret"),
                           digestAlgorithm: .sha256)

    // Or asymmetric (RSA) for production
    // let rsaKey = try Insecure.RSA.PrivateKey(pem: rsaPEM)
    // await app.jwt.keys.add(rsa: rsaKey, digestAlgorithm: .sha256)
}
```

### JWT Payload

```swift
import JWT

struct UserPayload: JWTPayload, Authenticatable {
    var subject: SubjectClaim       // User ID
    var expiration: ExpirationClaim
    var isAdmin: Bool

    func verify(using algorithm: some JWTAlgorithm) async throws {
        try expiration.verifyNotExpired()
    }

    var userID: UUID {
        UUID(uuidString: subject.value)!
    }
}
```

### Token Generation

```swift
struct AuthController: RouteCollection {
    func boot(routes: RoutesBuilder) throws {
        routes.post("login", use: login)
        routes.post("register", use: register)
        routes.post("refresh", use: refresh)
    }

    @Sendable
    func login(req: Request) async throws -> TokenResponse {
        let input = try req.content.decode(LoginRequest.self)

        guard let user = try await User.query(on: req.db)
            .filter(\.$email == input.email)
            .first() else {
            throw Abort(.unauthorized, reason: "Invalid credentials")
        }

        guard try Bcrypt.verify(input.password, created: user.passwordHash) else {
            throw Abort(.unauthorized, reason: "Invalid credentials")
        }

        let payload = UserPayload(
            subject: .init(value: user.id!.uuidString),
            expiration: .init(value: Date().addingTimeInterval(3600)),  // 1 hour
            isAdmin: user.isAdmin
        )

        let accessToken = try await req.jwt.sign(payload)

        // Generate refresh token
        let refreshToken = try await generateRefreshToken(for: user, on: req.db)

        return TokenResponse(
            accessToken: accessToken,
            refreshToken: refreshToken,
            expiresIn: 3600
        )
    }

    @Sendable
    func register(req: Request) async throws -> TokenResponse {
        try RegisterRequest.validate(content: req)
        let input = try req.content.decode(RegisterRequest.self)

        // Check for existing user
        guard try await User.query(on: req.db)
            .filter(\.$email == input.email)
            .first() == nil else {
            throw Abort(.conflict, reason: "Email already registered")
        }

        let passwordHash = try Bcrypt.hash(input.password)
        let user = User(name: input.name, email: input.email, passwordHash: passwordHash)
        try await user.save(on: req.db)

        // Generate tokens (same as login)
        let payload = UserPayload(
            subject: .init(value: user.id!.uuidString),
            expiration: .init(value: Date().addingTimeInterval(3600)),
            isAdmin: false
        )

        let accessToken = try await req.jwt.sign(payload)
        let refreshToken = try await generateRefreshToken(for: user, on: req.db)

        return TokenResponse(
            accessToken: accessToken,
            refreshToken: refreshToken,
            expiresIn: 3600
        )
    }

    private func generateRefreshToken(for user: User, on db: Database) async throws -> String {
        let token = [UInt8].random(count: 32).base64
        let refreshToken = RefreshToken(
            token: token,
            userID: user.id!,
            expiresAt: Date().addingTimeInterval(30 * 24 * 3600) // 30 days
        )
        try await refreshToken.save(on: db)
        return token
    }
}
```

### DTOs

```swift
struct LoginRequest: Content {
    let email: String
    let password: String
}

struct RegisterRequest: Content, Validatable {
    let name: String
    let email: String
    let password: String

    static func validations(_ validations: inout Validations) {
        validations.add("name", as: String.self, is: !.empty && .count(2...100))
        validations.add("email", as: String.self, is: .email)
        validations.add("password", as: String.self, is: .count(8...))
    }
}

struct TokenResponse: Content {
    let accessToken: String
    let refreshToken: String
    let expiresIn: Int
}
```

### Protected Routes

```swift
// routes.swift
func routes(_ app: Application) throws {
    let api = app.grouped("api", "v1")

    // Public
    try api.grouped("auth").register(collection: AuthController())

    // Protected
    let protected = api.grouped(UserPayload.authenticator(), UserPayload.guardMiddleware())
    try protected.register(collection: TodoController())

    // Admin only
    let admin = protected.grouped(AdminMiddleware())
    try admin.register(collection: AdminController())
}

// Access user in controllers
@Sendable
func index(req: Request) async throws -> [TodoDTO] {
    let user = try req.auth.require(UserPayload.self)
    return try await Todo.query(on: req.db)
        .filter(\.$user.$id == user.userID)
        .all()
        .map(\.toDTO)
}
```

### Admin Middleware

```swift
struct AdminMiddleware: AsyncMiddleware {
    func respond(to request: Request, chainingTo next: AsyncResponder) async throws -> Response {
        let user = try request.auth.require(UserPayload.self)
        guard user.isAdmin else {
            throw Abort(.forbidden, reason: "Admin access required")
        }
        return try await next.respond(to: request)
    }
}
```

## Bearer Token Authentication (Simple)

### Token Model

```swift
final class UserToken: Model, Content, @unchecked Sendable {
    static let schema = "user_tokens"

    @ID(key: .id)
    var id: UUID?

    @Field(key: "value")
    var value: String

    @Parent(key: "user_id")
    var user: User

    @Timestamp(key: "expires_at", on: .none)
    var expiresAt: Date?

    init() {}

    init(id: UUID? = nil, value: String, userID: UUID, expiresAt: Date) {
        self.id = id
        self.value = value
        self.$user.id = userID
        self.expiresAt = expiresAt
    }
}

extension UserToken: ModelTokenAuthenticatable {
    static let valueKey = \UserToken.$value
    static let userKey = \UserToken.$user
    typealias User = App.User

    var isValid: Bool {
        guard let expiresAt else { return false }
        return expiresAt > Date()
    }
}
```

### Authenticatable User

```swift
extension User: ModelAuthenticatable {
    static let usernameKey = \User.$email
    static let passwordHashKey = \User.$passwordHash

    func verify(password: String) throws -> Bool {
        try Bcrypt.verify(password, created: passwordHash)
    }
}
```

### Route Protection

```swift
let tokenProtected = api.grouped(UserToken.authenticator(), User.guardMiddleware())
try tokenProtected.register(collection: ProfileController())
```

## Sign in with Apple (Server-Side Validation)

### Apple Token Verification

```swift
import JWT

struct AppleIdentityToken: JWTPayload {
    let iss: IssuerClaim          // "https://appleid.apple.com"
    let sub: SubjectClaim         // Apple user identifier
    let aud: AudienceClaim        // Your app's bundle ID
    let iat: IssuedAtClaim
    let exp: ExpirationClaim
    let email: String?
    let emailVerified: String?
    let isPrivateEmail: String?

    enum CodingKeys: String, CodingKey {
        case iss, sub, aud, iat, exp, email
        case emailVerified = "email_verified"
        case isPrivateEmail = "is_private_email"
    }

    func verify(using algorithm: some JWTAlgorithm) async throws {
        try exp.verifyNotExpired()
        guard iss.value == "https://appleid.apple.com" else {
            throw JWTError.claimVerificationFailure(
                failedClaim: iss,
                reason: "Invalid issuer"
            )
        }
    }
}
```

### Apple Auth Controller

```swift
struct AppleAuthController: RouteCollection {
    func boot(routes: RoutesBuilder) throws {
        routes.post("auth", "apple", use: signInWithApple)
    }

    @Sendable
    func signInWithApple(req: Request) async throws -> TokenResponse {
        let input = try req.content.decode(AppleSignInRequest.self)

        // Fetch Apple's public keys and verify
        let appleToken = try await req.jwt.apple.verify(
            input.identityToken,
            applicationIdentifier: Environment.get("APPLE_APP_ID")!
        )

        let appleUserID = appleToken.subject.value

        // Find or create user
        let user: User
        if let existing = try await User.query(on: req.db)
            .filter(\.$appleID == appleUserID)
            .first() {
            user = existing
            // Update email if provided (first sign-in only)
            if let email = appleToken.email, user.email.isEmpty {
                user.email = email
                try await user.save(on: req.db)
            }
        } else {
            user = User(
                name: input.fullName ?? "Apple User",
                email: appleToken.email ?? "",
                appleID: appleUserID
            )
            try await user.save(on: req.db)
        }

        // Generate JWT
        let payload = UserPayload(
            subject: .init(value: user.id!.uuidString),
            expiration: .init(value: Date().addingTimeInterval(3600)),
            isAdmin: false
        )

        let accessToken = try await req.jwt.sign(payload)
        let refreshToken = try await generateRefreshToken(for: user, on: req.db)

        return TokenResponse(
            accessToken: accessToken,
            refreshToken: refreshToken,
            expiresIn: 3600
        )
    }
}

struct AppleSignInRequest: Content {
    let identityToken: String
    let authorizationCode: String
    let fullName: String?
}
```

## Password Hashing

### Always Use Bcrypt

```swift
// ✅ Hash password on registration
let hash = try Bcrypt.hash(input.password)
let user = User(email: input.email, passwordHash: hash)

// ✅ Verify on login
guard try Bcrypt.verify(input.password, created: user.passwordHash) else {
    throw Abort(.unauthorized)
}
```

```swift
// ❌ Bad — Never store plain text
user.password = input.password

// ❌ Bad — Don't use SHA256 for passwords
user.passwordHash = SHA256.hash(data: Data(input.password.utf8)).hexString
```

## Token Refresh Pattern

```swift
@Sendable
func refresh(req: Request) async throws -> TokenResponse {
    let input = try req.content.decode(RefreshRequest.self)

    guard let token = try await RefreshToken.query(on: req.db)
        .filter(\.$token == input.refreshToken)
        .with(\.$user)
        .first() else {
        throw Abort(.unauthorized, reason: "Invalid refresh token")
    }

    guard let expiresAt = token.expiresAt, expiresAt > Date() else {
        try await token.delete(on: req.db)
        throw Abort(.unauthorized, reason: "Refresh token expired")
    }

    // Delete old refresh token (rotate)
    try await token.delete(on: req.db)

    // Generate new tokens
    let payload = UserPayload(
        subject: .init(value: token.user.id!.uuidString),
        expiration: .init(value: Date().addingTimeInterval(3600)),
        isAdmin: token.user.isAdmin
    )

    let accessToken = try await req.jwt.sign(payload)
    let newRefreshToken = try await generateRefreshToken(for: token.user, on: req.db)

    return TokenResponse(
        accessToken: accessToken,
        refreshToken: newRefreshToken,
        expiresIn: 3600
    )
}

struct RefreshRequest: Content {
    let refreshToken: String
}
```

## Security Best Practices

### ✅ Do

- Always use HTTPS in production
- Hash passwords with Bcrypt (cost factor 12+)
- Use short-lived access tokens (1 hour)
- Rotate refresh tokens on use
- Validate all input with `Validatable`
- Use environment variables for secrets
- Implement rate limiting on auth endpoints
- Log failed login attempts

### ❌ Don't

- Store plain text passwords
- Use long-lived access tokens
- Skip token expiration checks
- Hardcode secrets in source code
- Return different errors for "user not found" vs "wrong password" (timing attack)
- Allow unlimited login attempts

### Constant-Time Comparison

```swift
// ✅ Good — Same error for both cases (prevents user enumeration)
@Sendable
func login(req: Request) async throws -> TokenResponse {
    let input = try req.content.decode(LoginRequest.self)

    let user = try await User.query(on: req.db)
        .filter(\.$email == input.email)
        .first()

    // Always hash-check even if user not found (constant time)
    let dummyHash = try Bcrypt.hash("dummy")
    let hash = user?.passwordHash ?? dummyHash
    guard try Bcrypt.verify(input.password, created: hash), user != nil else {
        throw Abort(.unauthorized, reason: "Invalid credentials")
    }

    // ... generate tokens
}
```

## References

- [Vapor JWT Documentation](https://docs.vapor.codes/security/jwt/)
- [Vapor Authentication](https://docs.vapor.codes/security/authentication/)
- [Sign in with Apple - Server](https://developer.apple.com/documentation/sign_in_with_apple/sign_in_with_apple_rest_api)
- **vapor-patterns.md** — Route protection setup
- **testing-patterns.md** — Testing auth endpoints
