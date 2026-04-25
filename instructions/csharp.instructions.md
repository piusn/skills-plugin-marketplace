---
description: "C# and .NET coding standards covering naming conventions, async/await patterns, dependency injection, error handling, LINQ, nullability, and modern C# features."
applyTo: "**/*.cs"
---

# C# and .NET Coding Standards (.NET 8+)

## Naming Conventions

- **PascalCase:** classes, methods, properties, public fields, enums, events, namespaces
- **camelCase:** local variables, parameters, private fields (prefix instance fields with `_`: `_logger`, `_userRepository`)
- **UPPER_CASE:** only for truly constant values (prefer PascalCase `const` in most cases)
- **Interfaces:** prefix with `I` (e.g., `IUserService`, `IOrderRepository`)
- **Async methods:** suffix with `Async` (e.g., `GetUserAsync`, `SaveChangesAsync`)
- **Boolean properties/methods:** prefix with `Is`, `Has`, `Can`, `Should` (e.g., `IsValid`, `HasPermission`, `CanExecute`)
- **No abbreviations:** `GetUserConfiguration` not `GetUsrCfg`, `cancellationToken` not `ct`
- **File names match class names exactly:** `UserService.cs` contains `class UserService`

## Modern C# Features (Prefer Latest)

- Use **file-scoped namespaces**: `namespace X;` not `namespace X { }`
- Use **`required` properties** where initialization is mandatory
- Use **primary constructors** for simple DI scenarios
- Use **pattern matching** (`is`, `switch` expressions) over type casting
- Use **`record`** for immutable data transfer objects
- Use **`init` setters** for immutable properties after construction
- Use **collection expressions** (`[1, 2, 3]`) where supported
- Use **raw string literals** (`"""`) for multi-line strings
- Use **`global using`** directives for frequently used namespaces

```csharp
// ✅ Good — modern C# style
namespace MyApp.Services;

public record UserDto(string Name, string Email);

public class UserService(IUserRepository repository, ILogger<UserService> logger)
{
    public async Task<UserDto?> GetUserAsync(Guid id, CancellationToken cancellationToken)
    {
        var user = await repository.GetByIdAsync(id, cancellationToken);
        return user switch
        {
            null => null,
            { IsActive: true } => new UserDto(user.Name, user.Email),
            _ => throw new InvalidOperationException($"User {id} is inactive")
        };
    }
}
```

## Async/Await

- All I/O-bound operations **MUST** be async
- Always accept and pass `CancellationToken` through the call chain
- **Never** use `.Result` or `.Wait()` — always `await`
- Use `ValueTask<T>` for hot paths where the result is often synchronous
- Use `ConfigureAwait(false)` in library code (not in ASP.NET controllers)
- Prefer `Task.WhenAll` for independent parallel operations
- Async methods that don't `await` should return the task directly (elide async/await)

```csharp
// ✅ Good — proper async with cancellation
public async Task<Order> ProcessOrderAsync(OrderRequest request, CancellationToken cancellationToken)
{
    ArgumentNullException.ThrowIfNull(request);

    var user = await _userRepository.GetByIdAsync(request.UserId, cancellationToken);
    var inventory = await _inventoryService.CheckAvailabilityAsync(request.Items, cancellationToken);

    return await _orderRepository.CreateAsync(new Order(user, inventory), cancellationToken);
}

// ✅ Good — parallel independent operations
public async Task<DashboardData> GetDashboardAsync(Guid userId, CancellationToken cancellationToken)
{
    var ordersTask = _orderService.GetRecentAsync(userId, cancellationToken);
    var notificationsTask = _notificationService.GetUnreadAsync(userId, cancellationToken);
    var statsTask = _statsService.GetSummaryAsync(userId, cancellationToken);

    await Task.WhenAll(ordersTask, notificationsTask, statsTask);

    return new DashboardData(ordersTask.Result, notificationsTask.Result, statsTask.Result);
}

// ✅ Good — elide async/await when just passing through
public Task<User?> GetByIdAsync(Guid id, CancellationToken cancellationToken)
    => _context.Users.FindAsync(new object[] { id }, cancellationToken).AsTask();
```

## Dependency Injection

- Use **constructor injection** — never `new` up services manually
- Register as the narrowest lifetime: `Transient` → `Scoped` → `Singleton`
- Inject `ILogger<T>` for logging, not `ILoggerFactory`
- Use `IOptions<T>` / `IOptionsMonitor<T>` for configuration, not raw config strings
- **Avoid service locator pattern** (`IServiceProvider.GetService<T>()`)
- One responsibility per service — if a constructor has >5 dependencies, split the class

```csharp
// ✅ Good — clean constructor injection with primary constructor
public class OrderService(
    IOrderRepository orderRepository,
    IPaymentGateway paymentGateway,
    ILogger<OrderService> logger,
    IOptions<OrderOptions> options) : IOrderService
{
    private readonly OrderOptions _options = options.Value;

    public async Task<Order> SubmitAsync(OrderRequest request, CancellationToken cancellationToken)
    {
        logger.LogInformation("Submitting order for user {UserId}", request.UserId);
        // ...
    }
}

// ✅ Good — registration
services.AddScoped<IOrderService, OrderService>();
services.AddSingleton<ICacheService, RedisCacheService>();
services.Configure<OrderOptions>(configuration.GetSection("Orders"));
```

