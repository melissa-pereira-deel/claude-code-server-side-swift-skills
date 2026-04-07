# Workflow Patterns

Xcode Cloud workflow configuration, start conditions, build actions, post-actions, and multi-platform workflows.

## Workflow Architecture

### Anatomy of a Workflow

```
Workflow
├── Name & Description
├── Repository & Branch
├── Start Conditions
│   ├── Branch Changes
│   ├── Pull Request Changes
│   ├── Tag Changes
│   └── Scheduled
├── Environment
│   ├── Xcode Version
│   └── macOS Version
├── Build Actions
│   ├── Build
│   ├── Test
│   ├── Analyze
│   └── Archive
└── Post-Actions
    ├── TestFlight (Internal/External)
    ├── App Store Submission
    └── Notify (Slack, Email)
```

## Common Workflow Patterns

### Pattern 1: PR Validation

**Purpose:** Validate every pull request before merge.

**Configuration:**
- **Start Condition:** Pull Request → Target branch: `main`
- **Environment:** Latest stable Xcode
- **Actions:** Build + Test
- **Post-Actions:** None (just validate)

**Script support:**
```bash
#!/bin/bash
# ci_pre_xcodebuild.sh
# Skip expensive steps for PR builds
if [ -n "$CI_PULL_REQUEST_NUMBER" ]; then
    echo "PR build — skipping non-essential steps"
fi
```

### Pattern 2: TestFlight on Merge

**Purpose:** Distribute to testers when code merges to main.

**Configuration:**
- **Start Condition:** Branch Changes → Branch: `main`
- **Environment:** Latest stable Xcode
- **Actions:** Archive
- **Post-Actions:** TestFlight Internal Testing

**Script support:**
```bash
#!/bin/bash
# ci_post_clone.sh
# Increment build number based on CI_BUILD_NUMBER
if [ -n "$CI_BUILD_NUMBER" ]; then
    echo "Setting build number to $CI_BUILD_NUMBER"
    # Build number is auto-managed by Xcode Cloud
fi
```

### Pattern 3: App Store Release on Tag

**Purpose:** Submit to App Store when a version tag is created.

**Configuration:**
- **Start Condition:** Tag Changes → Pattern: `v*` (e.g., v1.0.0, v2.1.0)
- **Environment:** Latest stable Xcode
- **Actions:** Archive
- **Post-Actions:** App Store Connect submission

**Script support:**
```bash
#!/bin/bash
# ci_pre_xcodebuild.sh
# Extract version from tag
if [ -n "$CI_TAG" ]; then
    VERSION="${CI_TAG#v}"  # Remove 'v' prefix
    echo "Building version: $VERSION"
fi
```

### Pattern 4: Nightly Build

**Purpose:** Run comprehensive tests on a schedule.

**Configuration:**
- **Start Condition:** Schedule → Daily at 2:00 AM
- **Environment:** Latest stable Xcode
- **Actions:** Build + Full Test Suite (Unit + UI + Performance)
- **Post-Actions:** Notify on failure

### Pattern 5: Multi-Platform Build

**Purpose:** Build and test across iOS, macOS, watchOS.

**Configuration:**
- Create separate workflows per platform
- Or use a single workflow with multiple destinations

```bash
#!/bin/bash
# ci_pre_xcodebuild.sh
# Platform-specific setup
case "$CI_XCODE_SCHEME" in
    "MyApp iOS")
        echo "iOS build setup"
        ;;
    "MyApp macOS")
        echo "macOS build setup"
        ;;
    "MyApp watchOS")
        echo "watchOS build setup"
        ;;
esac
```

## Start Conditions

### Branch Changes

```
Trigger: Push to specified branches
Branches:
  - main          # Production releases
  - develop       # Development builds
  - release/*     # Release candidates
  - feature/*     # Feature branch builds (optional)
```

**Auto-cancel:** Enable "Cancel running builds for the same branch" to save compute hours.

### Pull Request Changes

```
Trigger: PR opened, updated, or reopened
Target Branch: main (or develop)
Source Branch: Any (or pattern like feature/*)
```

### Tag Changes

```
Trigger: Tag creation matching pattern
Pattern: v*        # Matches v1.0.0, v2.1.3-beta
  or:    release/* # Matches release/1.0.0
```

### Scheduled Builds

```
Trigger: Cron-like schedule
Options:
  - Daily at specific time
  - Weekly on specific day
  - Custom interval
Branch: main (or develop)
```

## Environment Configuration

### Xcode Version Selection

```
Options:
  - Latest Release          # Auto-updates (⚠️ may break builds)
  - Latest Stable           # Recommended for production
  - Specific Version        # Pin to Xcode 15.4, 16.0, etc.
```

