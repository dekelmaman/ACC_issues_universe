# Test-Driven Development (TDD)

## Required Capabilities
- Test-first development methodology
- Comprehensive behavior analysis (happy path, edge cases, errors)
- iOS test framework usage (XCTestCase, Swift Testing)
- Mock object design
- Coverage gap analysis

## Inputs
- Feature requirements (URLs, text descriptions, or both)
- Existing implementation files to analyze for coverage
- Platform target (iOS Swift, KMP, or both)

## Outputs
- Comprehensive test design documents
- Test implementations following TDD methodology
- Coverage analysis matrices
- Test result reports

## Patterns & Rules

### TDD Methodology

Follow this systematic flow:

1. **Gather requirements** -- understand what the feature must do
2. **Design behaviors** -- identify happy path, edge cases, error handling, integration points
3. **Write tests FIRST** -- implement all test cases before production code
4. **Run tests (expect failures)** -- verify tests actually test something
5. **Implement feature** -- write minimal code to make tests pass
6. **Run tests again** -- verify all pass
7. **Refactor** -- clean up while keeping tests green

### Test Naming Convention

Format: `Test [scenario] should [expected result]`

Examples:
- `Test lightIssueDetailsModelByUid with no fieldsMetadata should return defaults plus category fields`
- `test_shouldHandleValidInput`
- `test_shouldReturnNilWhenInputIsEmpty`

### AAA Pattern: Arrange, Act, Assert

Every test follows this structure:

```swift
func test_shouldDoSomething() {
    // Arrange - Setup test data
    let input = TestData.sample

    // Act - Execute the behavior
    let result = subject.performAction(with: input)

    // Assert - Verify the result
    XCTAssertEqual(result, expected)
}
```

### iOS-Specific Test Patterns

#### XCTestCase Structure

```swift
import XCTest
@testable import PlanGrid

class FeatureTests: XCTestCase {

    private var mockDependency: MockDependency!
    private var subject: Feature!

    override func setUp() {
        super.setUp()
        mockDependency = MockDependency()
        subject = Feature(dependency: mockDependency)
    }

    override func tearDown() {
        subject = nil
        mockDependency = nil
        super.tearDown()
    }

    // MARK: - Happy Path Tests

    func test_shouldHandleValidInput() {
        // Arrange
        let expected = TestData.sample
        mockDependency.stubbedResult = expected

        // Act
        let result = subject.performAction()

        // Assert
        XCTAssertEqual(result, expected)
    }

    // MARK: - Edge Case Tests

    func test_shouldReturnNilWhenInputIsEmpty() {
        let result = subject.performAction(with: "")
        XCTAssertNil(result)
    }

    // MARK: - Error Handling Tests

    func test_shouldHandleErrorWhenDependencyFails() {
        mockDependency.stubbedError = TestError.sample
        // ...
    }
}
```

#### Organize with // MARK: Comments

Group tests by category:
- `// MARK: - Happy Path Tests`
- `// MARK: - Edge Case Tests`
- `// MARK: - Error Handling Tests`
- `// MARK: - Async Tests`

#### Mock Objects as Private Classes

Define mocks at the bottom of the test file:

```swift
// MARK: - Mock Objects

private class MockDependency: DependencyProtocol {
    var didCallPerformAction = false
    var stubbedResult: DataType?
    var stubbedError: Error?

    func performAction() -> DataType? {
        didCallPerformAction = true
        return stubbedResult
    }
}
```

#### Parameterized Tests

```swift
func test_shouldHandleMultipleInputs() {
    parameterize(
        cases: ("input1", "expected1"),
               ("input2", "expected2"),
               ("input3", "expected3")
    ) { input, expected in
        let result = subject.transform(input)
        XCTAssertEqual(result, expected)
    }
}
```

### Coverage Analysis Matrix

When analyzing coverage gaps, produce a matrix:

| Requirement | Test Cases | Priority | Status |
|------------|------------|----------|--------|
| Core logic | Case 1, 2 | High | Not written |
| Error handling | Case 3 | High | Not written |
| Edge cases | Case 4, 5 | Medium | Not written |

### Prioritization Rules

Prioritize test creation by:
1. **Complexity** -- complex business logic first
2. **Business criticality** -- revenue/safety-critical paths first
3. **Error handling** -- failure modes before happy paths beyond the first
4. **Edge cases** -- boundary conditions and null/empty states

### What to Test

- Happy path with valid inputs
- Error handling (network failures, invalid data)
- Edge cases (null, empty, very large/small values)
- Boundary conditions (min/max values, limits)
- Async behavior (timeouts, race conditions)
- State transitions
- Business logic validation

### What NOT to Test

- Framework code (UIKit, SwiftUI, Android SDK)
- Third-party libraries
- Simple getters/setters without logic
- Auto-generated code
- Trivial pass-through methods

### Best Practices

- **One test per behavior** -- do not combine multiple assertions for different behaviors
- **Test behavior, not implementation** -- tests should survive refactoring
- **Make tests fail first** -- verify tests actually test something
- **Keep tests fast** -- fast feedback encourages running them often
- **Clean up in tearDown** -- prevent test interference
- **Mock external dependencies** -- network, database, file system
- **Do not mock value objects** -- test with real instances of data classes/structs
- **Do not mock the system under test** -- only mock its dependencies

### When to Use TDD

Use TDD when:
- Building a new feature from scratch
- Adding significant new behavior
- Working with complex business logic
- Requirements are well-defined

Skip TDD for:
- Quick bug fixes
- Trivial changes
- UI-only changes
- Experimental prototypes
