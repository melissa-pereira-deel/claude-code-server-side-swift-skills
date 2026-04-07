# Deployment Patterns

Docker, docker-compose, Fly.io, Railway, and production configuration for server-side Swift.

## Docker

### Multi-Stage Dockerfile (Vapor)

```dockerfile
# ================================
# Build stage
# ================================
FROM swift:5.10-jammy AS build

RUN export DEBIAN_FRONTEND=noninteractive DEBCONF_NONINTERACTIVE_SEEN=true \
    && apt-get -q update \
    && apt-get -q dist-upgrade -y \
    && apt-get install -y libssl-dev \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /build

# Resolve dependencies first (cache layer)
COPY Package.swift Package.resolved ./
RUN swift package resolve --skip-update

# Copy source and build
COPY . .
RUN swift build -c release --static-swift-stdlib

# Create staging area
WORKDIR /staging
RUN cp "$(swift build --package-path /build -c release --show-bin-path)/App" ./
RUN cp /build/Public ./Public -r 2>/dev/null || true
RUN cp /build/Resources ./Resources -r 2>/dev/null || true

# ================================
# Run stage
# ================================
FROM ubuntu:jammy

RUN export DEBIAN_FRONTEND=noninteractive DEBCONF_NONINTERACTIVE_SEEN=true \
    && apt-get -q update \
    && apt-get -q dist-upgrade -y \
    && apt-get -q install -y \
      ca-certificates \
      tzdata \
      libcurl4 \
    && rm -rf /var/lib/apt/lists/*

RUN useradd --user-group --create-home --system --skel /dev/null --home-dir /app vapor

WORKDIR /app
COPY --from=build --chown=vapor:vapor /staging /app

USER vapor:vapor
EXPOSE 8080

ENTRYPOINT ["./App"]
CMD ["serve", "--env", "production", "--hostname", "0.0.0.0", "--port", "8080"]
```

### Hummingbird Dockerfile

```dockerfile
FROM swift:5.10-jammy AS build

WORKDIR /build
COPY Package.swift Package.resolved ./
RUN swift package resolve --skip-update

COPY . .
RUN swift build -c release --static-swift-stdlib

FROM ubuntu:jammy

RUN apt-get -q update && apt-get -q dist-upgrade -y \
    && apt-get -q install -y ca-certificates tzdata \
    && rm -rf /var/lib/apt/lists/*

RUN useradd --user-group --create-home --system --home-dir /app hb

WORKDIR /app
COPY --from=build /build/.build/release/App ./

USER hb:hb
EXPOSE 8080

ENTRYPOINT ["./App"]
```

## Docker Compose

### Development Setup

```yaml
# docker-compose.yml
version: "3.8"

services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - LOG_LEVEL=debug
      - DB_HOST=db
      - DB_PORT=5432
      - DB_USER=vapor
      - DB_PASS=vapor
      - DB_NAME=vapor_db
      - JWT_SECRET=${JWT_SECRET:-dev-secret}
      - REDIS_URL=redis://redis:6379
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      - .:/app  # Mount source for development
    command: ["serve", "--env", "development", "--hostname", "0.0.0.0", "--port", "8080"]

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: vapor
      POSTGRES_PASSWORD: vapor
      POSTGRES_DB: vapor_db
    ports:
      - "5432:5432"
    volumes:
      - db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U vapor -d vapor_db"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5

  migrate:
    build: .
    environment:
      - DB_HOST=db
      - DB_PORT=5432
      - DB_USER=vapor
      - DB_PASS=vapor
      - DB_NAME=vapor_db
    depends_on:
      db:
        condition: service_healthy
    command: ["migrate", "--yes"]
    profiles:
      - tools

volumes:
  db_data:
```

### Production Docker Compose

```yaml
version: "3.8"

services:
  app:
    build: .
    ports:
      - "8080:8080"
    env_file: .env.production
    deploy:
      replicas: 2
      restart_policy:
        condition: on-failure
        max_attempts: 3
    depends_on:
      db:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  db:
    image: postgres:16-alpine
    env_file: .env.production
    volumes:
      - db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  db_data:
    driver: local
```

## Fly.io Deployment

### fly.toml

```toml
app = "my-swift-app"
primary_region = "iad"

[build]

[env]
  LOG_LEVEL = "info"
  PORT = "8080"

[http_service]
  internal_port = 8080
  force_https = true
  auto_stop_machines = true
  auto_start_machines = true
  min_machines_running = 1
  processes = ["app"]

[[services]]
  protocol = "tcp"
  internal_port = 8080

  [[services.ports]]
    port = 80
    handlers = ["http"]

  [[services.ports]]
    port = 443
    handlers = ["tls", "http"]

  [services.concurrency]
    type = "connections"
    hard_limit = 250
    soft_limit = 200

  [[services.tcp_checks]]
    interval = "15s"
    timeout = "2s"
    grace_period = "30s"

[[vm]]
  memory = "512mb"
  cpu_kind = "shared"
  cpus = 1
```

### Fly.io Commands

```bash
# Install flyctl
curl -L https://fly.io/install.sh | sh

# Login
fly auth login

# Launch (first time)
fly launch --name my-swift-app --region iad

# Create PostgreSQL
fly postgres create --name my-swift-db --region iad
fly postgres attach my-swift-db --app my-swift-app

# Set secrets
fly secrets set JWT_SECRET="your-production-secret"
fly secrets set API_KEY="your-api-key"

# Deploy
fly deploy

# Run migrations
fly ssh console -C "./App migrate --yes"

# Logs
fly logs

# Scale
fly scale count 2
fly scale vm shared-cpu-2x --memory 1024
```

