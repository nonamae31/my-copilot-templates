---
name: savefood-dotnet-rules
description: >
  Expert .NET Backend rule system for the SaveFood project — a web marketplace for near-expiry food.
  Use this skill whenever writing, reviewing, scaffolding, or debugging any C# / ASP.NET Core code for SaveFood.
  Covers project structure, Repository Pattern, Identity + JWT + OAuth, AutoMapper DTOs, FluentValidation,
  Hangfire background jobs, SignalR notifications, VNPay payment, Global Exception Handling,
  and Render deployment. Trigger for ANY code task in the SaveFood solution.
---

# SaveFood — .NET Backend Master Rule

> **For AI coding agents**: These rules are ABSOLUTE. Never deviate unless the user explicitly overrides a specific rule.
> Read `references/architecture.md` for full folder tree and `references/code-samples.md` for copy-paste boilerplate.

---

## 0. Project Identity

| Key | Value |
|-----|-------|
| Solution name | `SaveFood.sln` |
| Runtime | .NET 8 (LTS) |
| IDE | Visual Studio 2022 Community |
| DB | SQL Server (LocalDB dev / Azure SQL prod) |
| ORM | Entity Framework Core 8 |
| Deploy | Render (Docker container) |
| Language | C# 12, nullable enabled |

---

## 1. Non-Negotiable Architecture Rules

1. **N-Tier + Repository Pattern only.** Flow: `Controller → Service → Repository → DbContext`. No business logic in Controllers. No raw DbContext calls outside Repositories.
2. **One DbContext** (`SaveFoodDbContext`) registered as `Scoped`.
3. **DTOs are mandatory** for every API input/output. Never expose EF models directly.
4. **AutoMapper** handles all Entity↔DTO mapping. No manual `new DTO { ... }` mapping blocks.
5. **FluentValidation** validates every request DTO. No `[Required]` DataAnnotations on DTOs (use FV rules instead).
6. **All async/await** — no `.Result` or `.Wait()` calls anywhere.
7. **Global Exception Handling Middleware** — never catch-and-swallow inside controllers.
8. **Interfaces before implementations** — every Repository and Service has an `I{Name}` interface in the same namespace.

---

## 2. Solution & Folder Structure

```
SaveFood/
├── SaveFood.sln
├── src/
│   ├── SaveFood.API/               ← ASP.NET Core Web API (entry point)
│   │   ├── Controllers/
│   │   ├── Middleware/             ← GlobalExceptionMiddleware, etc.
│   │   ├── Extensions/             ← IServiceCollection extension methods
│   │   ├── Program.cs
│   │   └── appsettings*.json
│   │
│   ├── SaveFood.Application/       ← Business logic layer
│   │   ├── DTOs/
│   │   │   ├── Auth/
│   │   │   ├── Product/
│   │   │   ├── Order/
│   │   │   └── Store/
│   │   ├── Interfaces/
│   │   │   ├── Repositories/
│   │   │   └── Services/
│   │   ├── Services/
│   │   ├── Validators/             ← FluentValidation validators
│   │   ├── Mappings/               ← AutoMapper profiles
│   │   └── Common/
│   │       ├── Result.cs           ← Result<T> pattern
│   │       └── PaginatedList.cs
│   │
│   ├── SaveFood.Domain/            ← EF Core models, enums, constants
│   │   ├── Entities/
│   │   ├── Enums/
│   │   └── Constants/
│   │
│   └── SaveFood.Infrastructure/    ← EF Core, Repositories, external services
│       ├── Data/
│       │   ├── SaveFoodDbContext.cs
│       │   ├── Configurations/     ← IEntityTypeConfiguration<T>
│       │   └── Migrations/
│       ├── Repositories/
│       ├── Services/               ← Email, VNPay, SignalR hub logic
│       └── Jobs/                   ← Hangfire job classes
│
├── tests/
│   └── SaveFood.Tests/
└── Dockerfile
```

> Full annotated tree → `references/architecture.md`

---

## 3. Domain Entities (Canonical)

