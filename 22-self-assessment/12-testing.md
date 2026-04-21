# Testing

> Read the questions, think about your answer, then click to reveal.

---

### 1. What is the testing pyramid, and why should the base be unit tests?

<details>
<summary>Reveal answer</summary>

The **testing pyramid** has three layers:
- **Unit tests** (base, most tests) -- test individual methods/classes in isolation. Fast, cheap, stable.
- **Integration tests** (middle) -- test multiple components together (API + DB, HTTP calls). Slower, more realistic.
- **E2E tests** (top, fewest tests) -- test the entire system through the UI or API. Slow, brittle, expensive.

The base is unit tests because they give the **fastest feedback loop**, are **cheapest to maintain**, and catch most logic bugs. Integration and E2E tests cover gaps that unit tests can't (wiring, infrastructure, real behavior).

Deep dive: [Testing Pyramid](../12-testing/01-testing-pyramid.md)

</details>

---

### 2. What is the difference between a mock, a stub, and a fake?

<details>
<summary>Reveal answer</summary>

- **Stub** -- returns pre-configured data. Used to control indirect inputs. You don't verify calls on it.
- **Mock** -- records interactions. You **verify** that specific methods were called with expected arguments.
- **Fake** -- a working implementation unsuitable for production (e.g., in-memory database, fake SMTP server).

```csharp
// Stub: just returns data
var stub = new Mock<IOrderRepo>();
stub.Setup(r => r.GetById(1)).Returns(order);

// Mock: verify interaction
mock.Verify(r => r.Save(It.IsAny<Order>()), Times.Once);
```

**Rule of thumb**: prefer stubs for most tests; use mocks sparingly to verify critical side effects.

Deep dive: [Testing Pyramid](../12-testing/01-testing-pyramid.md)

</details>

---

### 3. When should you mock and when should you NOT mock?

<details>
<summary>Reveal answer</summary>

**Mock when**:
- The dependency is slow (database, HTTP, file system).
- You need to test error paths (simulate exceptions, timeouts).
- You want to verify a side effect (email sent, event published).

**Don't mock when**:
- The dependency is a simple in-memory object (no I/O).
- You're mocking the thing you're testing (tests prove nothing).
- The mock setup is more complex than the real implementation.
- You're testing integration behavior -- use a real database or `WebApplicationFactory`.

Over-mocking leads to **tests that pass but don't prove your code works**.

Deep dive: [Mocking and Best Practices](../12-testing/02-mocking-and-best-practices.md)

</details>

---

### 4. What is `WebApplicationFactory` and when do you use it?

<details>
<summary>Reveal answer</summary>

`WebApplicationFactory<TEntryPoint>` is a test host from `Microsoft.AspNetCore.Mvc.Testing` that boots your entire ASP.NET Core app **in-memory** for integration testing. It creates an `HttpClient` that sends requests to the in-memory server.

```csharp
public class OrdersTests(WebApplicationFactory<Program> factory)
    : IClassFixture<WebApplicationFactory<Program>>
{
    [Fact]
    public async Task Get_Returns200()
    {
        var client = factory.CreateClient();
        var response = await client.GetAsync("/api/orders");
        response.StatusCode.Should().Be(HttpStatusCode.OK);
    }
}
```

You can override services (swap the real DB for an in-memory one) using `WithWebHostBuilder`.

Deep dive: [Mocking and Best Practices](../12-testing/02-mocking-and-best-practices.md)

</details>

---

### 5. What is the Arrange-Act-Assert (AAA) pattern?

<details>
<summary>Reveal answer</summary>

AAA structures every test into three clear phases:

1. **Arrange** -- set up the test data, dependencies, and the system under test.
2. **Act** -- call the method or perform the action being tested.
3. **Assert** -- verify the outcome (return value, state change, or interaction).

```csharp
[Fact]
public void Discount_AppliesTenPercent()
{
    // Arrange
    var calculator = new PriceCalculator();
    var price = 100m;

    // Act
    var result = calculator.ApplyDiscount(price, 0.10m);

    // Assert
    result.Should().Be(90m);
}
```

Keep each section **short and focused**. One logical assertion per test.

Deep dive: [Mocking and Best Practices](../12-testing/02-mocking-and-best-practices.md)

</details>

---

### 6. What makes a test "good"? Name a few properties.

<details>
<summary>Reveal answer</summary>

A well-known mnemonic is **FIRST**:

