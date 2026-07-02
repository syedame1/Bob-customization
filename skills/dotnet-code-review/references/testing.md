# Testing Checklist

When reviewing a diff/PR, "testing" usually means: does this change have adequate tests, and are any tests it touches still good tests? When reviewing a whole project, it also means assessing overall coverage and test quality patterns.

## Coverage gaps

- New logic (especially branching logic, edge cases, error paths) added with no corresponding test. The happy path being tested while error/edge cases aren't is the most common gap — check for: null/empty inputs, boundary values (0, -1, max), concurrent access if the code is meant to be thread-safe, and failure responses from dependencies (what happens if the downstream call throws or times out?).
- Bug fixes with no regression test — without one, nothing stops the same bug from coming back.
- Public API surface (controllers, public methods on services) with no tests at all, vs. only private/internal helpers being tested.

## Test quality

- **Testing implementation instead of behavior**: a test that breaks the moment internal implementation changes even though external behavior is unchanged (e.g., asserting a private method was called a specific number of times, or asserting internal field state, rather than asserting the observable outcome). This makes refactoring artificially risky.
- **Overly broad mocking**: mocking so much of the system under test that the test mostly verifies the mocks were called correctly rather than verifying real behavior. A sign: a test with more `Setup`/`Verify` lines than actual assertions about outcomes.
- **Missing Arrange-Act-Assert clarity**: tests where setup, action, and assertion are tangled together, making it hard to tell what's actually being tested. Doesn't need the literal comment structure, just the conceptual separation.
- **Assertion strength**: a test that only checks `result != null` or `Assert.IsTrue(success)` when it could meaningfully check the actual returned values/state — weak assertions pass even when the real behavior is wrong.
- **One test, one concept**: a single test method asserting many unrelated things — when it fails, it's unclear what actually broke. Prefer focused tests with descriptive names over one mega-test, though a single test covering several assertions about one cohesive operation's result is fine.

## Flakiness risks

- `Thread.Sleep` used to wait for async work to complete instead of properly awaiting it or using a polling/event-based wait — a classic source of flaky tests (works locally, fails intermittently in CI under different load).
- Tests depending on execution order, shared mutable state between tests (a static field, a shared database row), or wall-clock time (`DateTime.Now` comparisons without injecting a clock abstraction) — any of these can make a test pass or fail nondeterministically.
- Tests against a real external dependency (network call, real database, real clock) where a fake/in-memory/injected-clock version would be both faster and more reliable, unless this is deliberately an integration test (check naming/location conventions — integration test suites doing this intentionally is fine).

## Naming and discoverability

- Test names that don't describe the scenario and expected outcome (`Test1`, `UserTest`) vs. names that do (`CreateUser_WithDuplicateEmail_ThrowsValidationException`). This isn't pure style — a failing test with a descriptive name tells you what broke without opening the test body.

## Severity guidance for this category

Missing tests for new logic in a PR is typically Medium (High if the logic is security- or money-sensitive). Flaky-test patterns (`Thread.Sleep`, shared state) are Medium since they erode trust in the whole suite over time even though each instance seems minor. Naming and AAA-clarity issues are Low.