```csharp
// Domain/Entities/Product.cs
public class Product : BaseEntity
{
    public string Name { get; set; } = null!;
    public string Description { get; set; } = string.Empty;
    public decimal OriginalPrice { get; set; }
    public decimal DiscountedPrice { get; set; }
    public int StockQuantity { get; set; }
    public DateTime ExpiryDate { get; set; }
    public ProductStatus Status { get; set; } = ProductStatus.Active;
    public Guid StoreId { get; set; }
    public Store Store { get; set; } = null!;
    public ICollection<OrderItem> OrderItems { get; set; } = new List<OrderItem>();
}

// Domain/Enums/ProductStatus.cs
public enum ProductStatus { Active, Expired, SoldOut, Removed }
public enum OrderStatus { Pending, Confirmed, Preparing, Delivered, Cancelled }
public enum UserRole { Admin, StoreOwner, Customer }
```

> All entities inherit `BaseEntity` (Id: Guid, CreatedAt, UpdatedAt, IsDeleted).
> **Soft delete only** — never `DELETE` rows; set `IsDeleted = true`.

---

## 4. Repository Pattern

### Interface (Application layer)
```csharp
// Application/Interfaces/Repositories/IProductRepository.cs
public interface IProductRepository : IBaseRepository<Product>
{
    Task<IEnumerable<Product>> GetActiveByStoreAsync(Guid storeId, CancellationToken ct = default);
    Task<IEnumerable<Product>> GetExpiringWithinAsync(int hours, CancellationToken ct = default);
}
```

### Base interface
```csharp
public interface IBaseRepository<T> where T : BaseEntity
{
    Task<T?> GetByIdAsync(Guid id, CancellationToken ct = default);
    Task<IEnumerable<T>> GetAllAsync(CancellationToken ct = default);
    Task AddAsync(T entity, CancellationToken ct = default);
    void Update(T entity);
    void Delete(T entity);   // soft delete
    Task<int> SaveChangesAsync(CancellationToken ct = default);
}
```

### Implementation (Infrastructure layer)
```csharp
// Infrastructure/Repositories/ProductRepository.cs
public class ProductRepository : BaseRepository<Product>, IProductRepository
{
    public ProductRepository(SaveFoodDbContext ctx) : base(ctx) { }

    public async Task<IEnumerable<Product>> GetActiveByStoreAsync(Guid storeId, CancellationToken ct = default)
        => await _ctx.Products
            .Where(p => p.StoreId == storeId
                     && p.Status == ProductStatus.Active
                     && !p.IsDeleted
                     && p.ExpiryDate > DateTime.UtcNow)
            .AsNoTracking()
            .ToListAsync(ct);

    public async Task<IEnumerable<Product>> GetExpiringWithinAsync(int hours, CancellationToken ct = default)
        => await _ctx.Products
            .Where(p => p.ExpiryDate <= DateTime.UtcNow.AddHours(hours)
                     && p.Status == ProductStatus.Active
                     && !p.IsDeleted)
            .ToListAsync(ct);
}
```

---

## 5. DTO & AutoMapper Rules

- **Request DTOs**: suffix `Request` (e.g., `CreateProductRequest`)
- **Response DTOs**: suffix `Response` (e.g., `ProductResponse`)
- **No nested entity objects** in DTOs — flatten or use dedicated nested response DTOs

```csharp
// Application/DTOs/Product/CreateProductRequest.cs
public record CreateProductRequest(
    string Name,
    string Description,
    decimal OriginalPrice,
    decimal DiscountedPrice,
    int StockQuantity,
    DateTime ExpiryDate
);

// Application/Mappings/ProductProfile.cs
public class ProductProfile : Profile
{
    public ProductProfile()
    {
        CreateMap<Product, ProductResponse>()
            .ForMember(d => d.StoreName, o => o.MapFrom(s => s.Store.Name))
            .ForMember(d => d.IsExpiringSoon,
                o => o.MapFrom(s => s.ExpiryDate <= DateTime.UtcNow.AddHours(24)));

        CreateMap<CreateProductRequest, Product>()
            .ForMember(d => d.Id, o => o.MapFrom(_ => Guid.NewGuid()))
            .ForMember(d => d.Status, o => o.MapFrom(_ => ProductStatus.Active));
    }
}
```

---

## 6. FluentValidation Rules