- **Fast** — runs in milliseconds so developers run the suite often.
- **Independent** — order doesn't matter; each test can run alone.
- **Repeatable** — same input, same result, every time (no time/random/network flakiness).
- **Self-validating** — the test either passes or fails; no manual inspection of logs.
- **Timely** — written close to the code (ideally before it, if doing TDD).

Add to this:
- **Focused** — one behavior per test; failures point to a specific cause.
- **Readable** — the arrange/act/assert reads like a spec.
- **Deterministic** — fail the same way locally and in CI.

Flaky tests are worse than no tests — they train the team to ignore failures.

Deep dive: [Mocking and Best Practices](../12-testing/02-mocking-and-best-practices.md)

</details>

---

### 7. What is the difference between load testing, stress testing, and soak testing?

<details>
<summary>Reveal answer</summary>

All three fall under performance testing but answer different questions:

| Type | Question it answers | Typical shape |
|------|---------------------|----------------|
| **Load test** | Does the system meet SLOs at expected traffic? | Ramp to target RPS, hold for a while |
| **Stress test** | Where does the system break, and how does it degrade? | Keep ramping until errors spike |
| **Soak / endurance test** | Does the system hold up over hours/days? (memory leaks, connection pool exhaustion, log disk fill) | Hold target load for hours |
| **Spike test** | How does the system react to sudden bursts? | Instant jump from idle to peak |

Run them against **production-like environments**, not localhost — real networking and disk I/O are part of the answer.

Deep dive: [Stress and Load Testing](../12-testing/03-stress-and-load-testing.md)

</details>

---

### 8. What metrics should you watch during a load/stress test?

<details>
<summary>Reveal answer</summary>

Tools like **k6** or **NBomber** simulate hundreds/thousands of concurrent users and report:

Application-level:
- **Response time percentiles** (p50, p95, p99) -- averages hide tail latency; the p99 is what angry users feel.
- **Throughput** (requests/second) -- sustained RPS before error rate rises.
- **Error rate** -- percentage of 5xx, timeouts, dropped connections.

Resource-level:
- **CPU / memory** on app and DB servers.
- **Connection pool exhaustion** -- DB or HTTP connections running out.
- **GC pressure** in .NET — Gen2 collections, allocation rate.
- **Queue depth** — message broker or request queue backing up.

Correlate all of these on the same timeline to find the actual bottleneck, not the first symptom.

Deep dive: [Stress and Load Testing](../12-testing/03-stress-and-load-testing.md)

</details>

---

### 9. When do you use `Setup` in Moq, and what's the risk of over-setup?

<details>
<summary>Reveal answer</summary>

`mock.Setup(x => x.Method(args)).Returns(value)` teaches the mock how to respond. Use it only for the behavior your test actually depends on.

Risks of over-setup:
- **Coupling tests to implementation details** — if the code starts calling the dependency slightly differently, all tests break even though behavior is fine.
- **Hiding missing interactions** — a mock that returns `default` for un-setup calls can make buggy code look correct.
- **Noise** — setup blocks that dwarf the act/assert make tests hard to read.

Use `MockBehavior.Strict` when you want the mock to fail on any call you didn't explicitly set up — it catches unexpected interactions but is verbose and brittle; reach for it only on mocks where every call matters.

Deep dive: [Mocking and Best Practices](../12-testing/02-mocking-and-best-practices.md)

</details>

---

### 10. What is data-driven testing, and how do you do it in MSTest / xUnit?

<details>
<summary>Reveal answer</summary>

Data-driven testing runs the **same test body** against multiple inputs so edge cases don't require copy-pasted test methods.

**MSTest** uses `[DataRow]` for inline rows or `[DynamicData]` for data from a static member:

```csharp
[TestMethod]
[DataRow(100, 0.10, 90)]
[DataRow(50, 0.25, 37.5)]
public void Discount_ReturnsExpected(decimal price, decimal discount, decimal expected)
{
    new PriceCalculator().ApplyDiscount(price, discount).Should().Be(expected);
}
```

**xUnit** uses `[Theory]` with `[InlineData]`, `[MemberData]`, or `[ClassData]`:

```csharp
[Theory]
[InlineData(100, 0.10, 90)]
[InlineData(50, 0.25, 37.5)]
public void Discount_ReturnsExpected(decimal price, decimal discount, decimal expected) { ... }
```

Keep the **act/assert** identical across rows. When a row needs different logic, make it its own test.

Deep dive: [Mocking and Best Practices](../12-testing/02-mocking-and-best-practices.md)

</details>

---
