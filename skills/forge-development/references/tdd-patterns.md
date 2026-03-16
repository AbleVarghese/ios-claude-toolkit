# TDD Patterns for Swift/iOS

## Test Naming Convention

```
test_methodName_whenCondition_shouldExpectedBehavior
```

Examples:
```swift
func test_execute_whenBalanceSufficient_shouldReturnCompletedTransfer() async throws
func test_execute_whenBalanceInsufficient_shouldThrowInsufficientFunds() async
func test_loadUser_whenNotFound_shouldSetErrorState() async
func test_validate_whenEmailMalformed_shouldReturnInvalidEmail()
func test_init_whenMissingRequiredFields_shouldThrowMissingConfiguration()
```

**Why this convention**: Reading the test name tells you exactly what's being tested, under what condition, and what the expected outcome is — without reading the test body.

---

## Arrange-Act-Assert Template

Every test follows this structure. No exceptions.

```swift
func test_executeTransfer_whenAmountExceedsLimit_shouldThrowLimitExceeded() async {
    // Arrange — Set up test data and dependencies
    let mockRepo = MockTransferRepository()
    let mockBalanceService = MockBalanceService(balance: Decimal(10_000))
    let sut = TransferService(
        repository: mockRepo,
        balanceService: mockBalanceService
    )
    let request = createTransferRequest(amount: Decimal(50_001))  // Over daily limit

    // Act — Execute the behavior under test
    var caughtError: TransferError?
    do {
        _ = try await sut.execute(request)
        XCTFail("Expected TransferError.dailyLimitExceeded")
    } catch let error as TransferError {
        caughtError = error
    } catch {
        XCTFail("Unexpected error type: \(error)")
    }

    // Assert — Verify the outcome
    XCTAssertEqual(caughtError, .dailyLimitExceeded(limit: Decimal(50_000)))
    XCTAssertNil(mockRepo.savedTransfer, "Should not save transfer when limit exceeded")
    XCTAssertEqual(mockBalanceService.debitCallCount, 0, "Should not debit when limit exceeded")
}
```

**Why Arrange-Act-Assert**: It creates a visual separation between setup, execution, and verification. When a test fails, you can immediately see which phase went wrong.

---

## Test Factory Pattern

Never hardcode test objects inline. Use factories with sensible defaults that tests override selectively.

```swift
// File: Tests/TestUtilities/Factories/TransferFactory.swift

enum TransferFactory {
    static func createTransfer(
        id: UUID = UUID(),
        senderId: UUID = UUID(),
        recipientId: UUID = UUID(),
        amount: Decimal = Decimal(100),
        currency: Currency = .usd,
        status: TransferStatus = .completed,
        createdAt: Date = Date()
    ) -> Transfer {
        Transfer(
            id: id,
            senderId: senderId,
            recipientId: recipientId,
            amount: amount,
            currency: currency,
            status: status,
            createdAt: createdAt
        )
    }

    static func createTransferRequest(
        senderId: UUID = UUID(),
        recipientId: UUID = UUID(),
        amount: Decimal = Decimal(100),
        currency: Currency = .usd
    ) -> TransferRequest {
        TransferRequest(
            senderId: senderId,
            recipientId: recipientId,
            amount: amount,
            currency: currency
        )
    }
}
```

Usage in tests:
```swift
// Only override what matters for THIS test
let transfer = TransferFactory.createTransfer(status: .pending)
let bigRequest = TransferFactory.createTransferRequest(amount: Decimal(999_999))
let euroTransfer = TransferFactory.createTransfer(currency: .eur, amount: Decimal(500))
```

**Why factories**: Tests only specify what's relevant to the test case. If the `Transfer` struct gains a new required property, you update ONE factory — not 50 tests.

---

## Mock Setup Patterns

### Protocol-Based Mocks

```swift
// File: Tests/TestUtilities/Mocks/MockTransferRepository.swift

final class MockTransferRepository: TransferRepositoryProtocol {
    // Track calls for verification
    private(set) var savedTransfer: Transfer?
    private(set) var saveCallCount = 0
    private(set) var findByIdCalls: [UUID] = []

    // Configurable responses
    var findByIdResult: Transfer?
    var saveError: Error?

    func save(_ transfer: Transfer) async throws {
        saveCallCount += 1
        if let error = saveError { throw error }
        savedTransfer = transfer
    }

    func findById(_ id: UUID) async throws -> Transfer? {
        findByIdCalls.append(id)
        return findByIdResult
    }
}
```

### Spy Pattern for Verifying Calls

```swift
final class MockNotificationService: NotificationServiceProtocol {
    // Capture all calls with their arguments
    private(set) var sentNotifications: [(userId: UUID, message: String)] = []

    func sendNotification(to userId: UUID, message: String) async throws {
        sentNotifications.append((userId: userId, message: message))
    }
}

// In test:
func test_executeTransfer_shouldNotifySenderAndRecipient() async throws {
    let mockNotifications = MockNotificationService()
    let sut = TransferService(notificationService: mockNotifications, ...)

    _ = try await sut.execute(request)

    // Verify both sender and recipient were notified
    XCTAssertEqual(mockNotifications.sentNotifications.count, 2)
    XCTAssertEqual(mockNotifications.sentNotifications[0].userId, request.senderId)
    XCTAssertEqual(mockNotifications.sentNotifications[1].userId, request.recipientId)
}
```

### Mock with Configurable Error Injection