```csharp
// Application/Validators/CreateProductRequestValidator.cs
public class CreateProductRequestValidator : AbstractValidator<CreateProductRequest>
{
    public CreateProductRequestValidator()
    {
        RuleFor(x => x.Name).NotEmpty().MaximumLength(200);
        RuleFor(x => x.OriginalPrice).GreaterThan(0);
        RuleFor(x => x.DiscountedPrice)
            .GreaterThan(0)
            .LessThan(x => x.OriginalPrice)
            .WithMessage("Discounted price must be less than original price.");
        RuleFor(x => x.StockQuantity).GreaterThanOrEqualTo(1);
        RuleFor(x => x.ExpiryDate)
            .GreaterThan(DateTime.UtcNow)
            .WithMessage("Expiry date must be in the future.");
    }
}
```

Register in `Program.cs`:
```csharp
builder.Services.AddValidatorsFromAssemblyContaining<CreateProductRequestValidator>();
builder.Services.AddFluentValidationAutoValidation();
```

---

## 7. Authentication: JWT + Identity + OAuth

See full setup → `references/auth-setup.md`

**Rules:**
- Use `ASP.NET Core Identity` with custom `ApplicationUser : IdentityUser<Guid>`
- JWT issued on `/api/auth/login` and `/api/auth/register`
- Refresh token stored in DB (`RefreshToken` entity, hashed)
- Google OAuth via `Microsoft.AspNetCore.Authentication.Google`
- Facebook OAuth via `Microsoft.AspNetCore.Authentication.Facebook`
- Role-based authorization: `[Authorize(Roles = "Admin")]`, `[Authorize(Roles = "StoreOwner")]`

```csharp
// Domain/Entities/ApplicationUser.cs
public class ApplicationUser : IdentityUser<Guid>
{
    public string FullName { get; set; } = null!;
    public string? AvatarUrl { get; set; }
    public UserRole Role { get; set; } = UserRole.Customer;
    public Guid? StoreId { get; set; }      // null for Customer/Admin
    public Store? Store { get; set; }
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
}
```

JWT config (`appsettings.json`):
```json
"Jwt": {
  "Key": "YOUR_SECRET_256BIT_KEY_HERE",
  "Issuer": "SaveFoodAPI",
  "Audience": "SaveFoodClient",
  "ExpiryMinutes": 60,
  "RefreshTokenExpiryDays": 7
}
```

---

## 8. Global Exception Handling

```csharp
// API/Middleware/GlobalExceptionMiddleware.cs
public class GlobalExceptionMiddleware(RequestDelegate next, ILogger<GlobalExceptionMiddleware> logger)
{
    public async Task InvokeAsync(HttpContext context)
    {
        try { await next(context); }
        catch (Exception ex) { await HandleExceptionAsync(context, ex, logger); }
    }

    private static async Task HandleExceptionAsync(HttpContext ctx, Exception ex, ILogger logger)
    {
        logger.LogError(ex, "Unhandled exception: {Message}", ex.Message);
        var (status, code, message) = ex switch
        {
            SecurityTokenExpiredException  => (401, "TOKEN_EXPIRED",    "Your session has expired. Please log in again."),
            UnauthorizedAccessException    => (403, "FORBIDDEN",         "You do not have permission."),
            ProductExpiredException        => (422, "PRODUCT_EXPIRED",   "This product has already expired."),
            InsufficientStockException     => (422, "OUT_OF_STOCK",      "Insufficient stock available."),
            NotFoundException              => (404, "NOT_FOUND",         ex.Message),
            ValidationException            => (400, "VALIDATION_ERROR",  ex.Message),
            _                              => (500, "SERVER_ERROR",      "An unexpected error occurred.")
        };
        ctx.Response.StatusCode = status;
        ctx.Response.ContentType = "application/json";
        await ctx.Response.WriteAsJsonAsync(new { code, message, timestamp = DateTime.UtcNow });
    }
}
```

**Custom business exceptions** live in `Application/Common/Exceptions/`:
`ProductExpiredException`, `InsufficientStockException`, `NotFoundException`, `StoreNotFoundException`

---

## 9. Background Jobs (Hangfire)

