# Architecture Decisions: Server-Side Swift

A decision framework for software architects choosing server-side technologies. This file helps Claude Code present trade-offs and facilitate informed decisions — not make arbitrary tool choices.

## When Server-Side Swift Makes Sense

### The Three Strong Signals

Before choosing a Swift framework, validate that Swift itself is the right backend language:

1. **You already have a Swift team.** The strongest signal. iOS/macOS developers can share skills across client and server. Cultured Code (Things app) migrated from Python to Swift — their iOS team now owns the full stack.

2. **Client-server code sharing provides real value.** Shared models, validation logic, business rules, and API contracts between your iOS app and backend eliminate an entire class of serialization bugs and reduce duplication.

3. **You need compiled-language performance without GC overhead.** Apple's Password Monitoring Service migrated from Java to Swift and achieved 40% throughput increase, ~50% memory reduction, and sub-millisecond p99.9 latencies — with 85% less code.

**If none of these apply, server-side Swift is likely not the right choice.** Consider Go, Node.js, or Python instead.

### Real-World Production Evidence

| Company | Scale | Migration | Results |
|---------|-------|-----------|---------|
| **Apple** (Password Monitoring) | Billions of requests/day | Java → Swift | 40% throughput ↑, 50% memory ↓, 85% less code |
| **Cultured Code** (Things Cloud) | ~500 req/s on 4-node K8s | Python 2 → Swift/Vapor | 4x faster responses, 3x lower compute cost |