```swift
final class MockBalanceService: BalanceServiceProtocol {
    private let balance: Decimal
    var debitError: Error?
    var creditError: Error?
    private(set) var debitedAmount: Decimal?
    private(set) var creditedAmount: Decimal?
    private(set) var debitCallCount = 0

    init(balance: Decimal) {
        self.balance = balance
    }

    func getBalance(for userId: UUID) async throws -> Decimal {
        balance
    }

    func debit(userId: UUID, amount: Decimal) async throws {
        debitCallCount += 1
        if let error = debitError { throw error }
        debitedAmount = amount
    }

    func credit(userId: UUID, amount: Decimal) async throws {
        if let error = creditError { throw error }
        creditedAmount = amount
    }
}
```

---

## Testing Async Code

### Async Test Methods (Preferred)

```swift
// ✅ Swift 5.5+ async test — clean, no XCTestExpectation needed
func test_loadProfile_whenServiceSucceeds_shouldPopulateUser() async {
    let mockService = MockUserService()
    mockService.getUserResult = UserFactory.createUser(name: "Jane")
    let viewModel = await ProfileViewModel(userService: mockService)

    await viewModel.loadProfile(id: UUID())

    let user = await viewModel.user
    XCTAssertEqual(user?.name, "Jane")
    let isLoading = await viewModel.isLoading
    XCTAssertFalse(isLoading)
}
```

### XCTestExpectation (When Needed for Callbacks)

```swift
// Only use when wrapping legacy callback-based APIs in tests
func test_legacyCallback_shouldComplete() {
    let expectation = expectation(description: "Callback should fire")
    let sut = LegacyService()

    sut.fetchData { result in
        XCTAssertNotNil(result)
        expectation.fulfill()
    }

    wait(for: [expectation], timeout: 5.0)
}
```

### Testing Error Paths

```swift
func test_loadProfile_whenServiceFails_shouldSetErrorState() async {
    let mockService = MockUserService()
    mockService.getUserError = AppError.networkTimeout
    let viewModel = await ProfileViewModel(userService: mockService)

    await viewModel.loadProfile(id: UUID())

    let error = await viewModel.error
    XCTAssertEqual(error, .networkTimeout)
    let user = await viewModel.user
    XCTAssertNil(user)
    let isLoading = await viewModel.isLoading
    XCTAssertFalse(isLoading, "isLoading must be false after error")
}
```

---

## Common Test Anti-Patterns

### Anti-Pattern 1: Testing Implementation Details

```swift
// ❌ WRONG — Testing internal method calls, not behavior
func test_wrong_testsInternals() async {
    let sut = TransferService(...)
    _ = try? await sut.execute(request)
    // Testing that an internal method was called is fragile.
    // Refactoring the internals breaks the test even when behavior is unchanged.
    XCTAssertTrue(sut.didCallInternalValidation)  // DON'T
}

// ✅ CORRECT — Testing observable behavior
func test_correct_testsBehavior() async throws {
    let mockRepo = MockTransferRepository()
    let sut = TransferService(repository: mockRepo, ...)
    let result = try await sut.execute(request)
    XCTAssertEqual(result.status, .completed)
    XCTAssertNotNil(mockRepo.savedTransfer)  // Observable side effect
}
```

### Anti-Pattern 2: Shared Mutable State Between Tests

```swift
// ❌ WRONG — Tests share state and depend on execution order
class BadTests: XCTestCase {
    static var sharedUser: User?  // Shared across ALL tests

    func test_first() { Self.sharedUser = UserFactory.createUser() }
    func test_second() { XCTAssertNotNil(Self.sharedUser) }  // Fails if run alone!
}

// ✅ CORRECT — Each test creates its own fixtures
class GoodTests: XCTestCase {
    func test_first() {
        let user = UserFactory.createUser()
        // ... test with user
    }
    func test_second() {
        let user = UserFactory.createUser()
        // ... independent test
    }
}
```

### Anti-Pattern 3: Sleeping Instead of Awaiting

```swift
// ❌ WRONG — Arbitrary sleep is flaky and slow
func test_wrong_sleeps() async {
    await viewModel.load()
    try? await Task.sleep(nanoseconds: 2_000_000_000)  // 2 second sleep!
    XCTAssertFalse(viewModel.isLoading)
}

// ✅ CORRECT — Use async/await, the test completes when the work completes
func test_correct_awaits() async {
    await viewModel.load()  // This returns when loading is done
    let isLoading = await viewModel.isLoading
    XCTAssertFalse(isLoading)
}
```

### Anti-Pattern 4: Test Interdependence

```swift
// ❌ WRONG — Tests must run in a specific order
func test_1_createUser() async throws {
    createdUserId = try await service.create(user)  // Sets up for test_2
}
func test_2_getUser() async throws {
    let user = try await service.get(createdUserId!)  // Depends on test_1
}

// ✅ CORRECT — Each test is fully self-contained
func test_createUser_shouldReturnNewUser() async throws {
    let user = try await service.create(UserFactory.createUser())
    XCTAssertNotNil(user.id)
}
func test_getUser_whenExists_shouldReturnUser() async throws {
    let created = try await service.create(UserFactory.createUser())
    let fetched = try await service.get(created.id)
    XCTAssertEqual(fetched?.id, created.id)
}
```

---

## Reset Mocks Between Tests

```swift
class TransferServiceTests: XCTestCase {
    private var mockRepo: MockTransferRepository!
    private var mockBalance: MockBalanceService!
    private var sut: TransferService!

    override func setUp() {
        super.setUp()
        mockRepo = MockTransferRepository()
        mockBalance = MockBalanceService(balance: Decimal(1000))
        sut = TransferService(repository: mockRepo, balanceService: mockBalance)
    }

    override func tearDown() {
        mockRepo = nil
        mockBalance = nil
        sut = nil
        super.tearDown()
    }

    // Each test starts with fresh mocks — no state leakage
}
```
