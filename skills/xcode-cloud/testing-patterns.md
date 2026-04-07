# Testing Patterns

Test plans, parallel testing, UI tests, code coverage, and test optimization in Xcode Cloud.

## Test Plans

### What Are Test Plans?

Test plans (`.xctestplan`) let you define which tests run, with what configuration, and in what order. They are the recommended way to organize tests in Xcode Cloud.

### Creating a Test Plan

1. Xcode → Product → Test Plan → New Test Plan
2. Name it (e.g., `MyApp-CI.xctestplan`)
3. Add test targets
4. Configure test settings

### Test Plan Configurations

```json
{
    "configurations": [
        {
            "name": "Unit Tests",
            "options": {
                "targetForVariableExpansion": "MyApp"
            }
        },
        {
            "name": "UI Tests - iPhone",
            "options": {
                "targetForVariableExpansion": "MyApp",
                "environmentVariableEntries": [
                    { "key": "UI_TEST_DEVICE", "value": "iPhone" }
                ]
            }
        }
    ]
}
```

### Multiple Test Plans for Different Workflows

| Test Plan | Tests Included | Used In |
|-----------|---------------|---------|
| `PR-Tests.xctestplan` | Unit tests only | PR Validation workflow |
| `Full-Tests.xctestplan` | Unit + UI + Performance | Nightly workflow |
| `Smoke-Tests.xctestplan` | Critical path UI tests | Release workflow |

### Configuring Test Plan in Workflow

In Xcode Cloud workflow settings:
- Build Action → Test
- Test Plan: Select your `.xctestplan` file

## Parallel Testing

### Xcode Cloud Parallel Execution

Xcode Cloud automatically runs tests in parallel across multiple simulators when configured:

```
Workflow → Test Action → Test Plan
  → Enable "Execute tests in parallel" (default: on)
  → Simulator count: Auto (or specify)
```

### Optimizing for Parallel Execution

```swift
// ✅ Tests should be independent — no shared mutable state
@Suite("Todo Tests", .serialized)  // Use .serialized only when tests MUST run sequentially
struct TodoTests {
    @Test func createTodo() async throws { ... }
    @Test func deleteTodo() async throws { ... }
}

// ✅ Good — Independent tests (default parallel)
@Suite("API Tests")
struct APITests {
    @Test func fetchUsers() async throws { ... }
    @Test func fetchTodos() async throws { ... }
}
```

### Test Splitting Strategies

```
Strategy 1: By test target
  - UnitTests target → runs on simulator pool A
  - UITests target → runs on simulator pool B

Strategy 2: By test class
  - Xcode automatically distributes test classes across simulators

Strategy 3: By test plan configuration
  - Config A: iPhone 15 Pro — subset of tests
  - Config B: iPad Pro — subset of tests
```

## UI Tests in Xcode Cloud

### Simulator Selection

```
Workflow → Test Action → Destination
  → Platform: iOS Simulator
  → Device: iPhone 15 Pro (or Latest)
  → OS: iOS 18.0 (or Latest)
```

### UI Test Best Practices for CI

```swift
import XCTest

final class LoginUITests: XCTestCase {
    var app: XCUIApplication!

    override func setUp() {
        super.setUp()
        continueAfterFailure = false
        app = XCUIApplication()

        // Use launch arguments for CI-specific behavior
        if ProcessInfo.processInfo.environment["CI"] == "TRUE" {
            app.launchArguments.append("--uitesting")
            app.launchArguments.append("--reset-state")
        }

        app.launch()
    }

    func testLoginFlow() {
        // Use accessibility identifiers (not text labels)
        let emailField = app.textFields["loginEmailField"]
        let passwordField = app.secureTextFields["loginPasswordField"]
        let loginButton = app.buttons["loginButton"]

        // Wait for elements with timeout
        XCTAssertTrue(emailField.waitForExistence(timeout: 10))

        emailField.tap()
        emailField.typeText("test@example.com")

        passwordField.tap()
        passwordField.typeText("password123")

        loginButton.tap()

        // Wait for navigation
        let welcomeLabel = app.staticTexts["welcomeLabel"]
        XCTAssertTrue(welcomeLabel.waitForExistence(timeout: 15))
    }
}
```

### Handling Network in UI Tests

```swift
// ci_pre_xcodebuild.sh — Configure mock server for UI tests
if [[ "$CI_XCODEBUILD_ACTION" == "test"* ]]; then
    echo "Setting up mock server for UI tests..."
    # Start mock server or configure test environment
fi
```

```swift
// In the app — Check for CI environment
#if DEBUG
if ProcessInfo.processInfo.environment["CI"] == "TRUE" {
    // Use mock API client
    APIConfiguration.baseURL = "http://localhost:8080/mock"
}
#endif
```

### Screenshot Capture on Failure

```swift
override func tearDown() {
    if testRun?.failureCount ?? 0 > 0 {
        let screenshot = XCUIScreen.main.screenshot()
        let attachment = XCTAttachment(screenshot: screenshot)
        attachment.name = "Failure-\(name)"
        attachment.lifetime = .keepAlways
        add(attachment)
    }
    super.tearDown()
}
```

## Code Coverage

### Enabling Coverage

```
Workflow → Test Action → Code Coverage: Gather coverage data ✓
```

### Coverage Thresholds

Xcode Cloud doesn't enforce coverage thresholds natively, but you can check in scripts:

