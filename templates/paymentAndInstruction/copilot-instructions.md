# SaveFood — GitHub Copilot Custom Instructions
# Place this content in .github/copilot-instructions.md

## Project Context
SaveFood is an ASP.NET Core 8 web marketplace for near-expiry food items.
Stack: C# 12, SQL Server, Entity Framework Core 8, ASP.NET Core Identity, JWT, deployed on Render.

## Architecture Rules
Always follow N-Tier + Repository Pattern:
- Flow: Controller → Service (interface) → Repository (interface) → DbContext
- Never put business logic in controllers
- Never expose EF entities directly from API — use DTOs always
- Always create IInterface before implementation for every Service and Repository
- Register all dependencies as Scoped in ServiceCollectionExtensions.cs

## Code Style
- Use C# 12 features: primary constructors, records for DTOs, pattern matching
- All methods are async/await — never .Result or .Wait()
- Every async method accepts CancellationToken ct = default
- PascalCase for classes/methods/properties, _camelCase for private fields
- Use nameof() not magic strings

## DTOs & Mapping
- Request DTOs use suffix "Request", Response DTOs use suffix "Response"
- AutoMapper handles all Entity↔DTO mapping — never manual mapping
- FluentValidation validates all request DTOs — never DataAnnotations

## Data Access
- Soft delete only: set IsDeleted = true, never DELETE rows
- Global query filters on DbContext filter out IsDeleted entities
- All entities inherit BaseEntity (Id: Guid, CreatedAt, UpdatedAt, IsDeleted)
- Use .AsNoTracking() for read-only queries

## Auth
- ASP.NET Core Identity + JWT Bearer + Google/Facebook OAuth
- JWT claims include: userId, email, fullName, storeId, roles
- [Authorize(Roles = "Admin,StoreOwner")] for management endpoints
- [Authorize(Roles = "Customer")] for order endpoints

## Error Handling
- All exceptions bubble to GlobalExceptionMiddleware — never swallow in controllers
- Throw ProductExpiredException when purchasing expired products
- Throw InsufficientStockException when stock is insufficient
- Throw NotFoundException for missing resources

## Background Jobs
- Hangfire recurring job every 30 min to expire products past ExpiryDate
- After expiry: set Status = ProductStatus.Expired, notify via SignalR hub

## API Response Format
Always wrap responses: ApiResponse<T> { bool Success, string Message, T? Data }
List endpoints always return PaginatedList<T>

## NuGet
Always use: AutoMapper, FluentValidation.AspNetCore, Hangfire.AspNetCore, Hangfire.SqlServer, Serilog.AspNetCore, Swashbuckle.AspNetCore

## Deployment
Dockerfile multi-stage build targeting aspnet:8.0, EXPOSE 8080.
Run db.Database.MigrateAsync() on startup. Secrets via environment variables only.