## Railway Deployment

### railway.toml

```toml
[build]
builder = "dockerfile"
dockerfilePath = "Dockerfile"

[deploy]
startCommand = "./App serve --env production --hostname 0.0.0.0 --port $PORT"
healthcheckPath = "/health"
healthcheckTimeout = 30
restartPolicyType = "on_failure"
restartPolicyMaxRetries = 3
```

### Railway Commands

```bash
# Install Railway CLI
npm install -g @railway/cli

# Login
railway login

# Create project
railway init

# Add PostgreSQL
railway add --plugin postgresql

# Deploy
railway up

# Set environment variables
railway variables set JWT_SECRET="your-secret"

# Logs
railway logs
```

## Environment Configuration

### .env Template

```
# Server
PORT=8080
LOG_LEVEL=info
ENVIRONMENT=production

# Database
DB_HOST=localhost
DB_PORT=5432
DB_USER=vapor
DB_PASS=change-in-production
DB_NAME=app_db

# Authentication
JWT_SECRET=generate-a-strong-random-string-here
API_KEY=generate-a-strong-random-string-here

# CORS
ALLOWED_ORIGINS=https://myapp.com,https://www.myapp.com

# Redis (optional)
REDIS_URL=redis://localhost:6379

# Push Notifications (optional)
APNS_KEY_ID=ABC123DEFG
APNS_TEAM_ID=TEAMID1234
APNS_PRIVATE_KEY_PATH=./AuthKey_ABC123DEFG.p8
APNS_TOPIC=com.myapp.bundle
```

### Environment Detection in Swift

```swift
// configure.swift
func configure(_ app: Application) async throws {
    switch app.environment {
    case .production:
        app.logger.logLevel = .info
        // Use SSL for database
    case .development:
        app.logger.logLevel = .debug
        // Auto-migrate in development
        try await app.autoMigrate()
    case .testing:
        app.logger.logLevel = .debug
        // Use in-memory SQLite
        app.databases.use(.sqlite(.memory), as: .sqlite)
    default:
        break
    }
}
```

## Health Check Endpoint

```swift
// Always include a health check
app.get("health") { req async -> HealthResponse in
    var dbHealthy = false
    do {
        try await req.db.execute(query: .init(unsafeSQL: "SELECT 1"))
        dbHealthy = true
    } catch {
        req.logger.error("Database health check failed: \(error)")
    }

    return HealthResponse(
        status: dbHealthy ? "healthy" : "degraded",
        version: "1.0.0",
        database: dbHealthy ? "connected" : "disconnected",
        uptime: ProcessInfo.processInfo.systemUptime
    )
}

struct HealthResponse: Content {
    let status: String
    let version: String
    let database: String
    let uptime: TimeInterval
}
```

## SSL/TLS Configuration

### Production HTTPS

```swift
// For direct SSL termination (rarely needed — use reverse proxy)
import NIOSSL

let certChain = try NIOSSLCertificate.fromPEMFile("/path/to/cert.pem")
let privateKey = try NIOSSLPrivateKey(file: "/path/to/key.pem", format: .pem)

app.http.server.configuration.tlsConfiguration = .makeServerConfiguration(
    certificateChain: certChain.map { .certificate($0) },
    privateKey: .privateKey(privateKey)
)
```

## Logging Configuration

```swift
import Logging

// Custom log handler
func configure(_ app: Application) async throws {
    // JSON logging for production
    if app.environment == .production {
        LoggingSystem.bootstrap { label in
            var handler = StreamLogHandler.standardOutput(label: label)
            handler.logLevel = .info
            return handler
        }
    }
}
```

## Production Checklist

### Pre-Deployment

- [ ] Environment variables set (no hardcoded secrets)
- [ ] Database connection pool configured
- [ ] CORS origins restricted to actual domains
- [ ] Health check endpoint returns meaningful data
- [ ] Error middleware does not leak stack traces in production
- [ ] Logging level set to info or warning (not debug)
- [ ] Rate limiting enabled on auth endpoints

### Deployment

- [ ] Multi-stage Docker build (small final image)
- [ ] Database migrations run before new version starts
- [ ] SSL/TLS configured (via reverse proxy or platform)
- [ ] Auto-restart on failure configured
- [ ] Resource limits set (memory, CPU)

### Post-Deployment

- [ ] Health check responding correctly
- [ ] Logs flowing to monitoring system
- [ ] Database backups configured
- [ ] Alerting set up for errors/downtime

## Anti-Patterns

### ❌ Running Migrations on App Start in Production

```swift
// ❌ Bad — Can cause issues with multiple instances
func configure(_ app: Application) async throws {
    try await app.autoMigrate()  // Don't do this in production!
}

// ✅ Good — Run migrations separately
// fly ssh console -C "./App migrate --yes"
// Or use a separate migration job in docker-compose
```

### ❌ No Health Check

```swift
// ❌ Bad — Platform can't detect if your app is healthy

// ✅ Good — Always have a health endpoint
app.get("health") { req async -> HTTPStatus in .ok }
```

## References

- [Vapor Deployment Guide](https://docs.vapor.codes/deploy/docker/)
- [Fly.io Swift Guide](https://fly.io/docs/languages-and-frameworks/swift/)
- [Railway Documentation](https://docs.railway.app/)
- **vapor-patterns.md** — App configuration
- **hummingbird-patterns.md** — Hummingbird deployment
