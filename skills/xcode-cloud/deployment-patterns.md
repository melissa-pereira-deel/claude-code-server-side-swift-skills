# Deployment Patterns

TestFlight distribution, App Store submission, notarization for macOS, and multi-platform deployment via Xcode Cloud.

## TestFlight Distribution

### Internal Testing

**Configuration:**
- Maximum 100 testers (Apple Developer Program members)
- No Beta App Review required
- Available immediately after build processes

**Setup in Xcode Cloud:**
1. Workflow → Post-Actions → TestFlight Internal Testing
2. Select internal testing group
3. Build is available within minutes of successful archive

**Who can be an internal tester:**
- Team members with App Store Connect roles
- Must accept email invitation
- Must have TestFlight app installed

### External Testing

**Configuration:**
- Up to 10,000 testers per app
- Requires Beta App Review (usually 24-48 hours)
- Need public link or email invitation

**Setup in Xcode Cloud:**
1. Workflow → Post-Actions → TestFlight External Testing
2. Select external testing group
3. Provide "What to Test" notes
4. Build is submitted for Beta App Review

### Release Notes Automation

```bash
#!/bin/bash
# ci_post_xcodebuild.sh — Generate release notes

if [[ "$CI_XCODEBUILD_ACTION" == "archive" ]]; then
    # Generate notes from recent commits
    NOTES=$(git log --oneline -20 --no-merges 2>/dev/null || echo "New build available")

    # Write to file that TestFlight can use
    NOTES_FILE="$CI_PRIMARY_REPOSITORY_PATH/WhatToTest.en-US.txt"
    echo "Build #$CI_BUILD_NUMBER" > "$NOTES_FILE"
    echo "" >> "$NOTES_FILE"
    echo "Changes:" >> "$NOTES_FILE"
    echo "$NOTES" >> "$NOTES_FILE"
    echo "" >> "$NOTES_FILE"
    echo "Branch: $CI_BRANCH" >> "$NOTES_FILE"
    echo "Commit: ${CI_COMMIT:0:7}" >> "$NOTES_FILE"
fi
```

### TestFlight Beta Groups Strategy

| Group | Purpose | Audience | Frequency |
|-------|---------|----------|-----------|
| Internal QA | Daily builds from develop | QA team (5-10 people) | Every merge to develop |
| Internal Team | Feature validation | Full team (20-50 people) | Weekly from release branch |
| External Beta | User feedback | Beta testers (100-1000) | Release candidates only |
| Public Beta | Wide testing | Anyone with link | Major versions only |

## App Store Submission

### Automatic Submission via Xcode Cloud

**Configuration:**
1. Workflow → Post-Actions → Submit to App Store Connect
2. Trigger: Tag creation matching `v*`
3. Requires all App Store metadata to be ready in App Store Connect

### Pre-Submission Checklist

```bash
#!/bin/bash
# ci_pre_xcodebuild.sh — Pre-archive validation

if [[ "$CI_XCODEBUILD_ACTION" == "archive" ]]; then
    echo "=== Pre-Archive Validation ==="

    # 1. Verify version number
    if [ -n "$CI_TAG" ]; then
        EXPECTED_VERSION="${CI_TAG#v}"
        echo "Expected version: $EXPECTED_VERSION"

        # Verify Info.plist matches tag
        PLIST_VERSION=$(/usr/libexec/PlistBuddy -c "Print :CFBundleShortVersionString" \
            "$CI_PRIMARY_REPOSITORY_PATH/MyApp/Info.plist" 2>/dev/null || echo "unknown")

        if [ "$PLIST_VERSION" != "$EXPECTED_VERSION" ]; then
            echo "⚠️ Version mismatch: plist=$PLIST_VERSION, tag=$EXPECTED_VERSION"
            # Auto-fix
            /usr/libexec/PlistBuddy -c "Set :CFBundleShortVersionString $EXPECTED_VERSION" \
                "$CI_PRIMARY_REPOSITORY_PATH/MyApp/Info.plist"
            echo "✅ Fixed version to $EXPECTED_VERSION"
        fi
    fi

    # 2. Verify build number
    echo "Build number: $CI_BUILD_NUMBER"

    # 3. Check for debug code
    DEBUG_COUNT=$(grep -r "DEBUG_ONLY\|#if DEBUG" "$CI_PRIMARY_REPOSITORY_PATH/Sources/" \
        --include="*.swift" -l 2>/dev/null | wc -l | tr -d ' ')
    echo "Files with debug conditionals: $DEBUG_COUNT"
fi
```

### Version and Build Number Management

```bash
#!/bin/bash
# ci_pre_xcodebuild.sh

# Xcode Cloud auto-manages build number via CI_BUILD_NUMBER
# For version number, extract from tag:

if [ -n "$CI_TAG" ]; then
    VERSION="${CI_TAG#v}"  # v1.2.3 → 1.2.3

    # Update all targets
    for plist in $(find "$CI_PRIMARY_REPOSITORY_PATH" -name "Info.plist" \
        -not -path "*/Pods/*" -not -path "*/DerivedData/*"); do
        /usr/libexec/PlistBuddy -c "Set :CFBundleShortVersionString $VERSION" "$plist" 2>/dev/null || true
    done

    echo "Set version to $VERSION across all targets"
fi
```

## Code Signing

### Xcode Cloud Managed Signing (Recommended)

Xcode Cloud uses **cloud-managed certificates and profiles**:

1. **Automatic:** Xcode Cloud creates and manages signing certificates
2. **No manual profiles:** Provisioning is handled automatically
3. **No certificates to install:** Everything is cloud-managed

**Setup:**
- Project → Signing & Capabilities → Automatically manage signing ✓
- Team: Select your Apple Developer Team
- Xcode Cloud handles the rest

