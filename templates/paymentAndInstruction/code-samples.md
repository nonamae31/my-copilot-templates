# Code Samples Reference — SaveFood

## BaseEntity

```csharp
// Domain/Entities/BaseEntity.cs
public abstract class BaseEntity
{
    public Guid Id { get; set; } = Guid.NewGuid();
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public DateTime UpdatedAt { get; set; } = DateTime.UtcNow;
    public bool IsDeleted { get; set; } = false;
}
```

## SaveFoodDbContext

```csharp
// Infrastructure/Data/SaveFoodDbContext.cs
public class SaveFoodDbContext(DbContextOptions<SaveFoodDbContext> options)
    : IdentityDbContext<ApplicationUser, IdentityRole<Guid>, Guid>(options)
{
    public DbSet<Store> Stores => Set<Store>();
    public DbSet<Product> Products => Set<Product>();
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<OrderItem> OrderItems => Set<OrderItem>();
    public DbSet<RefreshToken> RefreshTokens => Set<RefreshToken>();

    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);
        builder.ApplyConfigurationsFromAssembly(typeof(SaveFoodDbContext).Assembly);
        // Global soft-delete filter
        builder.Entity<Product>().HasQueryFilter(p => !p.IsDeleted);
        builder.Entity<Order>().HasQueryFilter(o => !o.IsDeleted);
        builder.Entity<Store>().HasQueryFilter(s => !s.IsDeleted);
    }

    public override async Task<int> SaveChangesAsync(CancellationToken ct = default)
    {
        foreach (var entry in ChangeTracker.Entries<BaseEntity>()
            .Where(e => e.State == EntityState.Modified))
            entry.Entity.UpdatedAt = DateTime.UtcNow;
        return await base.SaveChangesAsync(ct);
    }
}
```

## ProductService (full example)

```csharp
// Application/Services/ProductService.cs
public class ProductService(
    IProductRepository productRepo,
    IMapper mapper,
    IHubContext<NotificationHub> hub,
    ILogger<ProductService> logger) : IProductService
{
    public async Task<ProductResponse> CreateAsync(CreateProductRequest request, Guid storeId, CancellationToken ct = default)
    {
        var product = mapper.Map<Product>(request);
        product.StoreId = storeId;
        await productRepo.AddAsync(product, ct);
        await productRepo.SaveChangesAsync(ct);
        logger.LogInformation("Product {ProductId} created for store {StoreId}", product.Id, storeId);
        return mapper.Map<ProductResponse>(product);
    }

    public async Task<ProductResponse> UpdateAsync(Guid id, UpdateProductRequest request, CancellationToken ct = default)
    {
        var product = await productRepo.GetByIdAsync(id, ct)
            ?? throw new NotFoundException($"Product {id} not found.");
        mapper.Map(request, product);
        productRepo.Update(product);
        await productRepo.SaveChangesAsync(ct);
        return mapper.Map<ProductResponse>(product);
    }

    public async Task DeleteAsync(Guid id, CancellationToken ct = default)
    {
        var product = await productRepo.GetByIdAsync(id, ct)
            ?? throw new NotFoundException($"Product {id} not found.");
        product.IsDeleted = true;
        productRepo.Update(product);
        await productRepo.SaveChangesAsync(ct);
    }

    public async Task<PaginatedList<ProductResponse>> GetAllAsync(ProductFilterRequest filter, CancellationToken ct = default)
    {
        var products = await productRepo.GetFilteredAsync(filter, ct);
        var mapped = mapper.Map<IEnumerable<ProductResponse>>(products.Items);
        return new PaginatedList<ProductResponse>(mapped, products.TotalCount, filter.PageNumber, filter.PageSize);
    }
}
```

## OrderService — business logic for placing orders

```csharp
public async Task<OrderResponse> PlaceOrderAsync(CreateOrderRequest request, Guid customerId, CancellationToken ct = default)
{
    // 1. Validate all products
    var order = new Order { CustomerId = customerId, Status = OrderStatus.Pending };
    decimal total = 0;

    foreach (var item in request.Items)
    {
        var product = await productRepo.GetByIdAsync(item.ProductId, ct)
            ?? throw new NotFoundException($"Product {item.ProductId} not found.");

        if (product.ExpiryDate <= DateTime.UtcNow)
            throw new ProductExpiredException($"'{product.Name}' has expired and cannot be purchased.");

        if (product.StockQuantity < item.Quantity)
            throw new InsufficientStockException($"Only {product.StockQuantity} units of '{product.Name}' available.");

        product.StockQuantity -= item.Quantity;
        if (product.StockQuantity == 0) product.Status = ProductStatus.SoldOut;
        productRepo.Update(product);

        order.OrderItems.Add(new OrderItem
        {
            ProductId = product.Id,
            Quantity = item.Quantity,
            UnitPrice = product.DiscountedPrice
        });
        total += product.DiscountedPrice * item.Quantity;
    }

    order.TotalAmount = total;
    await orderRepo.AddAsync(order, ct);
    await orderRepo.SaveChangesAsync(ct);

    // 2. Notify store owner in real-time
    await hub.Clients.Group($"store-{order.OrderItems.First().Product.StoreId}")
        .SendAsync("OrderPlaced", new { order.Id, order.TotalAmount, ItemCount = order.OrderItems.Count });

    return mapper.Map<OrderResponse>(order);
}
```

## ApiResponse<T> Wrapper

```csharp
// Application/Common/ApiResponse.cs
public class ApiResponse<T>
{
    public bool Success { get; init; }
    public string Message { get; init; } = string.Empty;
    public T? Data { get; init; }

    public static ApiResponse<T> Ok(T data, string message = "Success")
        => new() { Success = true, Data = data, Message = message };

    public static ApiResponse<T> Fail(string message)
        => new() { Success = false, Message = message };
}
```

## PaginatedList<T>

```csharp
public class PaginatedList<T>
{
    public IEnumerable<T> Items { get; }
    public int TotalCount { get; }
    public int PageNumber { get; }
    public int PageSize { get; }
    public int TotalPages => (int)Math.Ceiling(TotalCount / (double)PageSize);
    public bool HasPreviousPage => PageNumber > 1;
    public bool HasNextPage => PageNumber < TotalPages;

    public PaginatedList(IEnumerable<T> items, int totalCount, int pageNumber, int pageSize)
        => (Items, TotalCount, PageNumber, PageSize) = (items, totalCount, pageNumber, pageSize);
}
```

## BaseRepository<T> implementation

```csharp
// Infrastructure/Repositories/BaseRepository.cs
public abstract class BaseRepository<T>(SaveFoodDbContext ctx) : IBaseRepository<T>
    where T : BaseEntity
{
    protected readonly SaveFoodDbContext _ctx = ctx;
    private readonly DbSet<T> _set = ctx.Set<T>();

    public async Task<T?> GetByIdAsync(Guid id, CancellationToken ct = default)
        => await _set.FirstOrDefaultAsync(e => e.Id == id && !e.IsDeleted, ct);

    public async Task<IEnumerable<T>> GetAllAsync(CancellationToken ct = default)
        => await _set.Where(e => !e.IsDeleted).AsNoTracking().ToListAsync(ct);

    public async Task AddAsync(T entity, CancellationToken ct = default)
        => await _set.AddAsync(entity, ct);

    public void Update(T entity) => _set.Update(entity);

    public void Delete(T entity)
    {
        entity.IsDeleted = true;
        _set.Update(entity);
    }

    public async Task<int> SaveChangesAsync(CancellationToken ct = default)
        => await _ctx.SaveChangesAsync(ct);
}
```
