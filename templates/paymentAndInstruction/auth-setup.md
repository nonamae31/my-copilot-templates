# Auth Setup Reference — SaveFood

## Program.cs — Full Auth Wiring

```csharp
// Identity
builder.Services.AddIdentity<ApplicationUser, IdentityRole<Guid>>(options =>
{
    options.Password.RequireDigit = true;
    options.Password.RequiredLength = 8;
    options.Password.RequireNonAlphanumeric = false;
    options.User.RequireUniqueEmail = true;
    options.SignIn.RequireConfirmedEmail = false; // set true in production
})
.AddEntityFrameworkStores<SaveFoodDbContext>()
.AddDefaultTokenProviders();

// JWT
var jwtSettings = builder.Configuration.GetSection("Jwt");
builder.Services.AddAuthentication(options =>
{
    options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
})
.AddJwtBearer(options =>
{
    options.TokenValidationParameters = new TokenValidationParameters
    {
        ValidateIssuer = true,
        ValidateAudience = true,
        ValidateLifetime = true,
        ValidateIssuerSigningKey = true,
        ValidIssuer = jwtSettings["Issuer"],
        ValidAudience = jwtSettings["Audience"],
        IssuerSigningKey = new SymmetricSecurityKey(
            Encoding.UTF8.GetBytes(jwtSettings["Key"]!)),
        ClockSkew = TimeSpan.Zero // no tolerance for expired tokens
    };
    // Support JWT from SignalR query string
    options.Events = new JwtBearerEvents
    {
        OnMessageReceived = ctx =>
        {
            var accessToken = ctx.Request.Query["access_token"];
            var path = ctx.HttpContext.Request.Path;
            if (!string.IsNullOrEmpty(accessToken) && path.StartsWithSegments("/hubs"))
                ctx.Token = accessToken;
            return Task.CompletedTask;
        }
    };
})
.AddGoogle(options =>
{
    options.ClientId = builder.Configuration["Google:ClientId"]!;
    options.ClientSecret = builder.Configuration["Google:ClientSecret"]!;
    options.CallbackPath = "/api/auth/google-callback";
})
.AddFacebook(options =>
{
    options.AppId = builder.Configuration["Facebook:AppId"]!;
    options.AppSecret = builder.Configuration["Facebook:AppSecret"]!;
    options.CallbackPath = "/api/auth/facebook-callback";
});

builder.Services.AddAuthorization();
```

## JwtTokenService

```csharp
// Application/Services/JwtTokenService.cs
public class JwtTokenService(IConfiguration config)
{
    public string GenerateAccessToken(ApplicationUser user, IList<string> roles)
    {
        var claims = new List<Claim>
        {
            new(ClaimTypes.NameIdentifier, user.Id.ToString()),
            new(ClaimTypes.Email, user.Email!),
            new(ClaimTypes.Name, user.FullName),
            new("storeId", user.StoreId?.ToString() ?? string.Empty),
        };
        claims.AddRange(roles.Select(r => new Claim(ClaimTypes.Role, r)));

        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(config["Jwt:Key"]!));
        var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);
        var expiry = DateTime.UtcNow.AddMinutes(int.Parse(config["Jwt:ExpiryMinutes"]!));

        var token = new JwtSecurityToken(
            issuer: config["Jwt:Issuer"],
            audience: config["Jwt:Audience"],
            claims: claims,
            expires: expiry,
            signingCredentials: creds);

        return new JwtSecurityTokenHandler().WriteToken(token);
    }

    public RefreshToken GenerateRefreshToken(Guid userId)
        => new()
        {
            Id = Guid.NewGuid(),
            UserId = userId,
            Token = Convert.ToBase64String(RandomNumberGenerator.GetBytes(64)),
            ExpiresAt = DateTime.UtcNow.AddDays(int.Parse(config["Jwt:RefreshTokenExpiryDays"]!)),
            CreatedAt = DateTime.UtcNow
        };
}
```

## AuthController Skeleton

```csharp
[ApiController]
[Route("api/auth")]
public class AuthController(IAuthService authService) : ControllerBase
{
    [HttpPost("register")]
    public async Task<IActionResult> Register([FromBody] RegisterRequest request)
        => Ok(await authService.RegisterAsync(request));

    [HttpPost("login")]
    public async Task<IActionResult> Login([FromBody] LoginRequest request)
        => Ok(await authService.LoginAsync(request));

    [HttpPost("refresh")]
    public async Task<IActionResult> Refresh([FromBody] RefreshTokenRequest request)
        => Ok(await authService.RefreshTokenAsync(request));

    [HttpPost("revoke")]
    [Authorize]
    public async Task<IActionResult> Revoke()
    {
        await authService.RevokeTokenAsync(User.FindFirstValue(ClaimTypes.NameIdentifier)!);
        return NoContent();
    }

    [HttpGet("google-login")]
    public IActionResult GoogleLogin()
        => Challenge(new AuthenticationProperties { RedirectUri = "/api/auth/oauth-callback" }, GoogleDefaults.AuthenticationScheme);

    [HttpGet("facebook-login")]
    public IActionResult FacebookLogin()
        => Challenge(new AuthenticationProperties { RedirectUri = "/api/auth/oauth-callback" }, FacebookDefaults.AuthenticationScheme);

    [HttpGet("oauth-callback")]
    public async Task<IActionResult> OAuthCallback()
        => Ok(await authService.HandleOAuthCallbackAsync(HttpContext));
}
```
