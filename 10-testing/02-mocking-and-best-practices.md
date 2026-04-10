# Mocking and Testing Best Practices

## Arrange-Act-Assert (AAA)

Pattern for organizing tests:

```csharp
[TestMethod]
public async Task ApproveOrder_ShouldUpdateStatus()
{
    // Arrange — set up the scenario
    var order = new Order("customer-1", 100m);
    var repo = new Mock<IOrderRepository>();
    repo.Setup(r => r.GetByIdAsync(order.Id)).ReturnsAsync(order);
    var service = new OrderService(repo.Object);

    // Act — execute the action
    await service.ApproveAsync(order.Id);

    // Assert — verify the result
    Assert.AreEqual(OrderStatus.Approved, order.Status);
    repo.Verify(r => r.SaveAsync(order), Times.Once);
}
```

## Moq (mocking framework)

### Setup and return

```csharp
var mock = new Mock<IProductRepository>();

// Simple return
mock.Setup(r => r.GetByIdAsync(1)).ReturnsAsync(new Product("Notebook"));

// Return with callback
mock.Setup(r => r.GetByIdAsync(It.IsAny<int>()))
    .ReturnsAsync((int id) => new Product($"Product-{id}"));

// Throw exception
mock.Setup(r => r.GetByIdAsync(-1))
    .ThrowsAsync(new NotFoundException());
```

### Matchers (It)

```csharp
It.IsAny<int>()                    // any int
It.Is<int>(x => x > 0)            // positive int
It.IsIn(1, 2, 3)                   // 1, 2, or 3
It.IsRegex("[A-Z]+")              // regex
```

### Verify (verify calls)

```csharp
// Verify it was called
mock.Verify(r => r.SaveAsync(It.IsAny<Order>()), Times.Once);

// Verify it was NOT called
mock.Verify(r => r.RemoveAsync(It.IsAny<int>()), Times.Never);

// Verify call with specific parameter
mock.Verify(r => r.SaveAsync(It.Is<Order>(p => p.Status == OrderStatus.Approved)));
```

## Naming Conventions

```csharp
// Pattern: Method_Scenario_ExpectedResult
[TestMethod]
public void Sum_PositiveNumbers_ShouldReturnSum()
public void Sum_WithZero_ShouldReturnOtherNumber()
public void Approve_AlreadyApproved_ShouldThrowException()
public void CreateUser_DuplicateEmail_ShouldReturnConflict()
```

## Testing exceptions

```csharp
[TestMethod]
[ExpectedException(typeof(DomainException))]
public void Approve_CancelledOrder_ShouldThrowException()
{
    var order = new Order("customer-1", 100m);
    order.Cancel();
    order.Approve(); // should throw
}

// Or with Assert (more control)
[TestMethod]
public void Approve_CancelledOrder_ShouldThrowException()
{
    var order = new Order("customer-1", 100m);
    order.Cancel();

    var ex = Assert.ThrowsException<DomainException>(() => order.Approve());
    Assert.AreEqual("Cannot approve a cancelled order", ex.Message);
}
```

## DataRow / DynamicData (data-driven tests)

```csharp
[TestMethod]
[DataRow(1, 1, 2)]
[DataRow(0, 0, 0)]
[DataRow(-1, 1, 0)]
[DataRow(int.MaxValue, 1, int.MinValue)] // overflow
public void Sum_WithVariousInputs(int a, int b, int expected)
{
    Assert.AreEqual(expected, Calculator.Sum(a, b));
}

// DynamicData for complex scenarios
[TestMethod]
[DynamicData(nameof(DiscountScenarios), DynamicDataSourceType.Method)]
public void CalculateDiscount_WithVariousScenarios(Order order, decimal expected)
{
    Assert.AreEqual(expected, order.CalculateDiscount());
}

private static IEnumerable<object[]> DiscountScenarios()
{
    yield return new object[] { new Order(100m, CustomerType.Regular), 0m };
    yield return new object[] { new Order(100m, CustomerType.Premium), 10m };
    yield return new object[] { new Order(1000m, CustomerType.Premium), 150m };
}
```

## Integration tests with WebApplicationFactory

```csharp
[TestClass]
public class OrderApiTests
{
    private WebApplicationFactory<Program> _factory;
    private HttpClient _client;

    [TestInitialize]
    public void Setup()
    {
        _factory = new WebApplicationFactory<Program>()
            .WithWebHostBuilder(builder =>
            {
                builder.ConfigureServices(services =>
                {
                    // Replace real database with in-memory
                    services.RemoveAll<DbContextOptions<AppDbContext>>();
                    services.AddDbContext<AppDbContext>(options =>
                        options.UseInMemoryDatabase("TestDb"));
                });
            });
        _client = _factory.CreateClient();
    }

    [TestMethod]
    public async Task CreateOrder_ShouldReturn201()
    {
        var dto = new { CustomerId = "c1", Value = 100m };
        var response = await _client.PostAsJsonAsync("/api/orders", dto);

        Assert.AreEqual(HttpStatusCode.Created, response.StatusCode);
    }
}
```

## Best practices

1. **Test behavior, not implementation** -- if the implementation changes but behavior stays the same, the test should not break
2. **One assert per test** (ideally) -- makes diagnosis easier
3. **Do not test private methods** -- test through the public interface
4. **Tests must be independent** -- do not depend on execution order
5. **Do not test the framework** -- do not test whether EF Core does INSERT correctly
6. **Use builders** to create complex test objects
7. **Avoid logic in tests** -- no if/else/for in tests

---

[← Previous: Testing Pyramid](01-testing-pyramid.md) | [Next: Stress and Load Testing →](03-stress-and-load-testing.md) | [Back to index](README.md)
