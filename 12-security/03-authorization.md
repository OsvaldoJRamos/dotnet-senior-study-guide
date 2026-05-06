# Authorization

Authentication asks *who*; authorization asks *can they*. The three models you must know for a senior interview are **RBAC**, **ABAC**, and **policy-based authorization** (ASP.NET Core's unifying abstraction that can express both).

## RBAC vs ABAC

| | RBAC (Role-Based) | ABAC (Attribute-Based) |
|---|---|---|
| Decision based on | User's role membership | Attributes of user, resource, action, environment |
| Example | *"Admins can delete orders"* | *"Managers can approve expenses below $10 000 within their own department, during business hours"* |
| Scaling | Becomes a role explosion once rules multiply | Expresses fine-grained rules without new roles |
| Complexity | Easy to reason about, easy to audit | Policies harder to reason about at scale |
| Good fit | Small-to-medium apps, small permission surface | Regulated / multi-tenant / fine-grained products |

Real systems almost always combine the two: a role gates the *broad* capability, and attributes narrow it (*"Manager role AND resource.department == user.department"*).

## Claims — the .NET primitive

In ASP.NET Core, the authenticated user is a `ClaimsPrincipal` with one or more `ClaimsIdentity`, each carrying a list of `Claim` (type + value, optionally issuer). **Roles are claims**, but not all claims are roles — treat "role" as one of many decision inputs.

```csharp
var identity = new ClaimsIdentity(new[]
{
    new Claim(ClaimTypes.NameIdentifier, user.Id),
    new Claim(ClaimTypes.Email, user.Email),
    new Claim("department", user.Department),
    new Claim("employee_grade", user.Grade.ToString()),
}, authenticationScheme: "cookie");

var principal = new ClaimsPrincipal(identity);
```

## Role-based authorization

The simplest form: decorate endpoints with `[Authorize(Roles = "...")]`. Multiple roles separated by commas are **OR**; multiple `[Authorize]` attributes are **AND**.

```csharp
[Authorize(Roles = "Admin, Manager")]       // Admin OR Manager
public IActionResult Index() { ... }

[Authorize(Roles = "Admin")]
[Authorize(Roles = "Manager")]              // Admin AND Manager
public IActionResult DualRole() { ... }
```

> Role name comparison is case-sensitive (typical `StringComparison.Ordinal` — `"admin"` and `"Admin"` are different roles). Policy name comparison is case-insensitive.

## Policy-based authorization

The preferred mechanism for anything non-trivial. A **policy** is one or more **requirements**; each requirement is evaluated by one or more **handlers**.

### 1. Define a requirement (data carrier)

```csharp
public class MinimumAgeRequirement : IAuthorizationRequirement
{
    public MinimumAgeRequirement(int minimumAge) => MinimumAge = minimumAge;
    public int MinimumAge { get; }
}
```

`IAuthorizationRequirement` is a marker interface (empty). The requirement just carries policy data.

### 2. Write the handler

```csharp
public class MinimumAgeHandler : AuthorizationHandler<MinimumAgeRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        MinimumAgeRequirement requirement)
    {
        var dobClaim = context.User.FindFirst(ClaimTypes.DateOfBirth);
        if (dobClaim is null) return Task.CompletedTask;

        var dob = DateTime.Parse(dobClaim.Value);
        var age = DateTime.UtcNow.Year - dob.Year;
        if (dob > DateTime.UtcNow.AddYears(-age)) age--;

        if (age >= requirement.MinimumAge)
            context.Succeed(requirement);

        return Task.CompletedTask;
    }
}
```

Rules the docs call out explicitly:

- `context.Succeed(requirement)` marks success. Not calling it = fail for this handler.
- `context.Fail()` forces failure even if another handler succeeds.
- By default, `AuthorizationOptions.InvokeHandlersAfterFailure` is `true` — all handlers run even after `Fail()`. Set to `false` to short-circuit.
- **Multiple handlers for the same requirement = OR.** Multiple requirements in the same policy = AND.

### 3. Register

```csharp
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("AtLeast21", policy =>
        policy.Requirements.Add(new MinimumAgeRequirement(21)));
});

builder.Services.AddSingleton<IAuthorizationHandler, MinimumAgeHandler>();
```

### 4. Apply

```csharp
// Controllers
[Authorize(Policy = "AtLeast21")]
public class AtLeast21Controller : Controller { ... }

// Minimal API / endpoints
app.MapGet("/drinks", () => "cheers")
   .RequireAuthorization("AtLeast21");

// Imperative (programmatic)
var result = await _authService.AuthorizeAsync(User, resource: null, "AtLeast21");
if (!result.Succeeded) return Forbid();
```

## AuthorizationPolicyBuilder shortcuts

For common checks you don't need a handler — use the builder:

```csharp
builder.Services.AddAuthorizationBuilder()
    .AddPolicy("EmployeeOnly",   p => p.RequireClaim("EmployeeNumber"))
    .AddPolicy("Founders",       p => p.RequireClaim("EmployeeNumber", "1", "2", "3"))
    .AddPolicy("RequireAdmin",   p => p.RequireRole("Admin"))
    .AddPolicy("RequireLogin",   p => p.RequireAuthenticatedUser())
    .AddPolicy("ContosoOnly",    p => p.RequireAssertion(ctx =>
        ctx.User.HasClaim(c => c.Type == "email"
            && c.Value.EndsWith("@contoso.com", StringComparison.OrdinalIgnoreCase))));
```

> `RequireAssertion` is convenient for one-off rules but **harder to test and less reusable** than a proper handler. Reach for a handler once the rule is used in more than one place or depends on DI services.

## Resource-based authorization (ABAC on a specific resource)

Role checks alone can't answer *"can this user edit **this** order?"*. Pass the resource to the authorization service:

```csharp
public class OrderController(IAuthorizationService authz) : ControllerBase
{
    [HttpPut("/orders/{id}")]
    public async Task<IActionResult> Update(int id, OrderDto dto)
    {
        var order = await _db.Orders.FindAsync(id);
        if (order is null) return NotFound();

        var result = await authz.AuthorizeAsync(User, order, "CanEditOrder");
        if (!result.Succeeded) return Forbid();

        // ... apply changes
        return NoContent();
    }
}

// Handler receives the resource
public class CanEditOrderHandler : AuthorizationHandler<CanEditOrderRequirement, Order>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext ctx,
        CanEditOrderRequirement req,
        Order resource)
    {
        if (resource.CustomerId == ctx.User.FindFirst("sub")?.Value)
            ctx.Succeed(req);
        return Task.CompletedTask;
    }
}
```

This is the idiomatic ASP.NET Core pattern for **IDOR prevention** (OWASP A01:2021).

## Picking between role, claim, and policy

| Rule you're expressing | Use |
|---|---|
| "Admins only" | `[Authorize(Roles = "Admin")]` |
| "Presence of a specific claim" | `RequireClaim` |
| "Any business rule richer than a single claim" | Requirement + Handler |
| "Depends on the specific resource (IDOR)" | Resource-based authorization (`AuthorizeAsync(user, resource, policy)`) |
| "One-liner that won't grow" | `RequireAssertion` |

## Anti-patterns

- **Authorization inside business logic.** Scatters rules across layers; replace with centralized policies.
- **"Admin everywhere."** A bloated Admin role becomes de-facto root. Split into scoped roles (`BillingAdmin`, `SupportAdmin`) or move to claims/policies.
- **Putting permissions in the JWT as a giant array.** Tokens grow, caching breaks, revocation becomes impossible. Prefer checking permissions on the server against a cache keyed by the user id.
- **Forgetting to check authorization on the server because the UI already hides the button.** The button is a hint; the server must enforce.
- **Role name drift.** `"admin"` in DB vs `"Admin"` in code — role matching is case-sensitive.

---

[← Previous: Authentication Patterns](02-authentication-patterns.md) | [Next: Secrets Management →](04-secrets-management.md) | [Back to index](README.md)