**Best Practice:** Pin Xcode version for release workflows, use "Latest" for nightly testing.

### macOS Version

```
Options:
  - Latest Release
  - Specific Version (macOS 14 Sonoma, macOS 15 Sequoia)
```

## Build Actions

### Build

```
Action: Build
Scheme: MyApp
Platform: iOS (iPhone, iPad), macOS, watchOS
Configuration: Debug (for testing) or Release (for archive)
```

### Test

```
Action: Test
Scheme: MyApp
Platform: iOS Simulator (specific device and OS)
Test Plan: MyApp-Tests.xctestplan (optional)
Code Coverage: Enable/Disable
```

### Analyze

```
Action: Analyze
Scheme: MyApp
Purpose: Static analysis for potential bugs
```

### Archive

```
Action: Archive
Scheme: MyApp
Platform: iOS (Device), macOS
Configuration: Release
Distribution: App Store, TestFlight, or Export
```

## Post-Actions

### TestFlight Internal

```
Action: TestFlight Internal Testing
Group: Internal Testers (up to 100 testers)
Notes: Auto-generated from commit messages
No App Review required
```

### TestFlight External

```
Action: TestFlight External Testing
Group: Beta Testers (up to 10,000 testers)
Notes: Custom release notes
Requires Beta App Review
```

### App Store Submission

```
Action: Submit to App Store
Requires manual release or phased release configuration
```

### Notification

```
Action: Notify
Channels:
  - Email (team members)
  - Slack webhook (via ci_post_xcodebuild.sh)
Conditions: On failure, on success, always
```

## Workflow Organization

### Recommended Workflow Set

| Workflow | Trigger | Actions | Post-Actions |
|----------|---------|---------|-------------|
| PR Validation | Pull Request → main | Build + Test | None |
| Dev Build | Push → develop | Build + Test + Archive | TestFlight Internal |
| Release Candidate | Push → release/* | Build + Full Tests + Archive | TestFlight External |
| Production Release | Tag → v* | Archive | App Store Submission |
| Nightly Tests | Schedule (daily) | Build + Full Test Suite | Notify on failure |

### Branch Strategy Alignment

```
main ←── release/1.0 ←── develop ←── feature/xyz
  │           │              │
  │           │              └─ PR Validation workflow
  │           └─ Release Candidate workflow
  └─ Production Release workflow (via tag)
```

## Multi-Workflow Coordination

### Passing Data Between Scripts

```bash
#!/bin/bash
# ci_post_clone.sh — Write shared data
echo "CUSTOM_VAR=value" > "$CI_DERIVED_DATA_PATH/custom_env.sh"

# ci_pre_xcodebuild.sh — Read shared data
source "$CI_DERIVED_DATA_PATH/custom_env.sh"
echo "Using: $CUSTOM_VAR"
```

### Conditional Workflow Steps

```bash
#!/bin/bash
# ci_pre_xcodebuild.sh
# Different behavior based on action
case "$CI_XCODEBUILD_ACTION" in
    "build-for-testing"|"test-without-building"|"test")
        echo "Test build — minimal setup"
        ;;
    "archive")
        echo "Archive build — full setup"
        # Generate release notes, check version, etc.
        ;;
    *)
        echo "Build action: $CI_XCODEBUILD_ACTION"
        ;;
esac
```

## Anti-Patterns

### ❌ Using "Latest Xcode" for Release Workflows

```
// ❌ Bad — Xcode updates can break your build
Environment: Latest Release

// ✅ Good — Pin version for release stability
Environment: Xcode 16.0
```

### ❌ Running Full Test Suite on Every PR

```
// ❌ Bad — Wastes compute hours
PR Workflow: Build + Unit + UI + Performance Tests

// ✅ Good — Fast feedback on PRs, full suite nightly
PR Workflow: Build + Unit Tests only
Nightly Workflow: Build + All Tests
```

### ❌ No Auto-Cancel for Same Branch

```
// ❌ Bad — Multiple builds queue up for force-pushes
Auto-cancel: Disabled

// ✅ Good — Cancel superseded builds
Auto-cancel: Enabled (cancel running builds when new push arrives)
```

## References

- [Xcode Cloud Workflow Reference](https://developer.apple.com/documentation/xcode/xcode-cloud)
- [Configuring Start Conditions](https://developer.apple.com/documentation/xcode/configuring-start-conditions)
- **scripts-reference.md** — Custom script patterns
- **testing-patterns.md** — Test configuration
- **deployment-patterns.md** — Distribution setup