### Manual Signing (When Needed)

For apps requiring specific entitlements or custom provisioning:

1. Create profiles in Apple Developer Portal
2. Reference them in your Xcode project
3. Xcode Cloud will use the project's signing settings

```bash
#!/bin/bash
# ci_post_clone.sh — Manual signing setup (rare)

# If using manual signing, ensure provisioning profiles are available
if [ -d "$HOME/Library/MobileDevice/Provisioning Profiles" ]; then
    echo "Provisioning profiles found"
    ls "$HOME/Library/MobileDevice/Provisioning Profiles/"
fi
```

## macOS Notarization

### Automatic Notarization

For macOS apps distributed outside the App Store, Xcode Cloud handles notarization:

**Configuration:**
1. Archive action → Distribution: Developer ID
2. Xcode Cloud automatically submits to Apple's notary service
3. Staples the notarization ticket to the app

### Notarization in Scripts

```bash
#!/bin/bash
# ci_post_xcodebuild.sh — Verify notarization

if [[ "$CI_XCODEBUILD_ACTION" == "archive" ]] && [[ "$CI_PRODUCT_PLATFORM" == "macOS" ]]; then
    echo "Verifying notarization..."
    # Xcode Cloud handles this automatically for Developer ID distribution
    # But you can verify:
    if [ -d "$CI_ARCHIVE_PATH" ]; then
        echo "Archive ready for distribution: $CI_ARCHIVE_PATH"
    fi
fi
```

## Multi-Platform Deployment

### Platform-Specific Workflows

| Platform | Workflow | Distribution |
|----------|---------|-------------|
| iOS | PR + Merge + Tag | TestFlight → App Store |
| macOS | PR + Merge + Tag | TestFlight → App Store or Developer ID |
| watchOS | Bundled with iOS | Included in iOS submission |
| tvOS | Separate workflow | TestFlight → App Store |
| visionOS | Separate workflow | TestFlight → App Store |

### Universal Purchase Support

```bash
#!/bin/bash
# ci_pre_xcodebuild.sh — Multi-platform archive

echo "Platform: $CI_PRODUCT_PLATFORM"

case "$CI_PRODUCT_PLATFORM" in
    "iOS")
        echo "iOS archive — includes watchOS companion"
        ;;
    "macOS")
        echo "macOS archive — Catalyst or native"
        ;;
    *)
        echo "Other platform: $CI_PRODUCT_PLATFORM"
        ;;
esac
```

## Phased Releases

### Configuring Phased Release

After App Store submission via Xcode Cloud:
1. App Store Connect → App → Version → Release
2. Select "Release this version over a 7-day period"
3. Monitor crash rates and user feedback

```
Day 1: 1% of users
Day 2: 2% of users
Day 3: 5% of users
Day 4: 10% of users
Day 5: 20% of users
Day 6: 50% of users
Day 7: 100% of users
```

You can pause/resume/halt the rollout at any time.

## Post-Deployment Notifications

```bash
#!/bin/bash
# ci_post_xcodebuild.sh — Notify on successful archive

if [[ "$CI_XCODEBUILD_ACTION" == "archive" ]] && [ "${CI_XCODEBUILD_EXIT_CODE:-1}" -eq 0 ]; then
    # Slack notification
    if [ -n "$SLACK_WEBHOOK_URL" ]; then
        VERSION="${CI_TAG:-$CI_BRANCH}"
        curl -s -X POST "$SLACK_WEBHOOK_URL" \
            -H "Content-Type: application/json" \
            -d "{
                \"text\": \"🚀 *$CI_PRODUCT* build #$CI_BUILD_NUMBER ($VERSION) archived successfully and submitted to TestFlight/App Store.\",
                \"unfurl_links\": false
            }"
    fi

    # Discord notification
    if [ -n "$DISCORD_WEBHOOK_URL" ]; then
        curl -s -X POST "$DISCORD_WEBHOOK_URL" \
            -H "Content-Type: application/json" \
            -d "{
                \"content\": \"🚀 **$CI_PRODUCT** build #$CI_BUILD_NUMBER archived and submitted.\"
            }"
    fi
fi
```

## Anti-Patterns

### ❌ Submitting to External TestFlight on Every Merge

```
// ❌ Bad — Every merge triggers Beta App Review (24-48h delays)
Merge to main → External TestFlight

// ✅ Good — Internal for daily, external for release candidates
Merge to main → Internal TestFlight
Tag v* → External TestFlight
```

### ❌ No Version Validation Before Archive

```bash
# ❌ Bad — Archive with wrong version number
# No version check before archiving

# ✅ Good — Validate version matches tag
if [ -n "$CI_TAG" ]; then
    EXPECTED="${CI_TAG#v}"
    ACTUAL=$(/usr/libexec/PlistBuddy -c "Print :CFBundleShortVersionString" Info.plist)
    if [ "$EXPECTED" != "$ACTUAL" ]; then
        echo "ERROR: Version mismatch!" && exit 1
    fi
fi
```

### ❌ No Phased Release for Major Updates

```
// ❌ Bad — Ship to 100% of users immediately
Release: Full rollout

// ✅ Good — Gradual rollout to catch issues
Release: Phased (7-day rollout)
```

## References

- [TestFlight Documentation](https://developer.apple.com/testflight/)
- [App Store Submission](https://developer.apple.com/documentation/xcode/distributing-your-app-for-beta-testing-and-releases)
- [Notarization](https://developer.apple.com/documentation/security/notarizing_macos_software_before_distribution)
- **workflow-patterns.md** — Workflow triggers for deployment
- **scripts-reference.md** — Script patterns for deployment
