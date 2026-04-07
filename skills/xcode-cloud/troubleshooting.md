# Troubleshooting

Common Xcode Cloud errors, signing issues, SPM failures, timeouts, and debugging strategies.

## Most Common Errors

### 1. Code Signing Failures

**Error:** `No signing certificate "iOS Distribution" found`

**Causes:**
- Signing certificate not managed by Xcode Cloud
- Manual signing configured but profiles not available
- Team ID mismatch

**Fix:**
```
1. Open project in Xcode
2. Signing & Capabilities → Check "Automatically manage signing"
3. Select correct team
4. Push changes
5. Re-run workflow
```

**If using manual signing:**
```bash
# ci_post_clone.sh
# Ensure provisioning profiles are downloaded
echo "Checking signing configuration..."
echo "Team ID: $CI_TEAM_ID"
security find-identity -v -p codesigning 2>/dev/null || echo "No local certificates (expected in cloud)"
```

### 2. SPM Resolution Failures

**Error:** `Package resolution failed` or `Unable to resolve dependencies`

**Causes:**
- Private repository access not configured
- Package version conflicts
- Network timeout during resolution
- Package.resolved out of date

**Fix:**
```bash
# ci_post_clone.sh — Fix SPM issues

# 1. Clean package cache
rm -rf "$CI_PRIMARY_REPOSITORY_PATH/.build" 2>/dev/null || true

# 2. Re-resolve packages
cd "$CI_PRIMARY_REPOSITORY_PATH"
swift package resolve 2>&1 || {
    echo "Package resolution failed, trying clean resolve..."
    swift package reset
    swift package resolve
}
```

**For private packages:**
```bash
# ci_post_clone.sh — Configure access for private packages
# Option 1: Use Xcode Cloud's source control settings (recommended)
# Connect private repos in App Store Connect → Xcode Cloud → Source Control

# Option 2: Use environment secret
if [ -n "$GITHUB_PAT" ]; then
    git config --global url."https://x-access-token:${GITHUB_PAT}@github.com/".insteadOf "https://github.com/"
fi
```

### 3. Build Timeouts

**Error:** `Build exceeded maximum duration`

**Causes:**
- Compilation taking too long (complex type inference)
- Infinite loop in build script
- Network-dependent step hanging
- Large SPM dependency graph

**Fix:**
```bash
# ci_pre_xcodebuild.sh — Add timeouts to network operations
timeout 60 curl -s "https://api.example.com/config" > config.json || {
    echo "⚠️ Config fetch timed out, using defaults"
    echo '{}' > config.json
}
```

**Build settings optimization:**
```
Build Settings:
  SWIFT_COMPILATION_MODE = wholemodule  (for release)
  SWIFT_COMPILATION_MODE = incremental  (for debug/test)
  DEBUG_INFORMATION_FORMAT = dwarf       (not dwarf-with-dsym for test builds)
```

### 4. Test Failures in CI But Not Locally

**Common causes:**
- Timing-dependent tests (CI simulators are slower)
- Screen size differences
- Missing test data
- Environment-dependent code
- Locale/timezone differences

**Debugging:**
```bash
#!/bin/bash
# ci_pre_xcodebuild.sh — Log CI environment for debugging

if [[ "$CI_XCODEBUILD_ACTION" == "test"* ]]; then
    echo "=== CI Test Environment ==="
    echo "Xcode: $(xcodebuild -version)"
    echo "Swift: $(swift --version)"
    echo "macOS: $(sw_vers -productVersion)"
    echo "Available simulators:"
    xcrun simctl list devices available | head -20
    echo "Locale: $(defaults read NSGlobalDomain AppleLocale 2>/dev/null || echo 'default')"
    echo "Timezone: $(date +%Z)"
    echo "=========================="
fi
```

**Fix timing issues:**
```swift
// ❌ Bad — Hardcoded short timeouts
XCTAssertTrue(element.waitForExistence(timeout: 2))

// ✅ Good — CI-aware timeouts
let isCI = ProcessInfo.processInfo.environment["CI"] != nil
let timeout: TimeInterval = isCI ? 15 : 5
XCTAssertTrue(element.waitForExistence(timeout: timeout))
```

### 5. Archive Failures

**Error:** `Failed to create archive`

**Causes:**
- Missing entitlements
- Invalid bundle identifier
- Incorrect build settings for release
- Missing required capabilities

**Fix:**
```bash
# Verify build settings before archive
if [[ "$CI_XCODEBUILD_ACTION" == "archive" ]]; then
    echo "Checking archive requirements..."

    # Check for required files
    if [ ! -f "$CI_PRIMARY_REPOSITORY_PATH/MyApp/MyApp.entitlements" ]; then
        echo "⚠️ Entitlements file missing!"
    fi

    # Check scheme exists
    xcodebuild -list -project "$CI_XCODE_PROJECT" 2>/dev/null | grep "$CI_XCODE_SCHEME" || {
        echo "❌ Scheme '$CI_XCODE_SCHEME' not found!"
        xcodebuild -list -project "$CI_XCODE_PROJECT"
    }
fi
```

### 6. Script Permission Errors

**Error:** `Permission denied` when running ci_scripts

