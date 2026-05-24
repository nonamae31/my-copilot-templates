# Architecture Reference — SaveFood

## Full Solution Tree (annotated)

```
SaveFood/
├── SaveFood.sln
│
├── src/
│   │
│   ├── SaveFood.API/
│   │   ├── Controllers/
│   │   │   ├── AuthController.cs        ← register, login, refresh, OAuth
│   │   │   ├── ProductsController.cs    ← CRUD, search, filter
│   │   │   ├── OrdersController.cs      ← place, cancel, history
│   │   │   ├── StoresController.cs      ← store management (StoreOwner/Admin)
│   │   │   ├── PaymentController.cs     ← VNPay create URL + callback
│   │   │   └── AdminController.cs       ← user management, reports
│   │   │
│   │   ├── Middleware/
│   │   │   └── GlobalExceptionMiddleware.cs
│   │   │
│   │   ├── Extensions/
│   │   │   ├── ServiceCollectionExtensions.cs   ← all DI registrations
│   │   │   ├── AuthExtensions.cs                ← JWT + OAuth setup
│   │   │   ├── SwaggerExtensions.cs             ← Swagger with JWT support
│   │   │   └── HangfireExtensions.cs            ← job schedules
│   │   │
│   │   ├── Program.cs
│   │   ├── appsettings.json
│   │   ├── appsettings.Development.json
│   │   └── SaveFood.API.csproj
│   │
│   ├── SaveFood.Application/
│   │   ├── DTOs/
│   │   │   ├── Auth/
│   │   │   │   ├── RegisterRequest.cs
│   │   │   │   ├── LoginRequest.cs
│   │   │   │   ├── RefreshTokenRequest.cs
│   │   │   │   └── AuthResponse.cs          ← { accessToken, refreshToken, user }
│   │   │   ├── Product/
│   │   │   │   ├── CreateProductRequest.cs
│   │   │   │   ├── UpdateProductRequest.cs
│   │   │   │   ├── ProductFilterRequest.cs  ← pageNumber, pageSize, storeId, status
│   │   │   │   └── ProductResponse.cs
│   │   │   ├── Order/
│   │   │   │   ├── CreateOrderRequest.cs    ← Items: List<OrderItemRequest>
│   │   │   │   ├── OrderResponse.cs
│   │   │   │   └── UpdateOrderStatusRequest.cs
│   │   │   └── Store/
│   │   │       ├── CreateStoreRequest.cs
│   │   │       └── StoreResponse.cs
│   │   │
│   │   ├── Interfaces/
│   │   │   ├── Repositories/
│   │   │   │   ├── IBaseRepository.cs
│   │   │   │   ├── IProductRepository.cs
│   │   │   │   ├── IOrderRepository.cs
│   │   │   │   └── IStoreRepository.cs
│   │   │   └── Services/
│   │   │       ├── IAuthService.cs
│   │   │       ├── IProductService.cs
│   │   │       ├── IOrderService.cs
│   │   │       ├── IStoreService.cs
│   │   │       └── IVNPayService.cs
│   │   │
│   │   ├── Services/
│   │   │   ├── AuthService.cs
│   │   │   ├── ProductService.cs
│   │   │   ├── OrderService.cs
│   │   │   ├── StoreService.cs
│   │   │   └── JwtTokenService.cs
│   │   │
│   │   ├── Validators/
│   │   │   ├── RegisterRequestValidator.cs
│   │   │   ├── LoginRequestValidator.cs
│   │   │   ├── CreateProductRequestValidator.cs
│   │   │   ├── UpdateProductRequestValidator.cs
│   │   │   └── CreateOrderRequestValidator.cs
│   │   │
│   │   ├── Mappings/
│   │   │   ├── ProductProfile.cs
│   │   │   ├── OrderProfile.cs
│   │   │   ├── StoreProfile.cs
│   │   │   └── AuthProfile.cs
│   │   │
│   │   └── Common/
│   │       ├── ApiResponse.cs
│   │       ├── PaginatedList.cs
│   │       ├── Result.cs                    ← Result<T> for service return
│   │       └── Exceptions/
│   │           ├── NotFoundException.cs
│   │           ├── ProductExpiredException.cs
│   │           ├── InsufficientStockException.cs
│   │           └── ValidationException.cs
│   │
│   ├── SaveFood.Domain/
│   │   ├── Entities/
│   │   │   ├── BaseEntity.cs
│   │   │   ├── ApplicationUser.cs
│   │   │   ├── Store.cs
│   │   │   ├── Product.cs
│   │   │   ├── Order.cs
│   │   │   ├── OrderItem.cs
│   │   │   └── RefreshToken.cs
│   │   ├── Enums/
│   │   │   ├── ProductStatus.cs
│   │   │   ├── OrderStatus.cs
│   │   │   └── UserRole.cs
│   │   └── Constants/
│   │       └── AppConstants.cs              ← role names, pagination defaults
│   │
│   └── SaveFood.Infrastructure/
│       ├── Data/
│       │   ├── SaveFoodDbContext.cs
│       │   ├── Configurations/
│       │   │   ├── ProductConfiguration.cs  ← decimal precision, indexes
│       │   │   ├── OrderConfiguration.cs
│       │   │   └── StoreConfiguration.cs
│       │   └── Migrations/                  ← EF generated
│       │
│       ├── Repositories/
│       │   ├── BaseRepository.cs
│       │   ├── ProductRepository.cs
│       │   ├── OrderRepository.cs
│       │   └── StoreRepository.cs
│       │
│       ├── Services/
│       │   ├── VNPayService.cs
│       │   ├── NotificationHub.cs           ← SignalR hub
│       │   └── EmailService.cs              ← MailKit / SendGrid
│       │
│       └── Jobs/
│           ├── ProductExpiryJob.cs          ← Hangfire recurring
│           └── LowStockNotificationJob.cs
│
├── tests/
│   └── SaveFood.Tests/
│       ├── Services/
│       │   └── ProductServiceTests.cs
│       └── SaveFood.Tests.csproj
│
├── Dockerfile
├── docker-compose.yml                       ← local dev with SQL Server
└── .gitignore
```

## Project References

```
SaveFood.API          → SaveFood.Application, SaveFood.Infrastructure
SaveFood.Application  → SaveFood.Domain
SaveFood.Infrastructure → SaveFood.Application, SaveFood.Domain
SaveFood.Tests        → SaveFood.Application, SaveFood.Infrastructure
```

## EF Entity Configuration Example

```csharp
// Infrastructure/Data/Configurations/ProductConfiguration.cs
public class ProductConfiguration : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> builder)
    {
        builder.HasKey(p => p.Id);
        builder.Property(p => p.Name).IsRequired().HasMaxLength(200);
        builder.Property(p => p.OriginalPrice).HasPrecision(18, 2);
        builder.Property(p => p.DiscountedPrice).HasPrecision(18, 2);
        builder.HasIndex(p => p.ExpiryDate);          // for expiry job query
        builder.HasIndex(p => new { p.StoreId, p.Status });
        builder.HasOne(p => p.Store)
               .WithMany(s => s.Products)
               .HasForeignKey(p => p.StoreId)
               .OnDelete(DeleteBehavior.Restrict);
    }
}
```