```csharp
// Infrastructure/Jobs/ProductExpiryJob.cs
public class ProductExpiryJob(IProductRepository repo, IHubContext<NotificationHub> hub)
{
    // Called by Hangfire every 30 minutes
    [AutomaticRetry(Attempts = 3)]
    public async Task ExpireProductsAsync()
    {
        var expired = await repo.GetExpiringWithinAsync(0); // already past expiry
        foreach (var product in expired)
        {
            product.Status = ProductStatus.Expired;
            repo.Update(product);
            await hub.Clients.Group($"store-{product.StoreId}")
                .SendAsync("ProductExpired", new { product.Id, product.Name });
        }
        await repo.SaveChangesAsync();
    }
}
```

Register in `Program.cs`:
```csharp
builder.Services.AddHangfire(cfg => cfg.UseSqlServerStorage(connectionString));
builder.Services.AddHangfireServer();
// ...
app.UseHangfireDashboard("/hangfire", new DashboardOptions { Authorization = [new HangfireAdminFilter()] });
RecurringJob.AddOrUpdate<ProductExpiryJob>("expire-products", j => j.ExpireProductsAsync(), "*/30 * * * *");
```

---

## 10. SignalR Real-Time Notifications

```csharp
// Infrastructure/Services/NotificationHub.cs
[Authorize]
public class NotificationHub : Hub
{
    public override async Task OnConnectedAsync()
    {
        var userId = Context.UserIdentifier!;
        var storeId = Context.User?.FindFirst("storeId")?.Value;
        if (storeId != null) await Groups.AddToGroupAsync(Context.ConnectionId, $"store-{storeId}");
        await Groups.AddToGroupAsync(Context.ConnectionId, $"user-{userId}");
        await base.OnConnectedAsync();
    }
}
```

Notification events: `OrderPlaced`, `OrderStatusChanged`, `ProductExpired`, `LowStock`

---

## 11. VNPay Payment Integration

```csharp
// Infrastructure/Services/VNPayService.cs
public interface IVNPayService
{
    string CreatePaymentUrl(Order order, HttpContext context);
    VNPayResponse ProcessCallback(IQueryCollection query);
}
```

Config (`appsettings.json`):
```json
"VNPay": {
  "TmnCode": "YOUR_TMN_CODE",
  "HashSecret": "YOUR_HASH_SECRET",
  "BaseUrl": "https://sandbox.vnpayment.vn/paymentv2/vpcpay.html",
  "ReturnUrl": "https://yourdomain.com/api/payment/vnpay-return"
}
```

> Full VNPay implementation → `references/payment.md`

---

## 12. Mandatory NuGet Packages

| Package | Version | Purpose |
|---------|---------|---------|
| `Microsoft.AspNetCore.Identity.EntityFrameworkCore` | 8.x | Identity + EF |
| `Microsoft.EntityFrameworkCore.SqlServer` | 8.x | SQL Server provider |
| `Microsoft.EntityFrameworkCore.Tools` | 8.x | Migrations CLI |
| `Microsoft.AspNetCore.Authentication.JwtBearer` | 8.x | JWT auth |
| `Microsoft.AspNetCore.Authentication.Google` | 8.x | Google OAuth |
| `Microsoft.AspNetCore.Authentication.Facebook` | 8.x | Facebook OAuth |
| `AutoMapper` | 13.x | Object mapping |
| `AutoMapper.Extensions.Microsoft.DependencyInjection` | 12.x | DI integration |
| `FluentValidation.AspNetCore` | 11.x | Request validation |
| `Hangfire.AspNetCore` | 1.8.x | Background jobs |
| `Hangfire.SqlServer` | 1.8.x | Hangfire SQL storage |
| `Microsoft.AspNetCore.SignalR` | 8.x | Real-time |
| `Serilog.AspNetCore` | 8.x | Structured logging |
| `Serilog.Sinks.Console` | 5.x | Console sink |
| `Serilog.Sinks.File` | 5.x | File sink |
| `Swashbuckle.AspNetCore` | 6.x | Swagger/OpenAPI |
| `Microsoft.AspNetCore.OpenApi` | 8.x | OpenAPI support |

---

## 13. Controller Template

