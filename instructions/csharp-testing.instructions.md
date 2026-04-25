---
description: "C# testing standards using xUnit, Moq, FluentAssertions, and Arrange-Act-Assert pattern. Covers unit test structure, mocking, and test data management."
applyTo: "**/*Test*.cs,**/*Tests*/**/*.cs,**/*.Tests/**/*.cs"
---

# C# Testing Standards

## Framework Stack

- **Test framework:** xUnit — attributes: `[Fact]`, `[Theory]`, `[InlineData]`
- **Mocking:** Moq — `Mock<T>`, `.Setup()`, `.Verify()`
- **Assertions:** FluentAssertions (preferred) or xUnit built-in
- **Test data:** AutoFixture for complex object creation
- **Integration testing:** `WebApplicationFactory<T>` for ASP.NET, Testcontainers for databases

## Test Naming Convention

```
ClassName_MethodName_Scenario_ExpectedBehavior
```

Examples:
- `UserService_GetById_WithValidId_ReturnsUser`
- `OrderProcessor_Submit_WithEmptyCart_ThrowsValidationException`
- `LoginController_Post_WithInvalidCredentials_Returns401`

## Test Structure (Arrange-Act-Assert)

Every test follows the AAA pattern with clear section comments:

```csharp
[Fact]
public async Task GetUserAsync_WithValidId_ReturnsUser()
{
    // Arrange
    var userId = Guid.NewGuid();
    var expectedUser = new User { Id = userId, Name = "Test" };
    _mockRepo.Setup(r => r.GetByIdAsync(userId, It.IsAny<CancellationToken>()))
             .ReturnsAsync(expectedUser);

    // Act
    var result = await _sut.GetUserAsync(userId, CancellationToken.None);

    // Assert
    result.Should().NotBeNull();
    result.Name.Should().Be("Test");
}
```

## Mocking Best Practices

- Mock only **external dependencies** (repositories, HTTP clients, services)
- **Don't mock the system under test** (SUT)
- **Don't mock value objects or DTOs** — use real instances
- Use `It.IsAny<T>()` sparingly — prefer specific values for clarity
- Verify interactions **only when side effects matter** (e.g., `_mockRepo.Verify(r => r.SaveAsync(...), Times.Once)`)
- Use `MockBehavior.Strict` for critical paths to catch unexpected calls

```csharp
// ✅ Good — specific setup values
_mockRepo.Setup(r => r.GetByIdAsync(userId, It.IsAny<CancellationToken>()))
         .ReturnsAsync(expectedUser);

// ✅ Good — verify meaningful side effect
_mockRepo.Verify(r => r.SaveAsync(It.Is<Order>(o => o.Status == OrderStatus.Confirmed),
    It.IsAny<CancellationToken>()), Times.Once);

// ❌ Bad — over-mocking with It.IsAny everywhere
_mockRepo.Setup(r => r.GetByIdAsync(It.IsAny<Guid>(), It.IsAny<CancellationToken>()))
         .ReturnsAsync(It.IsAny<User>());
```

## FluentAssertions Patterns

```csharp
// Object assertions
result.Should().NotBeNull();
result.Should().BeOfType<UserDto>();
result.Name.Should().Be("Expected");
result.Age.Should().BeGreaterThan(0);

// Collection assertions
collection.Should().HaveCount(3);
collection.Should().Contain(x => x.Id == targetId);
collection.Should().BeInAscendingOrder(x => x.Name);
collection.Should().OnlyContain(x => x.IsActive);
collection.Should().AllSatisfy(x => x.Status.Should().Be(Status.Active));

// Exception assertions
var act = () => _sut.ProcessAsync(invalidRequest, CancellationToken.None);
await act.Should().ThrowAsync<NotFoundException>()
    .WithMessage("*not found*");

// Equivalency (ignoring specific properties)
result.Should().BeEquivalentTo(expected, options => options
    .Excluding(x => x.CreatedAt)
    .Excluding(x => x.Id));
```

## Theory Tests (Parameterized)

