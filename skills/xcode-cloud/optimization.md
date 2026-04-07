# Optimization

Build time reduction, compute hour management, caching strategies, and cost optimization for Xcode Cloud.

## Understanding Compute Hours

### How Hours Are Counted

- Clock time from build start to finish (not CPU time)
- Includes: clone, scripts, build, test, archive, signing
- Each workflow run consumes hours regardless of success/failure
- Cancelled builds still consume hours up to cancellation point

### Tier Planning

| Monthly Usage | Recommended Tier | Solo Dev | Small Team | Large Team |
|---------------|-----------------|----------|------------|------------|
| < 25 hours | Free | ✅ | ❌ | ❌ |
| 25-100 hours | Tier 1 | ✅ | ✅ | ❌ |
| 100-250 hours | Tier 2 | ❌ | ✅ | ✅ |
| 250+ hours | Tier 3+ | ❌ | ❌ | ✅ |

### Estimate Your Usage

```
Per PR: ~5-15 min (build + unit tests)
Per merge to main: ~10-20 min (build + archive + TestFlight)
Per release: ~15-30 min (full test + archive + App Store)
Nightly: ~20-40 min (full test suite)

Example solo dev:
  10 PRs/week × 10 min = 1.7 hours/week
  10 merges/week × 15 min = 2.5 hours/week
  1 nightly × 30 min × 5 = 2.5 hours/week
  Total: ~6.7 hours/week = ~27 hours/month (Tier 1)
```

## Build Time Optimization

### 1. Reduce Compilation Time

```swift
// ❌ Bad — Complex type inference slows compilation
let result = items.filter { $0.isActive && $0.date > Date() }
    .sorted { $0.priority > $1.priority }
    .map { ($0.name, $0.value * multiplier + offset) }
    .reduce(into: [:]) { $0[$1.0, default: 0] += $1.1 }

// ✅ Good — Explicit types help the compiler
let activeItems: [Item] = items.filter { $0.isActive && $0.date > Date() }
let sorted: [Item] = activeItems.sorted { $0.priority > $1.priority }
let mapped: [(String, Double)] = sorted.map { item -> (String, Double) in
    (item.name, item.value * multiplier + offset)
}
let result: [String: Double] = mapped.reduce(into: [:]) { dict, pair in
    dict[pair.0, default: 0] += pair.1
}
```

### 2. Minimize Dependencies

```swift
// Package.swift — Only include what you actually use
dependencies: [
    // ✅ Pin to specific versions (faster resolution)
    .package(url: "https://github.com/vapor/vapor.git", exact: "4.89.3"),

    // ❌ Avoid broad ranges (slower resolution)
    // .package(url: "...", from: "4.0.0"),
],
```

### 3. Optimize Build Settings

```
// Xcode Build Settings
SWIFT_COMPILATION_MODE:
  Debug: incremental    (faster builds)
  Release: wholemodule  (faster binary)

DEBUG_INFORMATION_FORMAT:
  Test builds: dwarf         (faster, no dSYM)
  Archive: dwarf-with-dsym   (needed for crash reports)

ENABLE_TESTABILITY:
  Test builds: YES
  Archive: NO (smaller binary)
```

### 4. Reduce Script Execution Time

```bash
#!/bin/bash
# ci_post_clone.sh — Optimized dependency installation

# ✅ Only install what's needed for this action
case "$CI_XCODEBUILD_ACTION" in
    "test"|"build-for-testing")
        # Minimal tools for testing
        brew install swiftlint 2>/dev/null || true
        ;;
    "archive")
        # Additional tools only for archive
        brew install swiftlint imagemagick 2>/dev/null || true
        ;;
    *)
        # Build only — minimal setup
        ;;
esac

# ✅ Skip unnecessary steps
if [ -n "$CI_PULL_REQUEST_NUMBER" ]; then
    echo "PR build — skipping optional tools"
    exit 0
fi
```

## Workflow Optimization

### 1. Auto-Cancel Superseded Builds

Enable "Cancel running builds" for branch and PR triggers:
- Saves hours when developers push frequently
- Only the latest commit's build runs to completion

### 2. Smart Workflow Triggers

```
// ❌ Bad — Every push to any branch triggers full suite
Trigger: Any branch, all tests

// ✅ Good — Appropriate workflows per trigger
PR: Unit tests only (5 min)
Main: Unit tests + archive + TestFlight (15 min)
Tag: Full tests + archive + App Store (25 min)
Nightly: Full test suite (30 min)
```

### 3. Branch Filtering

```
Start Conditions:
  Branch Changes:
    Include: main, develop, release/*
    Exclude: docs/*, ci/test-*

  Pull Request:
    Target: main, develop
    Source: Any
```

### 4. File-Based Triggers (Planned)

```
// Future: Only build when relevant files change
// Currently not supported natively in Xcode Cloud
// Workaround: Check in scripts

#!/bin/bash
# ci_pre_xcodebuild.sh — Skip build for docs-only changes
CHANGED_FILES=$(git diff --name-only HEAD~1 2>/dev/null || echo "")

if echo "$CHANGED_FILES" | grep -qvE '\.(md|txt|yml)$'; then
    echo "Source files changed — proceeding with build"
else
    echo "Only documentation changed — could skip build"
    # Note: Can't actually skip the build action from a script
    # but you can skip expensive steps
fi
```