```bash
#!/bin/bash
# ci_post_xcodebuild.sh — Check coverage

if [[ "$CI_XCODEBUILD_ACTION" == "test"* ]] && [ -d "$CI_RESULT_BUNDLE_PATH" ]; then
    # Extract coverage percentage
    COVERAGE=$(xcrun xccov view --report --json "$CI_RESULT_BUNDLE_PATH" 2>/dev/null | \
        python3 -c "import sys,json; data=json.load(sys.stdin); print(f'{data.get(\"lineCoverage\", 0)*100:.1f}')" 2>/dev/null || echo "0")

    echo "Code Coverage: ${COVERAGE}%"

    # Fail if below threshold
    THRESHOLD=70
    if (( $(echo "$COVERAGE < $THRESHOLD" | bc -l) )); then
        echo "❌ Coverage ${COVERAGE}% is below threshold ${THRESHOLD}%"
        # Uncomment to fail the build:
        # exit 1
    else
        echo "✅ Coverage ${COVERAGE}% meets threshold ${THRESHOLD}%"
    fi
fi
```

## Swift Testing Framework in CI

### Configuration for Swift Testing

```swift
import Testing

// Parallel by default — great for CI
@Suite("API Tests")
struct APITests {
    @Test("Fetch users returns list")
    func fetchUsers() async throws {
        let users = try await APIClient.shared.fetchUsers()
        #expect(!users.isEmpty)
    }
}

// Serial when needed (database tests)
@Suite("Database Tests", .serialized)
struct DatabaseTests {
    @Test("Insert user")
    func insertUser() async throws { ... }

    @Test("Query user")
    func queryUser() async throws { ... }
}

// Tags for filtering
extension Tag {
    @Tag static var critical: Self
    @Tag static var slow: Self
    @Tag static var flaky: Self
}

@Test("Login flow", .tags(.critical))
func testLogin() async throws { ... }

@Test("Large data import", .tags(.slow))
func testLargeImport() async throws { ... }
```

### Test Timeouts

```swift
@Test("Network request", .timeLimit(.minutes(2)))
func testNetworkRequest() async throws {
    let result = try await longRunningRequest()
    #expect(result.isValid)
}
```

## XCTest in CI

### Handling Flaky Tests

```swift
// Mark known flaky tests to reduce CI noise
func testFlakyNetworkOperation() throws {
    // XCTExpectFailure marks as "expected failure" instead of "test failure"
    XCTExpectFailure("Network timing is inconsistent in CI") {
        // Test that sometimes fails in CI
    }
}
```

### Test Ordering

```swift
// Force specific order for dependent tests
// In test plan: Execution Order → Alphabetical
// Name tests with numeric prefixes:
func test01_CreateAccount() { ... }
func test02_Login() { ... }
func test03_CreateData() { ... }
func test04_VerifyData() { ... }
```

## Test Results Processing

### Parsing xcresult in Post Script

```bash
#!/bin/bash
# ci_post_xcodebuild.sh

if [ -d "$CI_RESULT_BUNDLE_PATH" ]; then
    echo "=== Test Results ==="

    # Get test summary
    xcrun xcresulttool get --format json \
        --path "$CI_RESULT_BUNDLE_PATH" 2>/dev/null | \
        python3 -c "
import sys, json
data = json.load(sys.stdin)
metrics = data.get('metrics', {})
print(f'Tests Run: {metrics.get(\"testsCount\", {}).get(\"_value\", \"N/A\")}')
print(f'Failures: {metrics.get(\"testsFailedCount\", {}).get(\"_value\", \"0\")}')
print(f'Skipped: {metrics.get(\"testsSkippedCount\", {}).get(\"_value\", \"0\")}')
" 2>/dev/null || echo "Could not parse results"
fi
```

## Anti-Patterns

### ❌ Running UI Tests on Every PR

```
// ❌ Bad — UI tests are slow and consume compute hours
PR Workflow: Unit Tests + UI Tests (30 min per PR)

// ✅ Good — Fast feedback for PRs
PR Workflow: Unit Tests only (5 min per PR)
Nightly Workflow: Unit + UI + Performance Tests
```

### ❌ Tests Depending on Network

```swift
// ❌ Bad — Fails when API is down
func testFetchUsers() async throws {
    let users = try await liveAPIClient.fetchUsers()
    XCTAssertFalse(users.isEmpty)
}

// ✅ Good — Use mocks in CI
func testFetchUsers() async throws {
    let client = MockAPIClient(returning: [User.mock])
    let users = try await client.fetchUsers()
    XCTAssertFalse(users.isEmpty)
}
```

### ❌ Hardcoded Timeouts

```swift
// ❌ Bad — CI simulators are slower
XCTAssertTrue(element.waitForExistence(timeout: 2))

// ✅ Good — Generous timeouts for CI
let timeout: TimeInterval = ProcessInfo.processInfo.environment["CI"] != nil ? 15 : 5
XCTAssertTrue(element.waitForExistence(timeout: timeout))
```

### ❌ Ignoring Test Results

```bash
# ❌ Bad — Build passes even with test failures
xcodebuild test || true

# ✅ Good — Let Xcode Cloud handle test failures naturally
# Don't override exit codes in scripts
```

## References

- [Xcode Cloud Testing](https://developer.apple.com/documentation/xcode/configuring-your-xcode-cloud-workflow-s-actions)
- [Test Plans](https://developer.apple.com/documentation/xcode/organizing-tests-to-improve-feedback)
- [Swift Testing Framework](https://developer.apple.com/documentation/testing)
- **scripts-reference.md** — CI script patterns
- **optimization.md** — Reducing test time