Use `[Theory]` with `[InlineData]` for testing multiple inputs against the same logic:

```csharp
[Theory]
[InlineData("", false)]
[InlineData("valid@email.com", true)]
[InlineData("no-at-sign", false)]
[InlineData("user@domain.co.uk", true)]
public void IsValidEmail_WithInput_ReturnsExpected(string email, bool expected)
{
    var result = EmailValidator.IsValid(email);
    result.Should().Be(expected);
}
```

For complex test data, use `[MemberData]` or `[ClassData]`:

```csharp
public static IEnumerable<object[]> InvalidOrderData =>
[
    [new OrderRequest { Items = [] }, "Cart cannot be empty"],
    [new OrderRequest { Items = null! }, "Items is required"],
];

[Theory]
[MemberData(nameof(InvalidOrderData))]
public async Task Submit_WithInvalidOrder_ThrowsValidation(OrderRequest request, string expectedMessage)
{
    var act = () => _sut.SubmitAsync(request, CancellationToken.None);
    await act.Should().ThrowAsync<ValidationException>()
        .WithMessage($"*{expectedMessage}*");
}
```

## Test Fixture Pattern

```csharp
public class UserServiceTests : IAsyncLifetime
{
    private readonly Mock<IUserRepository> _mockRepo;
    private readonly Mock<IEmailService> _mockEmail;
    private readonly UserService _sut;

    public UserServiceTests()
    {
        _mockRepo = new Mock<IUserRepository>();
        _mockEmail = new Mock<IEmailService>();
        _sut = new UserService(
            _mockRepo.Object,
            _mockEmail.Object,
            NullLogger<UserService>.Instance);
    }

    public Task InitializeAsync() => Task.CompletedTask;
    public Task DisposeAsync() => Task.CompletedTask;
}
```

Use `IAsyncLifetime` when setup/teardown requires async operations (database seeding, container startup).

## What to Test

- ✅ Business logic and domain rules
- ✅ Input validation and edge cases (null, empty, boundary values)
- ✅ Error handling paths (exceptions, null returns, error results)
- ✅ State transitions and side effects
- ✅ Controller action results (status codes, response shapes)
- ✅ Middleware and filter behavior
- ✅ Mapping logic (entity → DTO, request → command)

## What NOT to Test

- ❌ Framework behavior (EF Core LINQ translation, ASP.NET routing)
- ❌ Simple property getters/setters
- ❌ Auto-generated code
- ❌ Third-party library internals
- ❌ Private methods directly (test through public API)
- ❌ Configuration registration (trust the DI container)

## Integration Tests (ASP.NET)

```csharp
public class UserApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public UserApiTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureTestServices(services =>
            {
                // Replace real services with test doubles
                services.AddScoped<IUserRepository, InMemoryUserRepository>();
            });
        }).CreateClient();
    }

    [Fact]
    public async Task GetUsers_ReturnsOkWithUsers()
    {
        // Act
        var response = await _client.GetAsync("/api/users");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);
        var users = await response.Content.ReadFromJsonAsync<List<UserDto>>();
        users.Should().NotBeEmpty();
    }

    [Fact]
    public async Task GetUser_WithInvalidId_ReturnsNotFound()
    {
        var response = await _client.GetAsync($"/api/users/{Guid.NewGuid()}");
        response.StatusCode.Should().Be(HttpStatusCode.NotFound);
    }
}
```

## Rules

- ⛔ **NEVER** modify an existing passing test to make new code work — fix the code, not the test
- ✅ One assertion concept per test (multiple asserts on the same object are OK)
- ✅ Tests must be **independent** — no shared mutable state between tests
- ✅ Tests must be **deterministic** — no flaky tests from timing, randomness, or external state
- ✅ Use `CancellationToken.None` in tests, not `default`
- ✅ Test both **success AND failure** paths
- ✅ Name test classes to match the class under test: `UserService` → `UserServiceTests`
- ✅ Mirror the source project structure in the test project
- ✅ Keep test files focused — one test class per file, one SUT per test class