## Error Handling

- Throw exceptions for **exceptional conditions**, not control flow
- Use **specific exception types**, not bare `Exception`
- Catch **specific exceptions**, not `catch (Exception ex)` (except at top-level middleware)
- Use `ArgumentNullException.ThrowIfNull()` and `ArgumentException.ThrowIfNullOrWhiteSpace()` for parameter validation
- Return **result types** (`Result<T>`) for expected failures in domain logic
- Always log the exception **with context**: `_logger.LogError(ex, "Failed to process order {OrderId}", orderId)`
- Use **structured logging** — never string interpolation in log messages

```csharp
// ✅ Good — specific exceptions with context
public async Task<User> GetUserAsync(Guid userId, CancellationToken cancellationToken)
{
    ArgumentNullException.ThrowIfNull(userId);

    var user = await _repository.GetByIdAsync(userId, cancellationToken)
        ?? throw new NotFoundException($"User {userId} not found");

    return user;
}

// ✅ Good — structured logging (NOT interpolation)
_logger.LogError(ex, "Failed to process order {OrderId} for user {UserId}", orderId, userId);

// ❌ Bad — string interpolation in log messages defeats structured logging
_logger.LogError(ex, $"Failed to process order {orderId} for user {userId}");
```

## LINQ

- Prefer **method syntax** for complex queries, **query syntax** for joins
- Use `Any()` not `Count() > 0` for existence checks
- Use `FirstOrDefault()` with null checks, not `Single()` unless uniqueness is guaranteed
- Avoid LINQ in tight loops — materialize with `.ToList()` when reused
- Use `AsNoTracking()` for read-only EF Core queries

```csharp
// ✅ Good
var hasOrders = await _context.Orders.AnyAsync(o => o.UserId == userId, cancellationToken);
var activeUsers = await _context.Users
    .AsNoTracking()
    .Where(u => u.IsActive)
    .OrderBy(u => u.Name)
    .ToListAsync(cancellationToken);

// ❌ Bad — Count for existence, missing AsNoTracking
var hasOrders = _context.Orders.Count(o => o.UserId == userId) > 0;
```

## Nullability

- Enable nullable reference types: `<Nullable>enable</Nullable>` in csproj
- Prefer `string?` over manual null checks — let the compiler enforce null safety
- Use `??` (null-coalescing) and `?.` (null-conditional) operators
- **Never suppress nullable warnings** (`!`) without a comment explaining why
- Use `[NotNullWhen(true)]` and `[MemberNotNull]` attributes for flow analysis

```csharp
// ✅ Good — compiler-enforced null safety
public async Task<UserDto?> FindUserAsync(string? email, CancellationToken cancellationToken)
{
    if (string.IsNullOrWhiteSpace(email))
        return null;

    var user = await _repository.FindByEmailAsync(email, cancellationToken);
    return user?.ToDto();
}

// ✅ Good — null-coalescing for defaults
var displayName = user.DisplayName ?? user.Email ?? "Unknown";

// ✅ Acceptable — suppression with justification
var userId = httpContext.User.FindFirst(ClaimTypes.NameIdentifier)!.Value; // Auth middleware guarantees this claim exists
```

## Code Organization

- **One class per file** (except nested classes)
- Group members in order: Fields → Constructor → Public Methods → Private Methods → Properties
- Keep methods **< 30 lines** — extract submethods for readability
- Keep classes **< 300 lines** — split large classes by responsibility
- Use `partial` classes **only** for generated code separation

```
src/
├── MyApp.Domain/           # Entities, value objects, domain events
├── MyApp.Application/      # Use cases, DTOs, interfaces
├── MyApp.Infrastructure/   # EF Core, HTTP clients, external services
└── MyApp.Api/              # Controllers, middleware, startup
```

## Documentation

- XML docs (`///`) on **all public APIs**
- Include `<param>`, `<returns>`, `<exception>` tags
- Document **WHY**, not WHAT — the code shows what
- Use `<inheritdoc/>` for interface implementations
- Add `// TODO:` with issue reference for known tech debt

```csharp
/// <summary>
/// Processes a refund for the specified order. Partial refunds are supported.
/// </summary>
/// <param name="orderId">The order to refund.</param>
/// <param name="amount">Refund amount. If null, refunds the full order total.</param>
/// <param name="cancellationToken">Cancellation token.</param>
/// <returns>The refund confirmation with transaction ID.</returns>
/// <exception cref="NotFoundException">Thrown when the order does not exist.</exception>
/// <exception cref="InvalidOperationException">Thrown when the order is not eligible for refund.</exception>
public async Task<RefundResult> ProcessRefundAsync(
    Guid orderId,
    decimal? amount = null,
    CancellationToken cancellationToken = default)
{
    // ...
}
```