```csharp
// API/Controllers/ProductsController.cs
[ApiController]
[Route("api/[controller]")]
public class ProductsController(IProductService productService) : ControllerBase
{
    [HttpGet]
    public async Task<IActionResult> GetAll([FromQuery] ProductFilterRequest filter)
        => Ok(await productService.GetAllAsync(filter));

    [HttpGet("{id:guid}")]
    public async Task<IActionResult> GetById(Guid id)
        => Ok(await productService.GetByIdAsync(id));

    [HttpPost]
    [Authorize(Roles = "StoreOwner,Admin")]
    public async Task<IActionResult> Create([FromBody] CreateProductRequest request)
    {
        var result = await productService.CreateAsync(request, GetCurrentStoreId());
        return CreatedAtAction(nameof(GetById), new { id = result.Id }, result);
    }

    [HttpPut("{id:guid}")]
    [Authorize(Roles = "StoreOwner,Admin")]
    public async Task<IActionResult> Update(Guid id, [FromBody] UpdateProductRequest request)
        => Ok(await productService.UpdateAsync(id, request));

    [HttpDelete("{id:guid}")]
    [Authorize(Roles = "StoreOwner,Admin")]
    public async Task<IActionResult> Delete(Guid id)
    {
        await productService.DeleteAsync(id);
        return NoContent();
    }

    private Guid GetCurrentStoreId()
        => Guid.Parse(User.FindFirstValue("storeId")
            ?? throw new UnauthorizedAccessException("No store associated."));
}
```

---

## 14. Render Deployment Rules

```dockerfile
# Dockerfile (in solution root)
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 8080

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["src/SaveFood.API/SaveFood.API.csproj", "src/SaveFood.API/"]
COPY ["src/SaveFood.Application/SaveFood.Application.csproj", "src/SaveFood.Application/"]
COPY ["src/SaveFood.Domain/SaveFood.Domain.csproj", "src/SaveFood.Domain/"]
COPY ["src/SaveFood.Infrastructure/SaveFood.Infrastructure.csproj", "src/SaveFood.Infrastructure/"]
RUN dotnet restore "src/SaveFood.API/SaveFood.API.csproj"
COPY . .
WORKDIR "/src/src/SaveFood.API"
RUN dotnet build -c Release -o /app/build

FROM build AS publish
RUN dotnet publish -c Release -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "SaveFood.API.dll"]
```

**Render env vars** (set in Render dashboard, never in code):
`ConnectionStrings__DefaultConnection`, `Jwt__Key`, `VNPay__HashSecret`, `Google__ClientSecret`, `Facebook__AppSecret`

Apply EF migrations on startup:
```csharp
// Program.cs — before app.Run()
using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<SaveFoodDbContext>();
    await db.Database.MigrateAsync();
}
```

---

## 15. DI Registration Pattern

All registrations in `API/Extensions/`:
```csharp
// ServiceCollectionExtensions.cs
public static IServiceCollection AddApplicationServices(this IServiceCollection services)
{
    services.AddScoped<IProductRepository, ProductRepository>();
    services.AddScoped<IOrderRepository, OrderRepository>();
    services.AddScoped<IStoreRepository, StoreRepository>();
    services.AddScoped<IProductService, ProductService>();
    services.AddScoped<IOrderService, OrderService>();
    services.AddScoped<IVNPayService, VNPayService>();
    services.AddAutoMapper(typeof(ProductProfile).Assembly);
    return services;
}
```

---

## 16. Coding Conventions

- **Naming**: PascalCase classes/methods, camelCase locals, `_camelCase` private fields
- **No magic strings**: use `nameof()`, constants, or enums
- **Cancellation tokens**: every async repository method accepts `CancellationToken ct = default`
- **Pagination**: all list endpoints return `PaginatedList<T>` (PageNumber, PageSize, TotalCount, Items)
- **Logging**: Serilog only; use `ILogger<T>` injection; log at `Information` for business events, `Error` for exceptions
- **Response envelope**: all API responses follow `{ data, message, success }` via `ApiResponse<T>` wrapper

---

> 📁 For deep-dive samples, see:
> - `references/architecture.md` — full annotated project tree
> - `references/auth-setup.md` — complete JWT + OAuth wiring in Program.cs
> - `references/code-samples.md` — full Service/Repository implementations
> - `references/payment.md` — VNPay HMAC signing implementation
