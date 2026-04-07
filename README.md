# Claude Code Skills: Server-Side Swift & Xcode Cloud

Production-ready [Claude Code](https://claude.ai/code) skills for **server-side Swift development** and **Xcode Cloud CI/CD**. These skills provide domain knowledge, code patterns, and workflows that Claude Code uses to assist you in building Swift backends and automating Apple's CI/CD pipeline.

## What Are Claude Code Skills?

Skills are markdown files that give Claude Code specialized knowledge about specific domains. When you ask Claude Code to help with Vapor routing or Xcode Cloud configuration, these skills provide the patterns, anti-patterns, and best practices it needs to give accurate, production-quality guidance.

## Skills Included

### 🖥️ Server-Side Swift (`skills/server-side-swift/`)

Everything you need to build backends and APIs in Swift — from project scaffolding to production deployment.

| File | What It Covers |
|------|---------------|
| `SKILL.md` | Entry point — activation triggers, project detection, architecture decision gate |
| `architecture-decisions.md` | **When to use server-side Swift** (and when not to), Vapor vs Hummingbird decision matrix, ecosystem gaps, comparison with Go/Node/Python, common pitfalls |
| `vapor-patterns.md` | Vapor 4 routing, controllers, middleware (CORS, rate limiting), DTOs, WebSocket |
| `hummingbird-patterns.md` | Hummingbird 2 patterns, RequestContext, Vapor vs Hummingbird decision tree |
| `database-patterns.md` | Fluent ORM models, migrations, queries, relationships, N+1 avoidance, transactions |
| `auth-patterns.md` | JWT authentication, Bearer tokens, Sign in with Apple server-side, token refresh |
| `deployment-patterns.md` | Multi-stage Docker, docker-compose, Fly.io, Railway, production checklist |
| `shared-code-patterns.md` | SharedModels package, client↔server DTOs, API endpoint protocols, APNSwift |
| `testing-patterns.md` | VaporTesting, HummingbirdTesting, Swift Testing, integration test patterns |

### ☁️ Xcode Cloud (`skills/xcode-cloud/`)

Complete CI/CD automation for Apple platforms — from PR validation to App Store submission.

| File | What It Covers |
|------|---------------|
| `SKILL.md` | Entry point — activation triggers, workflow components, environment variables |
| `workflow-patterns.md` | 5 workflow patterns (PR, TestFlight, App Store, Nightly, Multi-platform) |
| `scripts-reference.md` | Complete `ci_scripts/` reference, lifecycle hooks, environment variables |
| `testing-patterns.md` | Test plans, parallel testing, UI tests in CI, code coverage thresholds |
| `deployment-patterns.md` | TestFlight internal/external, App Store submission, notarization, phased releases |
| `troubleshooting.md` | 8 most common errors with fixes, signing, SPM, timeouts, quick reference |
| `optimization.md` | Build time reduction, compute hour management, caching, cost optimization |

## Installation

### Option 1: Clone to User Skills Directory (Global)

```bash
# Clone the repository
git clone https://github.com/melissa-pereira-deel/claude-code-server-side-swift-skills.git

# Copy skills to your Claude Code skills directory
cp -r claude-code-server-side-swift-skills/skills/* ~/.claude/skills/
```

Skills will be available globally across all your Claude Code sessions.

### Option 2: Clone to Project (Project-Scoped)

```bash
# From your project root
git clone https://github.com/melissa-pereira-deel/claude-code-server-side-swift-skills.git .claude-skills-temp

# Copy to project's .claude/skills/
mkdir -p .claude/skills
cp -r .claude-skills-temp/skills/* .claude/skills/
rm -rf .claude-skills-temp
```

Skills will be available only within that project.

### Option 3: Git Submodule

```bash
git submodule add https://github.com/melissa-pereira-deel/claude-code-server-side-swift-skills.git .claude/skills-external
```

## Usage Examples

Once installed, just ask Claude Code naturally:

**Server-Side Swift:**
- *"Create a REST API with Vapor and PostgreSQL"*
- *"Set up JWT authentication for my Swift backend"*
- *"Help me deploy my Vapor app to Fly.io"*
- *"Create shared models between my iOS app and server"*
- *"Should I use Vapor or Hummingbird for my microservice?"*

**Xcode Cloud:**
- *"Set up Xcode Cloud for my iOS app"*
- *"Create ci_scripts for my project"*
- *"My Xcode Cloud build is failing with signing errors"*
- *"Optimize my Xcode Cloud compute hours"*
- *"Configure TestFlight distribution on merge to main"*

## Frameworks & Tools Covered

### Server-Side Swift
- **Vapor 4** — Full-featured web framework with Fluent ORM
- **Hummingbird 2** — Lightweight, pure structured concurrency
- **Fluent** — ORM with PostgreSQL, SQLite, MySQL support
- **JWT** — Stateless authentication for mobile APIs
- **APNSwift** — Push notifications from server
- **Docker** — Multi-stage builds for production
- **Fly.io / Railway** — Cloud deployment platforms

### Xcode Cloud
- **Workflows** — Build, test, archive, distribute automation
- **ci_scripts/** — Post-clone, pre-build, post-build hooks
- **TestFlight** — Internal and external beta distribution
- **App Store Connect** — Automated submission
- **Test Plans** — Parallel testing, UI tests in CI
- **Code Signing** — Cloud-managed certificates

## Architecture-First Philosophy

These skills are designed with a core principle: **tool selection is an architectural decision, not an arbitrary AI choice.**

Before generating any code, the skills guide Claude Code to validate that server-side Swift is the right technology for your project. This means:

- **Presenting trade-offs honestly** — including when Go, Node.js, or Python might be a better fit
- **Surfacing ecosystem gaps** — like the lack of mature Swift libraries for Kafka, email processing, or Elasticsearch
- **Considering team context** — hiring pool, existing expertise, and long-term maintenance
- **Mapping scenarios to recommendations** — a solo indie dev has different needs than a 20-person enterprise team

The `architecture-decisions.md` file includes real-world data from Apple's Password Monitoring Service migration (Java → Swift: 40% throughput increase, 50% memory reduction) and Cultured Code's Things Cloud migration (Python → Swift: 4x faster, 3x cheaper), along with honest assessments of when server-side Swift is *not* the right choice.

The goal: your coding agent should inform architectural decisions, not make them for you.

## Compatibility

- **Swift**: 5.9+ (Swift 6 concurrency patterns included)
- **Vapor**: 4.89+
- **Hummingbird**: 2.0+
- **Xcode**: 15+ (Xcode Cloud)
- **Claude Code**: Any version with skills support

## Contributing

Contributions are welcome! If you have improvements, additional patterns, or new reference materials:

1. Fork this repository
2. Create a feature branch (`git checkout -b feature/vapor-websocket-patterns`)
3. Follow the existing file structure and formatting conventions
4. Ensure all code examples are syntactically correct Swift
5. Submit a Pull Request

### Style Guidelines

- Use `kebab-case` for file names
- Include both ✅ good and ❌ bad patterns with explanations
- All Swift code must compile (use realistic, complete examples)
- Keep SKILL.md files under 300 lines; reference files under 600 lines
- Cross-reference related files in the References section

## License

MIT License — see [LICENSE](LICENSE) for details.

## Acknowledgments

- [Vapor](https://vapor.codes/) — The most popular server-side Swift framework
- [Hummingbird](https://github.com/hummingbird-project/hummingbird) — Lightweight Swift HTTP server
- [Swift on Server](https://www.swift.org/documentation/server/) — Apple's official resources
- [Claude Code](https://claude.ai/code) — AI-powered development assistant