## Test Optimization

### 1. Test Plan Strategy

```
PR-Tests.xctestplan:
  - Unit tests only
  - Parallel execution: ON
  - Expected duration: 5 min

Nightly-Tests.xctestplan:
  - All tests (unit + UI + performance)
  - Parallel execution: ON
  - Expected duration: 25 min
```

### 2. Skip Slow Tests for PRs

```swift
import Testing

@Test("Large data import", .tags(.slow), .disabled("Skip in PR builds"))
func testLargeImport() async throws {
    // Only runs in nightly test plan
}

// Or in XCTest:
func testLargeImport() throws {
    try XCTSkipIf(
        ProcessInfo.processInfo.environment["CI_WORKFLOW"]?.contains("PR") == true,
        "Skipping slow test in PR builds"
    )
    // ... slow test
}
```

### 3. Test Splitting Across Workflows

```
Workflow 1: Core Tests (iPhone)
  → Tests tagged with .core
  → Duration: ~8 min

Workflow 2: UI Tests (iPhone + iPad)
  → UI test targets only
  → Duration: ~12 min
  → Trigger: Merge to main only
```

## Caching Strategies

### SPM Package Cache

Xcode Cloud caches SPM packages between builds automatically. To optimize:

```swift
// Package.swift — Use exact versions for faster resolution
.package(url: "https://github.com/vapor/vapor.git", exact: "4.89.3"),
```

### Homebrew Cache

```bash
#!/bin/bash
# ci_post_clone.sh — Use Homebrew efficiently

# Check if tool already installed (from cache)
if ! command -v swiftlint &> /dev/null; then
    echo "Installing SwiftLint..."
    brew install swiftlint
else
    echo "SwiftLint already available (cached)"
fi
```

### Derived Data

Xcode Cloud manages Derived Data automatically:
- Clean builds for archive actions
- Incremental builds may be used for test actions
- No manual control over caching

## Cost Monitoring

### Track Usage

```bash
#!/bin/bash
# ci_post_xcodebuild.sh — Log build duration for tracking

START_TIME="${CI_BUILD_START_TIME:-unknown}"
echo "=== Build Metrics ==="
echo "Workflow: $CI_WORKFLOW"
echo "Action: $CI_XCODEBUILD_ACTION"
echo "Branch: $CI_BRANCH"
echo "Build Number: $CI_BUILD_NUMBER"
echo "Exit Code: ${CI_XCODEBUILD_EXIT_CODE:-unknown}"
echo "===================="

# Send to analytics (optional)
if [ -n "$ANALYTICS_ENDPOINT" ]; then
    curl -s -X POST "$ANALYTICS_ENDPOINT/build-metrics" \
        -H "Content-Type: application/json" \
        -d "{
            \"workflow\": \"$CI_WORKFLOW\",
            \"action\": \"$CI_XCODEBUILD_ACTION\",
            \"branch\": \"$CI_BRANCH\",
            \"build_number\": $CI_BUILD_NUMBER,
            \"exit_code\": ${CI_XCODEBUILD_EXIT_CODE:-1}
        }" 2>/dev/null || true
fi
```

### Monthly Budget Alerts

Monitor in App Store Connect:
- App Store Connect → Xcode Cloud → Usage
- Set up alerts when approaching tier limits
- Review which workflows consume the most hours

## Optimization Checklist

### Quick Wins (< 1 hour to implement)

- [ ] Enable auto-cancel for same branch
- [ ] PR builds: unit tests only (no UI tests)
- [ ] Pin dependency versions in Package.swift
- [ ] Remove unused dependencies
- [ ] Skip optional tools in PR build scripts

### Medium Effort (1-4 hours)

- [ ] Create separate test plans for PR vs nightly
- [ ] Add explicit type annotations to complex expressions
- [ ] Optimize ci_post_clone.sh (conditional installation)
- [ ] Set up branch filtering (exclude docs branches)
- [ ] Reduce archive frequency (only on tag/merge)

### Advanced (4+ hours)

- [ ] Profile build with `-Xfrontend -warn-long-function-bodies=100`
- [ ] Split large targets into modules
- [ ] Implement test tagging for selective execution
- [ ] Set up build metrics tracking
- [ ] Configure per-platform workflows

## Comparison: Xcode Cloud vs Alternatives

| Feature | Xcode Cloud | GitHub Actions | Bitrise |
|---------|------------|----------------|---------|
| Setup | Xcode UI | YAML files | Web UI + YAML |
| Apple Silicon | ✅ Native | ❌ (x86 runners) | ✅ (M1 stacks) |
| Signing | Cloud-managed | Manual | Manual |
| TestFlight | Native integration | fastlane | fastlane |
| Free Tier | 25 hours/month | 2,000 min/month | 50 min/month |
| macOS Build | ✅ Included | $0.08/min | ✅ Included |
| Custom Hardware | ❌ | Self-hosted | ❌ |
| YAML Config | ❌ (UI only) | ✅ | ✅ |

## References

- [Xcode Cloud Pricing](https://developer.apple.com/xcode-cloud/)
- [Build Performance](https://developer.apple.com/documentation/xcode/improving-the-speed-of-incremental-builds)
- **workflow-patterns.md** — Efficient workflow design
- **scripts-reference.md** — Optimized scripts
- **troubleshooting.md** — Fix slow builds
