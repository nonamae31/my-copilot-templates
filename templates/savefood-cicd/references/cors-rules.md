# CORS Configuration — SaveFood Backend

## Full Program.cs Pattern

```csharp
// ═══════════════════════════════════════════════════════════════
// SaveFood API — Program.cs (CORS section)
// ═══════════════════════════════════════════════════════════════

var builder = WebApplication.CreateBuilder(args);

// ── CORS ────────────────────────────────────────────────────────
const string CorsPolicyName = "SaveFoodCorsPolicy";

var allowedOrigins = (builder.Configuration["ALLOWED_ORIGINS"] ?? "")
    .Split(',', StringSplitOptions.RemoveEmptyEntries | StringSplitOptions.TrimEntries);

builder.Services.AddCors(options =>
{
    options.AddPolicy(CorsPolicyName, policy =>
    {
        if (builder.Environment.IsDevelopment())
        {
            // Development: allow local Vite/CRA dev servers
            policy.WithOrigins(
                "http://localhost:3000",   // CRA default
                "http://localhost:5173"    // Vite default
            );
        }
        else
        {
            // Production: only allow explicitly configured origins
            if (allowedOrigins.Length == 0)
                throw new InvalidOperationException(
                    "ALLOWED_ORIGINS env var must be set in production");

            policy.WithOrigins(allowedOrigins);
        }

        policy
            .WithMethods("GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS")
            .WithHeaders(
                "Content-Type",
                "Authorization",
                "X-Requested-With",
                "Accept",
                "Origin"
            )
            .AllowCredentials()   // Required if using HttpOnly cookies for auth
            .SetPreflightMaxAge(TimeSpan.FromMinutes(10)); // Cache preflight for 10 min
    });
});

// ... other services ...

var app = builder.Build();

// ── Middleware Order (CRITICAL) ──────────────────────────────────
// CORS MUST be before UseAuthorization and UseAuthentication
app.UseRouting();
app.UseCors(CorsPolicyName);      // ← HERE, after UseRouting
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

---

## Middleware Order — Why It Matters

```
Request Pipeline (correct order):
1. UseRouting          → Matches route
2. UseCors            → Adds CORS headers (must know the route)
3. UseAuthentication   → Validates JWT token
4. UseAuthorization    → Checks permissions
5. MapControllers      → Executes controller action
```

**Wrong order symptoms**:
- CORS before UseRouting → Named policy won't work correctly
- CORS after UseAuthorization → Preflight OPTIONS requests get 401 before CORS headers are added

---

## Render Environment Variable

Set in Render Dashboard:
```
ALLOWED_ORIGINS=https://savefood.vercel.app,https://www.savefood.vn
```

Multiple origins separated by commas, no spaces, no trailing slash.

---

## Common CORS Errors & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `No 'Access-Control-Allow-Origin' header` | CORS middleware not applied or wrong order | Check middleware order, ensure `UseCors` is called |
| `Origin 'X' not allowed` | Origin not in `ALLOWED_ORIGINS` | Add exact origin to Render env var |
| `Request header not allowed` | Missing header in `WithHeaders` | Add header name to `WithHeaders(...)` |
| `Credentials flag is true but no cookies` | `AllowCredentials()` without `WithOrigins()` | Never use `AllowAnyOrigin()` with `AllowCredentials()` |
| `CORS on preflight fail (OPTIONS 404)` | OPTIONS not handled | ASP.NET Core handles OPTIONS automatically if `UseCors` is in pipeline |

---

## AllowCredentials() — When to Use

| Auth Method | AllowCredentials? |
|-------------|-------------------|
| JWT in `Authorization: Bearer` header | Optional (but include for flexibility) |
| HttpOnly Cookie (refresh tokens) | **Required** |
| Session cookies | **Required** |

> ⚠️ `AllowCredentials()` is **incompatible** with `AllowAnyOrigin()`. Always use `WithOrigins([...])`.

---

## Testing CORS Locally

```bash
# Test preflight from the browser's perspective:
curl -X OPTIONS http://localhost:8080/api/products \
  -H "Origin: http://localhost:5173" \
  -H "Access-Control-Request-Method: GET" \
  -H "Access-Control-Request-Headers: Authorization" \
  -v

# Expected response headers:
# access-control-allow-origin: http://localhost:5173
# access-control-allow-methods: GET,POST,PUT,DELETE,PATCH,OPTIONS
# access-control-allow-headers: Content-Type,Authorization,...
# access-control-allow-credentials: true
```
