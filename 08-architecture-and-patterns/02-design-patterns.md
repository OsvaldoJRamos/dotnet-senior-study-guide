# Design Patterns

Design patterns from the GoF (Gang of Four) are reusable solutions to common problems in software design.

**Full reference:** [Refactoring Guru - Design Patterns](https://refactoring.guru/pt-br/design-patterns/what-is-pattern)

## Categories

### Creational (how to create objects)

| Pattern | Purpose | Usage example |
|---|---|---|
| **Singleton** | Ensures a single instance of a class | Logger, configuration |
| **Factory Method** | Delegates object creation to subclasses | Creating notifications (email, SMS, push) |
| **Abstract Factory** | Creates families of related objects | Cross-platform UI |
| **Builder** | Builds complex objects step by step | Building queries, complex DTOs |
| **Prototype** | Clones existing objects | Copying configurations |

### Structural (how to compose objects)

| Pattern | Purpose | Usage example |
|---|---|---|
| **Adapter** | Converts the interface of one class into another | Integrating third-party libraries |
| **Decorator** | Adds responsibilities dynamically | Adding logging, caching to services |
| **Facade** | Simplified interface for a complex subsystem | API gateway, facade service |
| **Proxy** | Access controller for an object | Lazy loading, caching, access control |
| **Composite** | Treats individual objects and compositions uniformly | Menus, tree structures |

### Behavioral (how objects communicate)

| Pattern | Purpose | Usage example |
|---|---|---|
| **Strategy** | Defines a family of interchangeable algorithms | Shipping calculation, validations |
| **Observer** | Notifies objects about state changes | Events, pub/sub |
| **Command** | Encapsulates an action as an object | Undo/redo, command queues |
| **Template Method** | Defines algorithm skeleton, subclasses define steps | Data processing |
| **Chain of Responsibility** | Passes request through a chain of handlers | Middleware, validation pipeline |
| **Mediator** | Centralizes communication between objects | MediatR, event bus |

---

## Creational Patterns — Detailed Examples

### Factory Method

The Factory Method pattern defines an interface for creating objects but lets **subclasses decide** which class to instantiate. The key is a **Creator** base class that contains shared workflow logic and exposes an abstract `CreateXxx()` method which each subclass overrides. The Creator calls its own factory method without knowing the concrete product type.

**When to use:** When a class cannot anticipate the type of objects it needs to create, or when you want subclasses to specify the objects they create while reusing the surrounding workflow.

```csharp
// 1. Product interface
public interface INotification
{
    void Send(string recipient, string message);
}

// 2. Concrete products
public class EmailNotification : INotification
{
    public void Send(string recipient, string message)
    {
        Console.WriteLine($"Sending EMAIL to {recipient}: {message}");
    }
}

public class SmsNotification : INotification
{
    public void Send(string recipient, string message)
    {
        Console.WriteLine($"Sending SMS to {recipient}: {message}");
    }
}

public class PushNotification : INotification
{
    public void Send(string recipient, string message)
    {
        Console.WriteLine($"Sending PUSH to {recipient}: {message}");
    }
}

// 3. Creator base class — defines the factory method and the shared workflow
public abstract class NotificationCreator
{
    // The factory method — subclasses decide which product to instantiate
    protected abstract INotification CreateNotification();

    // Shared workflow that uses the factory method
    public void Notify(string recipient, string message)
    {
        INotification notification = CreateNotification();
        notification.Send(recipient, message);
    }
}

// 4. Concrete creators — each overrides the factory method
public class EmailCreator : NotificationCreator
{
    protected override INotification CreateNotification() => new EmailNotification();
}

public class SmsCreator : NotificationCreator
{
    protected override INotification CreateNotification() => new SmsNotification();
}

public class PushCreator : NotificationCreator
{
    protected override INotification CreateNotification() => new PushNotification();
}

// 5. Usage — the caller picks a creator; the workflow stays the same
NotificationCreator creator = new EmailCreator();
creator.Notify("user@example.com", "Welcome!");
```

> **Tip:** In .NET, you often do not need a full factory hierarchy. A `Func<T>` delegate or a simple static method can serve as a lightweight factory when the creation logic is trivial. The variant shown in many tutorials — an `INotificationFactory` interface with `EmailNotificationFactory` / `SmsNotificationFactory` classes — is closer to **Simple Factory** or **Abstract Factory**; the defining trait of GoF Factory Method is the abstract method on a Creator base class that subclasses override.

### Builder

The Builder pattern separates the construction of a complex object from its representation, allowing the same construction process to create different representations. The **fluent** variant chains method calls for readability.

**When to use:** When an object requires many optional parameters, or when the construction involves multiple steps that should be readable and maintainable.

```csharp
// The complex object
public class HttpRequestConfig
{
    public string Url { get; init; } = string.Empty;
    public string Method { get; init; } = "GET";
    public Dictionary<string, string> Headers { get; init; } = new();
    public string? Body { get; init; }
    public TimeSpan Timeout { get; init; } = TimeSpan.FromSeconds(30);
    public int MaxRetries { get; init; } = 3;
}

// Fluent builder
public class HttpRequestBuilder
{
    private string _url = string.Empty;
    private string _method = "GET";
    private readonly Dictionary<string, string> _headers = new();
    private string? _body;
    private TimeSpan _timeout = TimeSpan.FromSeconds(30);
    private int _maxRetries = 3;

    public HttpRequestBuilder WithUrl(string url)
    {
        _url = url;
        return this;
    }

    public HttpRequestBuilder WithMethod(string method)
    {
        _method = method;
        return this;
    }

    public HttpRequestBuilder WithHeader(string key, string value)
    {
        _headers[key] = value;
        return this;
    }

    public HttpRequestBuilder WithBody(string body)
    {
        _body = body;
        return this;
    }

    public HttpRequestBuilder WithTimeout(TimeSpan timeout)
    {
        _timeout = timeout;
        return this;
    }

    public HttpRequestBuilder WithMaxRetries(int retries)
    {
        _maxRetries = retries;
        return this;
    }

    public HttpRequestConfig Build()
    {
        if (string.IsNullOrWhiteSpace(_url))
            throw new InvalidOperationException("URL is required.");

        return new HttpRequestConfig
        {
            Url = _url,
            Method = _method,
            Headers = new Dictionary<string, string>(_headers),
            Body = _body,
            Timeout = _timeout,
            MaxRetries = _maxRetries
        };
    }
}

// Usage — reads like a specification
var config = new HttpRequestBuilder()
    .WithUrl("https://api.example.com/orders")
    .WithMethod("POST")
    .WithHeader("Authorization", "Bearer token123")
    .WithHeader("Content-Type", "application/json")
    .WithBody("""{"item": "keyboard", "qty": 1}""")
    .WithTimeout(TimeSpan.FromSeconds(10))
    .WithMaxRetries(5)
    .Build();
```

> **Tip:** In modern C#, you can often achieve the same readability with `required` properties and object initializers. Use the Builder pattern when you need **validation during construction** or when the build process itself is non-trivial.

### Singleton

The Singleton pattern ensures a class has only one instance and provides a global point of access to it. However, the manual implementation has significant drawbacks.

**Why manual Singleton is problematic:**

| Problem | Explanation |
|---|---|
| Hard to test | Global state makes unit testing difficult — you cannot replace the instance |
| Hidden dependencies | Classes access the singleton directly instead of receiving it through the constructor |
| Thread safety | Correct double-checked locking is tricky and error-prone |
| Lifetime control | The instance lives forever — no way to dispose or recreate it |

```csharp
// BAD — manual Singleton
public class ManualSingleton
{
    private static ManualSingleton? _instance;
    private static readonly object _lock = new();

    private ManualSingleton() { }

    public static ManualSingleton Instance
    {
        get
        {
            if (_instance is null)
            {
                lock (_lock)
                {
                    _instance ??= new ManualSingleton();
                }
            }
            return _instance;
        }
    }
}

// GOOD — let the DI container manage the lifetime
builder.Services.AddSingleton<IMyService, MyService>();

// The class itself is a normal class with no awareness of being a singleton
public class MyService : IMyService
{
    private readonly ILogger<MyService> _logger;

    public MyService(ILogger<MyService> logger)
    {
        _logger = logger;
    }

    public void DoWork()
    {
        _logger.LogInformation("Working...");
    }
}
```

> **Tip:** The DI container in .NET gives you three lifetimes — `AddSingleton`, `AddScoped`, and `AddTransient`. Prefer DI over manual singleton every time. You get testability, lifetime control, and proper dependency injection for free.

---

## Structural Patterns — Detailed Examples

### Adapter

The Adapter pattern converts the interface of a class into another interface that clients expect. It lets classes work together that otherwise could not because of incompatible interfaces.

**When to use:** When integrating a third-party library whose API does not match your application's interface, or when wrapping legacy code.

```csharp
// Your application's interface
public interface IPaymentGateway
{
    Task<PaymentResult> ProcessPaymentAsync(decimal amount, string currency, string cardToken);
}

public record PaymentResult(bool Success, string TransactionId, string? ErrorMessage);

// Third-party SDK — you cannot modify this
public class StripeClient
{
    public async Task<StripeCharge> CreateChargeAsync(StripeChargeRequest request)
    {
        // Stripe SDK logic...
        await Task.Delay(100); // simulating API call
        return new StripeCharge { Id = "ch_abc123", Status = "succeeded" };
    }
}

public class StripeChargeRequest
{
    public long AmountInCents { get; set; }
    public string Currency { get; set; } = string.Empty;
    public string Source { get; set; } = string.Empty;
}

public class StripeCharge
{
    public string Id { get; set; } = string.Empty;
    public string Status { get; set; } = string.Empty;
}

// Adapter — bridges your interface to the third-party SDK
public class StripePaymentAdapter : IPaymentGateway
{
    private readonly StripeClient _stripeClient;

    public StripePaymentAdapter(StripeClient stripeClient)
    {
        _stripeClient = stripeClient;
    }

    public async Task<PaymentResult> ProcessPaymentAsync(
        decimal amount, string currency, string cardToken)
    {
        var request = new StripeChargeRequest
        {
            AmountInCents = (long)(amount * 100),
            Currency = currency.ToLower(),
            Source = cardToken
        };

        var charge = await _stripeClient.CreateChargeAsync(request);

        return new PaymentResult(
            Success: charge.Status == "succeeded",
            TransactionId: charge.Id,
            ErrorMessage: charge.Status != "succeeded" ? $"Charge status: {charge.Status}" : null
        );
    }
}

// DI registration
builder.Services.AddSingleton<StripeClient>();
builder.Services.AddScoped<IPaymentGateway, StripePaymentAdapter>();
```

> **Tip:** The Adapter pattern is one of the most frequently used in real-world .NET projects. Wrapping third-party dependencies behind your own interface makes it trivial to swap providers or mock them in tests.

### Decorator

The Decorator pattern attaches additional responsibilities to an object dynamically. Decorators provide a flexible alternative to subclassing for extending functionality.

**When to use:** When you need to add cross-cutting concerns (logging, caching, retry logic, validation) without modifying the original class.

```csharp
// Base interface
public interface IProductRepository
{
    Task<Product?> GetByIdAsync(Guid id);
    Task<IEnumerable<Product>> GetAllAsync();
}

// Core implementation
public class ProductRepository : IProductRepository
{
    private readonly AppDbContext _context;

    public ProductRepository(AppDbContext context)
    {
        _context = context;
    }

    public async Task<Product?> GetByIdAsync(Guid id)
        => await _context.Products.FindAsync(id);

    public async Task<IEnumerable<Product>> GetAllAsync()
        => await _context.Products.ToListAsync();
}

// Caching decorator — adds caching without touching the original class
public class CachedProductRepository : IProductRepository
{
    private readonly IProductRepository _inner;
    private readonly IMemoryCache _cache;
    private readonly TimeSpan _cacheDuration = TimeSpan.FromMinutes(5);

    public CachedProductRepository(IProductRepository inner, IMemoryCache cache)
    {
        _inner = inner;
        _cache = cache;
    }

    public async Task<Product?> GetByIdAsync(Guid id)
    {
        string cacheKey = $"product:{id}";

        return await _cache.GetOrCreateAsync(cacheKey, async entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = _cacheDuration;
            return await _inner.GetByIdAsync(id);
        });
    }

    public async Task<IEnumerable<Product>> GetAllAsync()
    {
        return await _cache.GetOrCreateAsync("products:all", async entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = _cacheDuration;
            return await _inner.GetAllAsync();
        }) ?? Enumerable.Empty<Product>();
    }
}

// Logging decorator — adds logging without touching the original class
public class LoggingProductRepository : IProductRepository
{
    private readonly IProductRepository _inner;
    private readonly ILogger<LoggingProductRepository> _logger;

    public LoggingProductRepository(IProductRepository inner, ILogger<LoggingProductRepository> logger)
    {
        _inner = inner;
        _logger = logger;
    }

    public async Task<Product?> GetByIdAsync(Guid id)
    {
        _logger.LogInformation("Getting product {ProductId}", id);
        var product = await _inner.GetByIdAsync(id);
        _logger.LogInformation("Product {ProductId} {Result}", id, product is null ? "not found" : "found");
        return product;
    }

    public async Task<IEnumerable<Product>> GetAllAsync()
    {
        _logger.LogInformation("Getting all products");
        var products = await _inner.GetAllAsync();
        _logger.LogInformation("Retrieved {Count} products", products.Count());
        return products;
    }
}

// DI registration — decorators wrap each other like layers
builder.Services.AddScoped<ProductRepository>();
builder.Services.AddScoped<IProductRepository>(sp =>
{
    var repo = sp.GetRequiredService<ProductRepository>();
    var cache = sp.GetRequiredService<IMemoryCache>();
    var logger = sp.GetRequiredService<ILogger<LoggingProductRepository>>();

    // Order: Logging → Caching → Repository
    var cached = new CachedProductRepository(repo, cache);
    return new LoggingProductRepository(cached, logger);
});
```

> **Tip:** Libraries like [Scrutor](https://github.com/khellang/Scrutor) simplify decorator registration in .NET DI with the `.Decorate<TInterface, TDecorator>()` extension method.

### Facade

The Facade pattern provides a simplified interface to a complex subsystem. It does not encapsulate the subsystem — it just provides a convenient entry point.

**When to use:** When a subsystem has many classes and the client only needs a subset of the functionality, or when you want to reduce coupling between the client and the subsystem.

```csharp
// Complex subsystem classes
public class InventoryService
{
    public bool CheckStock(string productId, int quantity)
    {
        // Check warehouse inventory...
        return true;
    }

    public void ReserveStock(string productId, int quantity)
    {
        // Reserve items in warehouse...
    }
}

public class PaymentService
{
    public bool ChargeCustomer(string customerId, decimal amount)
    {
        // Process payment...
        return true;
    }
}

public class ShippingService
{
    public string CreateShipment(string orderId, string address)
    {
        // Create shipping label...
        return "TRACK-12345";
    }
}

public class NotificationService
{
    public void SendOrderConfirmation(string email, string orderId, string trackingNumber)
    {
        // Send email...
    }
}

// Facade — simplifies the entire "place order" workflow
public class OrderFacade
{
    private readonly InventoryService _inventory;
    private readonly PaymentService _payment;
    private readonly ShippingService _shipping;
    private readonly NotificationService _notification;

    public OrderFacade(
        InventoryService inventory,
        PaymentService payment,
        ShippingService shipping,
        NotificationService notification)
    {
        _inventory = inventory;
        _payment = payment;
        _shipping = shipping;
        _notification = notification;
    }

    public OrderResult PlaceOrder(OrderRequest request)
    {
        // Step 1: Check inventory
        if (!_inventory.CheckStock(request.ProductId, request.Quantity))
            return OrderResult.Failed("Out of stock");

        // Step 2: Charge customer
        if (!_payment.ChargeCustomer(request.CustomerId, request.TotalAmount))
            return OrderResult.Failed("Payment declined");

        // Step 3: Reserve stock
        _inventory.ReserveStock(request.ProductId, request.Quantity);

        // Step 4: Create shipment
        string tracking = _shipping.CreateShipment(request.OrderId, request.ShippingAddress);

        // Step 5: Notify customer
        _notification.SendOrderConfirmation(request.Email, request.OrderId, tracking);

        return OrderResult.Succeeded(tracking);
    }
}

// The client only interacts with the facade
public class CheckoutController : ControllerBase
{
    private readonly OrderFacade _orderFacade;

    public CheckoutController(OrderFacade orderFacade) => _orderFacade = orderFacade;

    [HttpPost]
    public IActionResult PlaceOrder([FromBody] OrderRequest request)
    {
        var result = _orderFacade.PlaceOrder(request);
        return result.Success ? Ok(result) : BadRequest(result);
    }
}
```

> **Tip:** A Facade is not the same as a "God Service." The facade should **delegate** to specialized services, not contain the business logic itself. If your facade is growing large, you probably need to split the subsystem differently.

---

## Behavioral Patterns — Detailed Examples

### Strategy

The Strategy pattern defines a family of algorithms, encapsulates each one, and makes them interchangeable. It lets the algorithm vary independently from the clients that use it.

**When to use:** When you have multiple ways to perform an operation and want to select the appropriate one at runtime.

```csharp
// Strategy interface
public interface IShippingCalculator
{
    string Name { get; }
    decimal Calculate(Order order);
}

// Concrete strategies
public class StandardShipping : IShippingCalculator
{
    public string Name => "Standard (5-7 business days)";

    public decimal Calculate(Order order)
    {
        if (order.TotalAmount >= 100m)
            return 0m; // free shipping over $100

        return order.TotalWeight switch
        {
            <= 1.0m => 5.99m,
            <= 5.0m => 9.99m,
            _ => 14.99m
        };
    }
}

public class ExpressShipping : IShippingCalculator
{
    public string Name => "Express (2-3 business days)";

    public decimal Calculate(Order order)
    {
        decimal baseRate = 15.99m;
        decimal weightSurcharge = order.TotalWeight * 2.5m;
        return baseRate + weightSurcharge;
    }
}

public class OvernightShipping : IShippingCalculator
{
    public string Name => "Overnight (next business day)";

    public decimal Calculate(Order order)
    {
        decimal baseRate = 29.99m;
        decimal weightSurcharge = order.TotalWeight * 5.0m;
        return baseRate + weightSurcharge;
    }
}

// Context — uses the strategy
public class OrderService
{
    private readonly IEnumerable<IShippingCalculator> _calculators;

    public OrderService(IEnumerable<IShippingCalculator> calculators)
    {
        _calculators = calculators;
    }

    public ShippingQuote GetShippingQuote(Order order, string shippingMethod)
    {
        var calculator = _calculators
            .FirstOrDefault(c => c.Name.Contains(shippingMethod, StringComparison.OrdinalIgnoreCase))
            ?? throw new ArgumentException($"Unknown shipping method: {shippingMethod}");

        decimal cost = calculator.Calculate(order);
        return new ShippingQuote(calculator.Name, cost);
    }

    public IEnumerable<ShippingQuote> GetAllQuotes(Order order)
    {
        return _calculators.Select(c => new ShippingQuote(c.Name, c.Calculate(order)));
    }
}

// DI registration — register all strategies
builder.Services.AddScoped<IShippingCalculator, StandardShipping>();
builder.Services.AddScoped<IShippingCalculator, ExpressShipping>();
builder.Services.AddScoped<IShippingCalculator, OvernightShipping>();
builder.Services.AddScoped<OrderService>();
```

> **Tip:** Injecting `IEnumerable<IShippingCalculator>` gives you all registered implementations. This is a powerful .NET DI feature that works naturally with the Strategy pattern.

### Observer

The Observer pattern defines a one-to-many dependency between objects so that when one object changes state, all its dependents are notified and updated automatically. In C#, the built-in `event` keyword implements this pattern natively.

**When to use:** When changes in one object should trigger updates in other objects, and you want loose coupling between them.

```csharp
// Using C# events (the idiomatic way)
public class OrderEventArgs : EventArgs
{
    public Guid OrderId { get; init; }
    public string CustomerEmail { get; init; } = string.Empty;
    public decimal TotalAmount { get; init; }
}

// Subject — raises events
public class OrderProcessor
{
    public event EventHandler<OrderEventArgs>? OrderPlaced;
    public event EventHandler<OrderEventArgs>? OrderCancelled;

    public void PlaceOrder(Order order)
    {
        // Process the order...

        // Notify all observers
        OrderPlaced?.Invoke(this, new OrderEventArgs
        {
            OrderId = order.Id,
            CustomerEmail = order.CustomerEmail,
            TotalAmount = order.TotalAmount
        });
    }
}

// Observers — react to events
public class EmailObserver
{
    public void OnOrderPlaced(object? sender, OrderEventArgs e)
    {
        Console.WriteLine($"Sending confirmation email to {e.CustomerEmail}");
    }
}

public class AnalyticsObserver
{
    public void OnOrderPlaced(object? sender, OrderEventArgs e)
    {
        Console.WriteLine($"Tracking order {e.OrderId} — revenue: {e.TotalAmount:C}");
    }
}

// Wiring up observers
var processor = new OrderProcessor();
var emailObserver = new EmailObserver();
var analyticsObserver = new AnalyticsObserver();

processor.OrderPlaced += emailObserver.OnOrderPlaced;
processor.OrderPlaced += analyticsObserver.OnOrderPlaced;

// In a DI-based application, consider using MediatR notifications instead:
public record OrderPlacedNotification(Guid OrderId, string Email, decimal Total) : INotification;

public class SendConfirmationEmail : INotificationHandler<OrderPlacedNotification>
{
    public Task Handle(OrderPlacedNotification notification, CancellationToken ct)
    {
        // Send email...
        return Task.CompletedTask;
    }
}
```

### Chain of Responsibility

The Chain of Responsibility pattern passes a request along a chain of handlers. Each handler decides either to process the request or pass it to the next handler. **ASP.NET Core middleware is a textbook implementation of this pattern.**

**When to use:** When you want to decouple the sender of a request from its receivers, and more than one object may handle the request.

```csharp
// How ASP.NET Core middleware IS Chain of Responsibility:
app.UseExceptionHandler("/error");    // Handler 1: catches exceptions
app.UseHttpsRedirection();             // Handler 2: redirects HTTP → HTTPS
app.UseAuthentication();               // Handler 3: identifies the user
app.UseAuthorization();                // Handler 4: checks permissions
app.UseRateLimiter();                  // Handler 5: limits request rate
app.MapControllers();                  // Handler 6: routes to controller

// Each middleware calls next() to pass the request down the chain,
// or short-circuits by NOT calling next().

// Custom middleware — a handler in the chain
public class RequestTimingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestTimingMiddleware> _logger;

    public RequestTimingMiddleware(RequestDelegate next, ILogger<RequestTimingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var stopwatch = Stopwatch.StartNew();

        await _next(context); // pass to next handler in chain

        stopwatch.Stop();
        _logger.LogInformation(
            "{Method} {Path} completed in {Elapsed}ms",
            context.Request.Method,
            context.Request.Path,
            stopwatch.ElapsedMilliseconds);
    }
}

// Building your own chain (outside of ASP.NET):
public abstract class ValidationHandler
{
    private ValidationHandler? _next;

    public ValidationHandler SetNext(ValidationHandler next)
    {
        _next = next;
        return next;
    }

    public virtual ValidationResult Handle(OrderRequest request)
    {
        if (_next is not null)
            return _next.Handle(request);

        return ValidationResult.Success;
    }
}

public class StockValidationHandler : ValidationHandler
{
    public override ValidationResult Handle(OrderRequest request)
    {
        if (request.Quantity <= 0)
            return ValidationResult.Failure("Quantity must be greater than zero");

        return base.Handle(request); // pass to next handler
    }
}

public class CreditValidationHandler : ValidationHandler
{
    public override ValidationResult Handle(OrderRequest request)
    {
        if (request.TotalAmount > 10_000m)
            return ValidationResult.Failure("Amount exceeds credit limit");

        return base.Handle(request);
    }
}

// Usage
var stock = new StockValidationHandler();
var credit = new CreditValidationHandler();
stock.SetNext(credit);

var result = stock.Handle(new OrderRequest { Quantity = 5, TotalAmount = 500m });
```

### Mediator (MediatR)

The Mediator pattern defines an object that encapsulates how a set of objects interact. It promotes loose coupling by preventing objects from referring to each other directly. [MediatR](https://github.com/jbogard/MediatR) is the most popular implementation in .NET.

**When to use:** When you want to reduce coupling between components that need to communicate, especially in CQRS architectures where commands and queries flow through a central mediator.

```csharp
// 1. Define the command (request)
public record CreateOrderCommand(
    string CustomerId,
    string ProductId,
    int Quantity
) : IRequest<OrderResponse>;

public record OrderResponse(Guid OrderId, string Status);

// 2. Define the handler
public class CreateOrderHandler : IRequestHandler<CreateOrderCommand, OrderResponse>
{
    private readonly AppDbContext _context;
    private readonly ILogger<CreateOrderHandler> _logger;

    public CreateOrderHandler(AppDbContext context, ILogger<CreateOrderHandler> logger)
    {
        _context = context;
        _logger = logger;
    }

    public async Task<OrderResponse> Handle(
        CreateOrderCommand command, CancellationToken cancellationToken)
    {
        var order = new Order
        {
            Id = Guid.NewGuid(),
            CustomerId = command.CustomerId,
            ProductId = command.ProductId,
            Quantity = command.Quantity,
            Status = "Pending"
        };

        _context.Orders.Add(order);
        await _context.SaveChangesAsync(cancellationToken);

        _logger.LogInformation("Order {OrderId} created", order.Id);

        return new OrderResponse(order.Id, order.Status);
    }
}

// 3. Optional: pipeline behavior (cross-cutting concerns)
public class LoggingBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    private readonly ILogger<LoggingBehavior<TRequest, TResponse>> _logger;

    public LoggingBehavior(ILogger<LoggingBehavior<TRequest, TResponse>> logger)
    {
        _logger = logger;
    }

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        _logger.LogInformation("Handling {RequestName}", typeof(TRequest).Name);
        var response = await next();
        _logger.LogInformation("Handled {RequestName}", typeof(TRequest).Name);
        return response;
    }
}

// 4. Usage in a controller
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    private readonly IMediator _mediator;

    public OrdersController(IMediator mediator) => _mediator = mediator;

    [HttpPost]
    public async Task<IActionResult> Create([FromBody] CreateOrderCommand command)
    {
        var result = await _mediator.Send(command);
        return CreatedAtAction(nameof(Create), new { id = result.OrderId }, result);
    }
}

// 5. DI registration
builder.Services.AddMediatR(cfg =>
{
    cfg.RegisterServicesFromAssembly(typeof(Program).Assembly);
    cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
});
```

> **Tip:** MediatR pipeline behaviors are themselves an implementation of the **Chain of Responsibility** pattern — each behavior can process the request, pass it to the next behavior, and then process the response.

> **Licensing note:** In **April 2025**, Jimmy Bogard announced MediatR was going commercial; it launched in **July 2025** under **Lucky Penny Software** and now requires license key registration (current version **v14.1.0** as of early 2026). In-process mediator alternatives include hand-rolled `IRequest`/`IRequestHandler` patterns (a few interfaces + a dispatcher resolving handlers via DI) or community forks. Evaluate licensing terms before adopting in new projects.

### Template Method

The Template Method pattern defines the skeleton of an algorithm in a base class, letting subclasses override specific steps without changing the algorithm's structure.

**When to use:** When several classes share the same algorithm structure but differ in specific steps.

```csharp
// Base class defines the skeleton
public abstract class DataImporter<T>
{
    // Template method — defines the algorithm structure (sealed prevents override)
    public async Task<ImportResult> ImportAsync(string source)
    {
        var rawData = await ReadDataAsync(source);
        var parsed = Parse(rawData);
        var valid = Validate(parsed);
        await SaveAsync(valid);

        return new ImportResult(
            TotalRecords: parsed.Count,
            ValidRecords: valid.Count,
            SkippedRecords: parsed.Count - valid.Count
        );
    }

    // Steps that subclasses must implement
    protected abstract Task<string> ReadDataAsync(string source);
    protected abstract List<T> Parse(string rawData);

    // Step with default behavior that can be overridden
    protected virtual List<T> Validate(List<T> items) => items;

    protected abstract Task SaveAsync(List<T> items);
}

// Concrete implementation for CSV
public class CsvProductImporter : DataImporter<Product>
{
    private readonly AppDbContext _context;

    public CsvProductImporter(AppDbContext context) => _context = context;

    protected override async Task<string> ReadDataAsync(string source)
        => await File.ReadAllTextAsync(source);

    protected override List<Product> Parse(string rawData)
    {
        return rawData
            .Split('\n', StringSplitOptions.RemoveEmptyEntries)
            .Skip(1) // skip header
            .Select(line =>
            {
                var fields = line.Split(',');
                return new Product
                {
                    Name = fields[0].Trim(),
                    Price = decimal.Parse(fields[1].Trim()),
                    Stock = int.Parse(fields[2].Trim())
                };
            })
            .ToList();
    }

    protected override List<Product> Validate(List<Product> items)
        => items.Where(p => p.Price > 0 && !string.IsNullOrWhiteSpace(p.Name)).ToList();

    protected override async Task SaveAsync(List<Product> items)
    {
        _context.Products.AddRange(items);
        await _context.SaveChangesAsync();
    }
}

// Concrete implementation for JSON
public class JsonProductImporter : DataImporter<Product>
{
    private readonly AppDbContext _context;
    private readonly HttpClient _httpClient;

    public JsonProductImporter(AppDbContext context, HttpClient httpClient)
    {
        _context = context;
        _httpClient = httpClient;
    }

    protected override async Task<string> ReadDataAsync(string source)
        => await _httpClient.GetStringAsync(source);

    protected override List<Product> Parse(string rawData)
        => JsonSerializer.Deserialize<List<Product>>(rawData) ?? new();

    protected override async Task SaveAsync(List<Product> items)
    {
        _context.Products.AddRange(items);
        await _context.SaveChangesAsync();
    }
}
```

---

## Most used patterns in day-to-day .NET

### Singleton
```csharp
// In modern .NET, use DI with AddSingleton
builder.Services.AddSingleton<IMyService, MyService>();
```

### Strategy
```csharp
public interface IShippingCalculator
{
    decimal Calculate(Order order);
}

public class StandardShipping : IShippingCalculator { ... }
public class ExpressShipping : IShippingCalculator { ... }

// Usage via DI - the strategy is injected
public class OrderService
{
    private readonly IShippingCalculator _shipping;
    public OrderService(IShippingCalculator shipping) => _shipping = shipping;
}
```

### Repository
```csharp
public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(Guid id);
    Task<IEnumerable<T>> GetAllAsync();
    Task AddAsync(T entity);
    Task UpdateAsync(T entity);
    Task DeleteAsync(Guid id);
}
```

---

## Anti-Patterns

Anti-patterns are common responses to recurring problems that are **ineffective and counterproductive**. Recognizing them is just as important as knowing the correct patterns.

### God Class

A class that knows too much or does too much. It centralizes responsibilities that should be distributed.

```csharp
// BAD — God Class: manages users, sends emails, processes payments, generates reports
public class ApplicationManager
{
    public void CreateUser(string name, string email) { /* ... */ }
    public void SendEmail(string to, string subject, string body) { /* ... */ }
    public void ProcessPayment(decimal amount, string cardNumber) { /* ... */ }
    public void GenerateMonthlyReport() { /* ... */ }
    public void BackupDatabase() { /* ... */ }
    public void SyncInventory() { /* ... */ }
    // ... 2000 more lines
}

// GOOD — each class has a single responsibility
public class UserService { /* user operations */ }
public class EmailService { /* email operations */ }
public class PaymentService { /* payment operations */ }
public class ReportService { /* reporting operations */ }
```

**How to fix:** Apply the Single Responsibility Principle. If you cannot describe what a class does in one sentence without using "and," it probably does too much.

### Spaghetti Code

Code with no clear structure — tangled control flow, deeply nested conditionals, no separation of concerns.

```csharp
// BAD — everything tangled together
public IActionResult ProcessOrder(int id)
{
    var conn = new SqlConnection("Server=...");
    conn.Open();
    var cmd = new SqlCommand("SELECT * FROM Orders WHERE Id = " + id, conn);
    var reader = cmd.ExecuteReader();
    if (reader.Read())
    {
        var total = (decimal)reader["Total"];
        if (total > 0)
        {
            if (total > 100)
            {
                // apply discount
                total = total * 0.9m;
                // send email
                var smtp = new SmtpClient("smtp.example.com");
                // ... 50 more lines of nested logic
            }
        }
    }
    // ...
}

// GOOD — structured with clear layers and responsibilities
public async Task<IActionResult> ProcessOrder(int id)
{
    var order = await _orderRepository.GetByIdAsync(id);
    if (order is null) return NotFound();

    var discount = _discountCalculator.Calculate(order);
    order.ApplyDiscount(discount);

    await _orderRepository.UpdateAsync(order);
    await _notificationService.SendOrderConfirmation(order);

    return Ok(order);
}
```

**How to fix:** Introduce layers (controller → service → repository), extract methods, and follow SOLID principles.

### Golden Hammer

Using one familiar solution for every problem, regardless of whether it fits.

| Symptom | Example |
|---|---|
| Everything is a microservice | A simple CRUD app split into 15 services |
| Everything uses MediatR | Simple controller → service calls wrapped in commands for no reason |
| Everything is a stored procedure | Business logic living in the database |
| Everything is NoSQL | Using MongoDB for highly relational data |

**How to fix:** Choose tools based on the problem's requirements, not on familiarity. A monolith is fine for most projects. A simple service call is fine when you do not need CQRS.

### Premature Optimization

Optimizing code before you have evidence that it is a bottleneck. This often introduces complexity without measurable benefit.

```csharp
// BAD — premature micro-optimization, harder to read
public unsafe int SumArray(int[] arr)
{
    fixed (int* p = arr)
    {
        int sum = 0;
        for (int* ptr = p; ptr < p + arr.Length; ptr++)
            sum += *ptr;
        return sum;
    }
}

// GOOD — write clear code first, optimize when profiling shows a bottleneck
public int SumArray(int[] arr) => arr.Sum();
```

> **"Premature optimization is the root of all evil."** — Donald Knuth. Always **measure first** using tools like BenchmarkDotNet, dotnet-trace, or Application Insights before optimizing.

### Magic Numbers / Strings

Hardcoded values scattered throughout the code with no explanation of what they represent.

```csharp
// BAD — what do these numbers mean?
if (user.RoleId == 3)
    discount = total * 0.15m;

if (order.StatusId == 7)
    SendNotification(order, "smtp.internal.corp:587");

// GOOD — named constants or enums make intent clear
if (user.Role == UserRole.PremiumMember)
    discount = total * DiscountRates.Premium;

if (order.Status == OrderStatus.Shipped)
    _notificationService.SendShipmentNotification(order);
```

**How to fix:** Use constants, enums, or configuration for all meaningful values. If a value needs a comment to explain it, it should be a named constant.

### Copy-Paste Programming

Duplicating code blocks instead of extracting reusable abstractions. When a bug is found, it must be fixed in every copy.

```csharp
// BAD — same validation logic duplicated across controllers
public IActionResult CreateProduct(ProductDto dto)
{
    if (string.IsNullOrWhiteSpace(dto.Name)) return BadRequest("Name required");
    if (dto.Price <= 0) return BadRequest("Invalid price");
    if (dto.Name.Length > 200) return BadRequest("Name too long");
    // ...
}

public IActionResult UpdateProduct(Guid id, ProductDto dto)
{
    if (string.IsNullOrWhiteSpace(dto.Name)) return BadRequest("Name required");
    if (dto.Price <= 0) return BadRequest("Invalid price");
    if (dto.Name.Length > 200) return BadRequest("Name too long");
    // ...
}

// GOOD — extracted into a reusable validator (e.g., FluentValidation)
public class ProductDtoValidator : AbstractValidator<ProductDto>
{
    public ProductDtoValidator()
    {
        RuleFor(x => x.Name).NotEmpty().MaximumLength(200);
        RuleFor(x => x.Price).GreaterThan(0);
    }
}
```

**How to fix:** Follow the DRY principle. Extract shared logic into methods, base classes, or dedicated services.

### Lava Flow

Dead code, obsolete methods, commented-out blocks, and unused classes that nobody dares to remove because "it might be needed" or "nobody knows what it does."

```csharp
public class OrderService
{
    // TODO: remove after migration (added 2019-03-15)
    // public void ProcessOrderLegacy(int id) { ... }

    // Old implementation - DO NOT DELETE
    // public decimal CalculateShipping_v2(Order order) { ... }

    public void ProcessOrder(int id) { /* current implementation */ }

    [Obsolete("Use ProcessOrder instead")]
    public void ProcessOrderV3(int id) { /* nobody calls this */ }

    // Helper method — not sure if still used
    private decimal ApplyOldDiscountRules(Order order) { /* ... */ return 0; }
}
```

**How to fix:**
1. Use your IDE's "Find All References" to confirm code is unused
2. Check git history if you need to understand what the code did
3. Delete it — version control means it is never truly gone
4. Run tests after removal to confirm nothing breaks

---

## Patterns in the .NET Framework

The .NET framework itself is a rich showcase of design patterns. Recognizing them helps you understand the framework better and apply patterns correctly in your own code.

| Pattern | .NET Implementation | How it works |
|---|---|---|
| **Iterator** | `IEnumerable<T>` / `IEnumerator<T>` | `foreach` loops use the iterator pattern under the hood |
| **Decorator** | `Stream` → `BufferedStream`, `GZipStream` | Streams wrap other streams, adding behavior |
| **Factory Method** | `DbConnection` → `SqlConnection`, `NpgsqlConnection` | `DbProviderFactory.CreateConnection()` returns the concrete type |
| **Chain of Responsibility** | ASP.NET Core middleware pipeline | Each middleware decides to handle or pass the request |
| **Options** | `IOptions<T>`, `IOptionsSnapshot<T>`, `IOptionsMonitor<T>` | Strongly-typed access to configuration sections |
| **Facade** | `ILogger<T>` | Simple interface hiding complex logging infrastructure |
| **Observer** | `IObservable<T>` / `IObserver<T>`, `event` keyword | Built-in reactive and event-based notification |
| **Builder** | `WebApplicationBuilder`, `HostBuilder` | Fluent configuration of the application host |
| **Strategy** | `IComparer<T>`, `IEqualityComparer<T>` | Swap sorting/comparison algorithms at runtime |
| **Template Method** | `ControllerBase`, `BackgroundService` | Base classes with virtual/abstract methods for customization |

### Examples in practice

#### Iterator — `IEnumerable<T>` / `yield return`

```csharp
// The compiler generates an IEnumerator state machine from yield return
public static IEnumerable<int> EvenNumbers(int max)
{
    for (int i = 0; i <= max; i++)
    {
        if (i % 2 == 0)
            yield return i; // lazy evaluation — one item at a time
    }
}

// The consumer uses the iterator transparently
foreach (var number in EvenNumbers(100))
{
    Console.WriteLine(number);
}
```

#### Decorator — `Stream` wrapping

```csharp
// Each stream wraps the previous one, adding a layer of behavior
await using var fileStream = File.OpenRead("data.bin");                     // raw file I/O
await using var bufferedStream = new BufferedStream(fileStream);            // + buffering
await using var gzipStream = new GZipStream(bufferedStream,                // + compression
    CompressionMode.Decompress);
using var reader = new StreamReader(gzipStream);                           // + text decoding

string content = await reader.ReadToEndAsync();
```

#### Options pattern — `IOptions<T>`

```csharp
// appsettings.json
// {
//   "SmtpSettings": {
//     "Host": "smtp.example.com",
//     "Port": 587,
//     "EnableSsl": true
//   }
// }

public class SmtpSettings
{
    public string Host { get; set; } = string.Empty;
    public int Port { get; set; }
    public bool EnableSsl { get; set; }
}

// Registration
builder.Services.Configure<SmtpSettings>(
    builder.Configuration.GetSection("SmtpSettings"));

// Usage — injected as IOptions<T>
public class EmailService
{
    private readonly SmtpSettings _settings;

    public EmailService(IOptions<SmtpSettings> options)
    {
        _settings = options.Value;
    }
}
```

| Variant | Behavior | Use when |
|---|---|---|
| `IOptions<T>` | Read once at startup, never changes | Configuration is static |
| `IOptionsSnapshot<T>` | Re-reads per request (scoped) | Configuration may change between requests |
| `IOptionsMonitor<T>` | Notifies on change (singleton-safe) | Long-lived services need live updates |

---

[← Previous: SOLID](01-solid.md) | [Back to index](README.md) | [Next: KISS, DRY and YAGNI →](03-kiss-dry-yagni.md)