Sources: [swift.org/blog](https://www.swift.org/blog/swift-at-apple-migrating-the-password-monitoring-service-from-java/), [culturedcode.com/things/blog](https://culturedcode.com/things/blog/2025/05/a-swift-cloud/)

## When NOT to Use Server-Side Swift

Present these honestly to the architect. A good decision framework includes disqualifiers:

### 1. You need to hire quickly from a large talent pool
Server-side Swift developers are rare. The 2023 SSWG survey received hundreds of responses — compare to millions of Node.js or Python developers. Only ~2% of freelance server-side Swift devs pass vetting on platforms like Toptal. If you need to scale a backend team from 3 to 15 engineers in 6 months, Go or Node.js are safer hiring bets.

### 2. Your backend is not connected to an Apple client
The primary advantage — shared code with iOS/macOS — vanishes if your clients are web, Android, or cross-platform. Without code sharing, you inherit Swift's smaller ecosystem with none of the benefits.

### 3. You depend on many third-party integrations
Kafka, Elasticsearch, payment processors, email services, SMS — Node.js and Python have mature, well-maintained SDKs for nearly everything. Swift may have gaps that force you to write C wrappers or shell out to another language. Cultured Code had to use Python for email processing because mature Swift email libraries don't exist.

### 4. Fast iteration cycles matter more than runtime performance
Swift compile times are the #1 complaint. Cultured Code reported ~10-minute builds for 30,000 lines. Vapor's dependency graph compounds this. For rapid prototyping or early-stage startups where shipping speed > runtime efficiency, Node.js or Python offer faster development velocity.

### 5. Your infrastructure is Windows-based
Swift's server story is Linux and macOS. Windows support is a 2025 SSWG goal but not yet mature.

## Swift vs Alternatives: Honest Comparison

| Criterion | Swift | Go | Node.js | Python/FastAPI |
|-----------|-------|----|---------|----------------|
| **Performance** | Compiled, near-C, no GC pauses | Compiled, excellent concurrency | Single-threaded event loop, good I/O | Interpreted, slowest raw performance |
| **Ecosystem maturity** | Small but growing; gaps in libraries | Extensive stdlib; mature | Massive npm ecosystem | Enormous; dominant in ML/AI |
| **Hiring pool** | Very small for server-side | Large and growing | Largest backend talent pool | Huge; 75%+ ML practitioners |
| **Type safety** | Strong; compile-time guarantees | Strong; simpler type system | Weak (TypeScript helps) | Dynamic; optional typing |
| **Concurrency** | Structured (async/await, actors) | Goroutines (proven, simple) | Event loop + async/await | asyncio; GIL limitations |
| **Cold start** | Fast (no runtime) | Fast (small binaries) | Medium (V8 init) | Slow (interpreter init) |
| **Build time** | Slow (type checker, large deps) | Fast | N/A (interpreted) | N/A (interpreted) |
| **Code sharing with iOS** | Native | None | None | None |
| **Best for** | iOS-connected backends, perf-critical | General-purpose backends | Rapid development, ecosystem | ML/AI integration, scripting |

### Decision by Scenario

| Scenario | Recommended | Why |
|----------|-------------|-----|
| Solo indie dev with iOS app needing a backend | **Swift (Vapor)** | Same language, shared models, one person owns everything |
| Startup MVP, need to ship fast, web + mobile | **Node.js/TypeScript** | Fastest iteration, largest ecosystem, one language for web + API |
| Microservice in an existing infrastructure | **Go** | Smallest binaries, fastest builds, easiest to deploy, large talent pool |
| ML-powered API | **Python/FastAPI** | Direct access to PyTorch, TensorFlow, HuggingFace |
| High-performance API for iOS app, team of 3+ Swift devs | **Swift (Vapor or Hummingbird)** | Code sharing + performance + team expertise |
| AWS Lambda function triggered by iOS app | **Swift (Hummingbird)** | Smallest cold start, first-class Lambda support, shared models |
| Enterprise backend, 20+ developer team | **Go or Java/Kotlin** | Hiring pool, mature tooling, battle-tested at scale |

## Vapor vs Hummingbird: Architectural Decision

### Market Context (2025-2026)

- **Vapor**: 78% of server-side Swift developers (2023 SSWG survey). SSWG Graduated.
- **Hummingbird**: 12% and growing. SSWG Incubating.
- **Key convergence**: Vapor 5's new HTTP server is based on Hummingbird's work (by Adam Fowler). The frameworks are sharing infrastructure at the networking layer.

### Weighted Decision Matrix

Present this to the architect. Each factor has a weight reflecting real-world impact:

| Factor | Weight | Vapor 4 | Hummingbird 2 |
|--------|--------|---------|----------------|
| **Time to first endpoint** | High | Faster — scaffolding, conventions, more docs | Slower — assemble your own stack |
| **Community & hiring** | High | Larger — more tutorials, books, Stack Overflow answers | Smaller — fewer resources, harder to find experience |
| **Binary size** | Low-Medium | ~50MB | ~15MB |
| **Startup time** | Low-Medium | ~200ms | ~50ms |
| **Compile time** | Medium | Slower — larger dependency graph | Faster — minimal dependencies |
| **Concurrency model** | Medium | async/await + EventLoopFuture legacy | Pure structured concurrency |
| **Architectural future** | Medium | Vapor 5 converging toward Hummingbird's model | Already on the modern concurrency stack |
| **ORM** | Medium | Fluent built-in | Bring your own (can use Fluent via hummingbird-fluent) |
| **Auth/WebSocket** | Medium | Built-in | Separate packages (hummingbird-auth, hummingbird-websocket) |
| **Lambda/serverless** | High (if applicable) | Via adapter | First-class support |
| **Long-term maintenance** | Medium | Larger community but heavier upgrade path (v4→v5 is major) | Lighter surface, fewer breaking changes |

### When to Choose Vapor

- Team is **new to server-side Swift** and needs a guided path
- Building a **full-featured web app** with auth, ORM, templates, WebSocket
- Need **rapid prototyping** — Vapor's conventions get you to a working API faster
- Hiring matters — easier to find developers with Vapor experience
- Building a **monolithic API** that will grow over time

### When to Choose Hummingbird

- Building **microservices or Lambda functions** where binary size and cold start matter
- Team is **experienced** and comfortable assembling components
- Want **pure structured concurrency** without EventLoopFuture legacy
- Building a **focused API** (e.g., a single-purpose service in a larger system)
- Compile time is a **real bottleneck** in your CI/CD pipeline

### When Either Works

- Standard REST API for an iOS app with PostgreSQL and JWT auth
- Team has 2-5 Swift developers comfortable with either
- No strong serverless or binary size requirements

In this case, **default to Vapor** for the larger community — unless the team explicitly prefers Hummingbird's architecture.

## SSWG Ecosystem: What's Available and What's Missing

### Graduated Packages (Production-Ready)

| Package | Purpose |
|---------|---------|
| SwiftNIO | Event-driven networking |
| SwiftLog / SwiftMetrics | Observability |
| PostgresNIO | PostgreSQL driver |
| AsyncHTTPClient | HTTP client |
| APNSwift | Push notifications |
| gRPC Swift | gRPC framework |
| Swift Crypto | Cryptography |
| Soto for AWS | AWS SDK |
| JWTKit | JSON Web Tokens |
| MultipartKit | Multipart parsing |
| Vapor | Web framework |

### Known Ecosystem Gaps

| Gap | Impact | Workaround |
|-----|--------|------------|
| **Redis** | RediStack unmaintained since 2023. No Swift 6, no clusters | Rewrite planned by SSWG for 2025 |
| **MySQL** | No graduated driver | Use vapor-community MySQL driver (works but less maintained) |
| **Kafka** | No mature Swift client | Use C library via Swift C interop |
| **Email** | No mature email processing library | Cultured Code uses Python Lambda for email |
| **Elasticsearch** | No official client | Community packages exist |
| **IDE tooling on Linux** | SourceKit-LSP has gaps | VS Code + LSP works but has papercuts |

### What This Means for Architecture

If your project requires libraries from the "Gaps" list as core dependencies, factor in the cost of:
- Writing Swift wrappers around C libraries
- Maintaining forks of community packages
- Using a second language (Python, Node) for specific services

This is acceptable in a microservices architecture where one service can use a different language. It's problematic in a monolithic architecture where you want one language throughout.

## Common Pitfalls to Present to Architects

### 1. Build Times
Swift's type checker is expensive. Large dependency graphs (Vapor) compound this. Plan for:
- 5-10 minute clean builds for medium projects
- Impact on CI/CD pipeline costs and developer iteration speed
- Mitigation: explicit type annotations, module splitting, caching in CI

### 2. Linux Parity
Code that compiles on macOS may fail on Linux. Foundation has divergent behavior between Darwin and swift-corelibs-foundation. **Test on Linux early and continuously** — not as an afterthought before deployment.

### 3. Crash Semantics
Swift's "crash early" philosophy (preconditionFailure, fatalError, force-unwraps) is appropriate for client apps but dangerous for servers. A single bad request hitting a force-unwrap takes down the entire process. There is no Go-style `recover()`. **Enforce `guard let` + error throwing discipline server-side.**

### 4. Rapid Language Evolution
Swift's concurrency model has evolved significantly across versions (5.5, 5.9, 6.0). Each shift requires framework updates. Vapor 4→5 is a major rewrite driven by Swift 6 changes. Budget for migration effort when planning long-term.

## How Claude Code Should Use This

1. **Before asking "which framework?"** — validate that server-side Swift itself is the right choice for the project
2. **Present trade-offs, don't decide** — show the comparison tables and let the architect choose
3. **Ask about context** — team size, existing infrastructure, hiring plans, performance requirements
4. **Be honest about gaps** — if the project needs Kafka or email processing, surface the ecosystem gap
5. **Default to Vapor when there's no strong signal** — larger community is a real advantage for most teams
6. **Never dismiss alternatives** — "Have you considered Go/Node for this?" is a valid question