**Fix:**
```bash
# Check current permissions
git ls-files -s ci_scripts/

# Fix permissions
chmod +x ci_scripts/ci_post_clone.sh
chmod +x ci_scripts/ci_pre_xcodebuild.sh
chmod +x ci_scripts/ci_post_xcodebuild.sh

# Stage the permission change
git add ci_scripts/
git commit -m "Fix ci_scripts permissions"
```

**Verify in git:**
```bash
# Should show 100755 (executable)
git ls-tree HEAD ci_scripts/
# 100755 blob abc123 ci_scripts/ci_post_clone.sh
```

### 7. CocoaPods Issues

**Error:** `Unable to find a specification for...` or `Pod install failed`

**Fix:**
```bash
#!/bin/bash
# ci_post_clone.sh — CocoaPods setup

if [ -f "Podfile" ]; then
    echo "Installing CocoaPods..."

    # Ensure CocoaPods is installed
    gem install cocoapods 2>/dev/null || true

    # Update repo if needed
    pod repo update

    # Install pods
    pod install --repo-update

    echo "CocoaPods installation complete"
fi
```

### 8. Simulator Boot Failures

**Error:** `Failed to boot simulator` or `Unable to boot device`

**Fix:**
```bash
#!/bin/bash
# ci_pre_xcodebuild.sh — Ensure simulators are ready

if [[ "$CI_XCODEBUILD_ACTION" == "test"* ]]; then
    echo "Preparing simulators..."

    # List available simulators
    xcrun simctl list devices available

    # Boot default simulator if needed
    DEVICE_ID=$(xcrun simctl list devices available | grep "iPhone 15 Pro" | head -1 | \
        grep -oE '[A-F0-9-]{36}')

    if [ -n "$DEVICE_ID" ]; then
        xcrun simctl boot "$DEVICE_ID" 2>/dev/null || true
        echo "Booted simulator: $DEVICE_ID"
    fi
fi
```

## Debugging Strategies

### 1. Enable Verbose Logging

```bash
#!/bin/bash
# ci_pre_xcodebuild.sh
set -ex  # -e: fail on error, -x: print each command

echo "=== Debug Information ==="
env | sort  # Print all environment variables
echo "========================"
```

### 2. Artifact Collection

```bash
#!/bin/bash
# ci_post_xcodebuild.sh — Collect debug artifacts

ARTIFACTS_DIR="$CI_PRIMARY_REPOSITORY_PATH/ci_artifacts"
mkdir -p "$ARTIFACTS_DIR"

# Copy build logs
if [ -d "$CI_DERIVED_DATA_PATH/Logs" ]; then
    cp -r "$CI_DERIVED_DATA_PATH/Logs" "$ARTIFACTS_DIR/" 2>/dev/null || true
fi

# Copy test results
if [ -d "$CI_RESULT_BUNDLE_PATH" ]; then
    cp -r "$CI_RESULT_BUNDLE_PATH" "$ARTIFACTS_DIR/" 2>/dev/null || true
fi

echo "Artifacts collected in: $ARTIFACTS_DIR"
ls -la "$ARTIFACTS_DIR/"
```

### 3. Reproduce Locally

```bash
# Match CI environment locally
# 1. Check Xcode version
xcodebuild -version

# 2. Clean build (matches CI fresh build)
rm -rf ~/Library/Developer/Xcode/DerivedData/MyApp-*
xcodebuild clean build -scheme MyApp -destination 'generic/platform=iOS'

# 3. Run tests like CI
xcodebuild test \
    -scheme MyApp \
    -destination 'platform=iOS Simulator,name=iPhone 15 Pro,OS=18.0' \
    -resultBundlePath TestResults.xcresult
```

### 4. Check Build Logs in App Store Connect

1. App Store Connect → Xcode Cloud → Builds
2. Click failing build
3. View Logs → Download full build logs
4. Search for error messages

## Quick Reference: Error → Fix

| Error | Likely Cause | Quick Fix |
|-------|-------------|-----------|
| `No signing certificate` | Auto-sign disabled | Enable "Automatically manage signing" |
| `Package resolution failed` | Private repo not connected | Add repo in ASC → Source Control |
| `Build timeout` | Complex Swift types | Add explicit type annotations |
| `Permission denied (script)` | Script not executable | `git update-index --chmod=+x` |
| `Pod install failed` | CocoaPods not installed | Add `gem install cocoapods` to ci_post_clone |
| `Simulator not found` | Wrong destination | Update workflow destination |
| `Test timeout` | CI is slower | Increase timeouts for CI |
| `Archive failed` | Missing entitlements | Check entitlements file exists |
| `Module not found` | SPM cache issue | Clean `.build` in ci_post_clone |
| `Unable to boot device` | Simulator issue | Pre-boot in ci_pre_xcodebuild |

## When to Contact Apple

File a Feedback (feedbackassistant.apple.com) when:
- Infrastructure issues (builds stuck in "Preparing")
- Xcode Cloud dashboard not updating
- Certificate/provisioning issues with cloud-managed signing
- Build times significantly longer than expected
- Features documented but not working

## References

- [Xcode Cloud Troubleshooting](https://developer.apple.com/documentation/xcode/resolving-common-configuration-and-build-issues)
- [Apple Developer Forums](https://developer.apple.com/forums/tags/xcode-cloud)
- **scripts-reference.md** — Script patterns
- **optimization.md** — Build performance
- **workflow-patterns.md** — Configuration
