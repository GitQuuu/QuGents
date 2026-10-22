# Auth templates (Identity + JWT + refresh tokens)

Skeletons for the Auth module in the N-layer layout from `dotnet-n-layer`. Types such as `ServiceResult<T>`, `IUnitOfWork`, `IBaseRepository<T>` and `IResponseService` come from that skill. **Not compiled in the sandbox (no NuGet access): run `dotnet build` and the auth tests the first time you use them.**

Packages:

| Project | Packages |
|---|---|
| DataAccess | `Microsoft.AspNetCore.Identity.EntityFrameworkCore`, `Microsoft.EntityFrameworkCore.SqlServer` |
| Services | `Microsoft.IdentityModel.JsonWebTokens` (Identity types come via DataAccess) |
| Api | `Microsoft.AspNetCore.Authentication.JwtBearer` |

## Contents

1. Entities and DbContext
2. Refresh-token repository
3. Identity registration (DataAccess)
4. `JwtOptions`, `Roles`, `AuthTokens`
5. `TokenService`
6. `AuthService`
7. `AuthBll`
8. Thin `AuthController` with the refresh cookie
9. Api: `AddJwtAuthentication`, `GetUserId()`, role seeding, rate limiting
10. Configuration and secrets

---

## 1. Entities and DbContext

```csharp
// <App>.DataAccess/Entities/ApplicationUser.cs
using Microsoft.AspNetCore.Identity;

public sealed class ApplicationUser : IdentityUser
{
    public string DisplayName { get; set; } = "";
}

// <App>.DataAccess/Entities/RefreshToken.cs
public sealed class RefreshToken
{
    public Guid Id { get; set; }
    public string UserId { get; set; } = "";
    public string TokenHash { get; set; } = "";          // SHA-256 hex, never the raw token
    public Guid FamilyId { get; set; }                     // all rotations of one login
    public DateTimeOffset CreatedAt { get; set; }
    public DateTimeOffset ExpiresAt { get; set; }
    public DateTimeOffset? RevokedAt { get; set; }
    public string? ReplacedByTokenHash { get; set; }

    public bool IsActive(DateTimeOffset now) => RevokedAt is null && ExpiresAt > now;
}

// <App>.DataAccess/AppDbContext.cs
using Microsoft.AspNetCore.Identity.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore;

public sealed class AppDbContext(DbContextOptions<AppDbContext> options) : IdentityDbContext<ApplicationUser>(options)
{
    public DbSet<RefreshToken> RefreshTokens => Set<RefreshToken>();

    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder); // required: maps the Identity tables

        builder.Entity<RefreshToken>(e =>
        {
            e.HasIndex(t => t.TokenHash).IsUnique();
            e.HasIndex(t => new { t.UserId, t.FamilyId });
            e.Property(t => t.TokenHash).HasMaxLength(64);
            e.HasOne<ApplicationUser>().WithMany().HasForeignKey(t => t.UserId).OnDelete(DeleteBehavior.Cascade);
        });
    }
}
```

## 2. Refresh-token repository

```csharp
// <App>.DataAccess/Repositories/Auth/IRefreshTokenRepository.cs
using Microsoft.EntityFrameworkCore;

public interface IRefreshTokenRepository : IBaseRepository<RefreshToken>
{
    Task<RefreshToken?> GetByHashAsync(string tokenHash, CancellationToken ct = default);   // tracked
    Task<IReadOnlyList<RefreshToken>> GetActiveByFamilyAsync(Guid familyId, DateTimeOffset now, CancellationToken ct = default);
    Task<IReadOnlyList<RefreshToken>> GetActiveByUserAsync(string userId, DateTimeOffset now, CancellationToken ct = default);
}

public sealed class RefreshTokenRepository(AppDbContext db) : BaseRepository<RefreshToken>(db), IRefreshTokenRepository
{
    public Task<RefreshToken?> GetByHashAsync(string tokenHash, CancellationToken ct = default) =>
        Set.FirstOrDefaultAsync(t => t.TokenHash == tokenHash, ct);

    public async Task<IReadOnlyList<RefreshToken>> GetActiveByFamilyAsync(Guid familyId, DateTimeOffset now, CancellationToken ct = default) =>
        await Set.Where(t => t.FamilyId == familyId && t.RevokedAt == null && t.ExpiresAt > now).ToListAsync(ct);

    public async Task<IReadOnlyList<RefreshToken>> GetActiveByUserAsync(string userId, DateTimeOffset now, CancellationToken ct = default) =>
        await Set.Where(t => t.UserId == userId && t.RevokedAt == null && t.ExpiresAt > now).ToListAsync(ct);
}
```

Add `IRefreshTokenRepository RefreshTokens { get; }` to `IUnitOfWork`.

## 3. Identity registration (DataAccess)

```csharp
// <App>.DataAccess/DependencyInjection.cs
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;

public static class DataAccessRegistration
{
    public static IServiceCollection AddDataAccess(this IServiceCollection services, IConfiguration config)
    {
        services.AddDbContext<AppDbContext>(o => o.UseSqlServer(config.GetConnectionString("Default")));

        services.AddIdentityCore<ApplicationUser>(o =>
            {
                o.User.RequireUniqueEmail = true;
                o.Password.RequiredLength = 12;
                o.Lockout.MaxFailedAccessAttempts = 5;
                o.Lockout.DefaultLockoutTimeSpan = TimeSpan.FromMinutes(15);
            })
            .AddRoles<IdentityRole>()
            .AddEntityFrameworkStores<AppDbContext>();

        services.AddScoped<IUnitOfWork, UnitOfWork>();
        services.AddScoped<IRefreshTokenRepository, RefreshTokenRepository>();
        // ...other repositories
        return services;
    }
}
```

## 4. `JwtOptions`, `Roles`, `AuthTokens`

```csharp
// <App>.Services/Modules/Auth/JwtOptions.cs
using System.ComponentModel.DataAnnotations;

public sealed class JwtOptions
{
    public const string Section = "Jwt";

    [Required] public string Issuer { get; init; } = "";
    [Required] public string Audience { get; init; } = "";
    [Required, MinLength(32)] public string SigningKey { get; init; } = "";
    [Range(1, 120)] public int AccessTokenMinutes { get; init; } = 15;
    [Range(1, 90)] public int RefreshTokenDays { get; init; } = 14;
}

// <App>.Services/Modules/Auth/Roles.cs
public static class Roles
{
    public const string Admin = "Admin";
    public const string User = "User";
    public static readonly string[] All = [Admin, User];
}

// <App>.Services/Modules/Auth/AuthTokens.cs
public sealed record AuthTokens(string AccessToken, DateTimeOffset AccessTokenExpiresAt, string RefreshToken, DateTimeOffset RefreshTokenExpiresAt);
```

## 5. `TokenService`

```csharp
// <App>.Services/Modules/Auth/TokenService.cs
using System.Security.Claims;
using System.Security.Cryptography;
using System.Text;
using Microsoft.Extensions.Options;
using Microsoft.IdentityModel.JsonWebTokens;
using Microsoft.IdentityModel.Tokens;

public interface ITokenService
{
    (string Token, DateTimeOffset ExpiresAt) CreateAccessToken(ApplicationUser user, IEnumerable<string> roles);
    (string Token, string Hash) CreateRefreshToken();
    string Hash(string refreshToken);
}

public sealed class TokenService(IOptions<JwtOptions> options, TimeProvider clock) : ITokenService
{
    private readonly JwtOptions _jwt = options.Value;

    public (string Token, DateTimeOffset ExpiresAt) CreateAccessToken(ApplicationUser user, IEnumerable<string> roles)
    {
        var now = clock.GetUtcNow();
        var expires = now.AddMinutes(_jwt.AccessTokenMinutes);

        var claims = new List<Claim>
        {
            new(JwtRegisteredClaimNames.Sub, user.Id),
            new(JwtRegisteredClaimNames.Email, user.Email ?? ""),
            new(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
        };
        claims.AddRange(roles.Select(role => new Claim("role", role)));

        var descriptor = new SecurityTokenDescriptor
        {
            Subject = new ClaimsIdentity(claims),
            Issuer = _jwt.Issuer,
            Audience = _jwt.Audience,
            IssuedAt = now.UtcDateTime,
            NotBefore = now.UtcDateTime,
            Expires = expires.UtcDateTime,
            SigningCredentials = new SigningCredentials(
                new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_jwt.SigningKey)), SecurityAlgorithms.HmacSha256),
        };

        return (new JsonWebTokenHandler().CreateToken(descriptor), expires);
    }

    public (string Token, string Hash) CreateRefreshToken()
    {
        var token = Convert.ToBase64String(RandomNumberGenerator.GetBytes(64));
        return (token, Hash(token));
    }

    public string Hash(string refreshToken) =>
        Convert.ToHexString(SHA256.HashData(Encoding.UTF8.GetBytes(refreshToken)));
}
```

## 6. `AuthService`

Domain work: credentials, token issuing, rotation. Stages changes; never commits.

```csharp
// <App>.Services/Modules/Auth/AuthService.cs (root partial: fields + ctor) and one partial per operation folder
using System.Net;
using Microsoft.AspNetCore.Identity;
using Microsoft.Extensions.Options;

public interface IAuthService
{
    Task<ServiceResult<ApplicationUser>> RegisterUserAsync(RegisterDto dto);
    Task<ServiceResult<ApplicationUser>> ValidateCredentialsAsync(string email, string password);
    Task<AuthTokens> IssueTokensAsync(ApplicationUser user, Guid familyId, CancellationToken ct);
    Task<ServiceResult<AuthTokens>> RotateAsync(string refreshToken, CancellationToken ct);
    Task RevokeFamilyAsync(string refreshToken, CancellationToken ct);
}

public sealed record RegisterDto(string Email, string Password, string DisplayName);

// Explicit fields + constructor (not a primary constructor) so every partial file can use them.
public sealed partial class AuthService : IAuthService
{
    private const string InvalidCredentials = "Invalid email or password.";

    private readonly UserManager<ApplicationUser> users;
    private readonly IUnitOfWork uow;
    private readonly ITokenService tokens;
    private readonly IOptions<JwtOptions> jwt;
    private readonly TimeProvider clock;

    public AuthService(UserManager<ApplicationUser> users, IUnitOfWork uow, ITokenService tokens, IOptions<JwtOptions> jwt, TimeProvider clock)
    {
        this.users = users;
        this.uow = uow;
        this.tokens = tokens;
        this.jwt = jwt;
        this.clock = clock;
    }

    // UserManager saves immediately (see SKILL.md gotchas). Nothing else is staged in these use cases.
    public async Task<ServiceResult<ApplicationUser>> RegisterUserAsync(RegisterDto dto)
    {
        var user = new ApplicationUser { UserName = dto.Email, Email = dto.Email, DisplayName = dto.DisplayName };
        var created = await users.CreateAsync(user, dto.Password);
        if (!created.Succeeded)
            return ServiceResult<ApplicationUser>.Fail(HttpStatusCode.BadRequest, string.Join(" ", created.Errors.Select(e => e.Description)));

        await users.AddToRoleAsync(user, Roles.User);
        return ServiceResult<ApplicationUser>.Ok(user, HttpStatusCode.Created);
    }

    public async Task<ServiceResult<ApplicationUser>> ValidateCredentialsAsync(string email, string password)
    {
        var user = await users.FindByEmailAsync(email);
        if (user is null || await users.IsLockedOutAsync(user))
            return ServiceResult<ApplicationUser>.Fail(HttpStatusCode.Unauthorized, InvalidCredentials);

        if (!await users.CheckPasswordAsync(user, password))
        {
            await users.AccessFailedAsync(user); // counts towards lockout
            return ServiceResult<ApplicationUser>.Fail(HttpStatusCode.Unauthorized, InvalidCredentials);
        }

        await users.ResetAccessFailedCountAsync(user);
        return ServiceResult<ApplicationUser>.Ok(user);
    }

    public async Task<AuthTokens> IssueTokensAsync(ApplicationUser user, Guid familyId, CancellationToken ct)
    {
        var now = clock.GetUtcNow();
        var roles = await users.GetRolesAsync(user);
        var (accessToken, accessExpires) = tokens.CreateAccessToken(user, roles);
        var (refreshToken, refreshHash) = tokens.CreateRefreshToken();
        var refreshExpires = now.AddDays(jwt.Value.RefreshTokenDays);

        await uow.RefreshTokens.AddAsync(new RefreshToken
        {
            Id = Guid.NewGuid(),
            UserId = user.Id,
            TokenHash = refreshHash,
            FamilyId = familyId,
            CreatedAt = now,
            ExpiresAt = refreshExpires,
        }, ct);

        return new AuthTokens(accessToken, accessExpires, refreshToken, refreshExpires);
    }

    public async Task<ServiceResult<AuthTokens>> RotateAsync(string refreshToken, CancellationToken ct)
    {
        var now = clock.GetUtcNow();
        var stored = await uow.RefreshTokens.GetByHashAsync(tokens.Hash(refreshToken), ct);
        if (stored is null)
            return ServiceResult<AuthTokens>.Fail(HttpStatusCode.Unauthorized, "Invalid refresh token.");

        if (stored.RevokedAt is not null)
        {
            // A rotated token was used again: treat as theft and end the whole session family.
            foreach (var active in await uow.RefreshTokens.GetActiveByFamilyAsync(stored.FamilyId, now, ct))
                active.RevokedAt = now;
            return ServiceResult<AuthTokens>.Fail(HttpStatusCode.Unauthorized, "Invalid refresh token.");
        }

        if (stored.ExpiresAt <= now)
            return ServiceResult<AuthTokens>.Fail(HttpStatusCode.Unauthorized, "Refresh token expired.");

        var user = await users.FindByIdAsync(stored.UserId);
        if (user is null || await users.IsLockedOutAsync(user))
            return ServiceResult<AuthTokens>.Fail(HttpStatusCode.Unauthorized, "Invalid refresh token.");

        var issued = await IssueTokensAsync(user, stored.FamilyId, ct);
        stored.RevokedAt = now;
        stored.ReplacedByTokenHash = tokens.Hash(issued.RefreshToken);
        return ServiceResult<AuthTokens>.Ok(issued);
    }

    public async Task RevokeFamilyAsync(string refreshToken, CancellationToken ct)
    {
        var now = clock.GetUtcNow();
        var stored = await uow.RefreshTokens.GetByHashAsync(tokens.Hash(refreshToken), ct);
        if (stored is null) return;
        foreach (var active in await uow.RefreshTokens.GetActiveByFamilyAsync(stored.FamilyId, now, ct))
            active.RevokedAt = now;
    }
}
```

(Shown as one class for readability; in the codebase split it into the root partial and one partial per operation folder.)

## 7. `AuthBll`

```csharp
// <App>.Services/Modules/Auth/AuthBll.cs (+ partials per operation)
using System.Net;

public interface IAuthBll
{
    Task<ServiceResult<AuthTokens>> RegisterAsync(RegisterDto dto, CancellationToken ct);
    Task<ServiceResult<AuthTokens>> LoginAsync(string email, string password, CancellationToken ct);
    Task<ServiceResult<AuthTokens>> RefreshAsync(string refreshToken, CancellationToken ct);
    Task LogoutAsync(string refreshToken, CancellationToken ct);
}

public sealed partial class AuthBll : IAuthBll
{
    private readonly IAuthService auth;
    private readonly IUnitOfWork uow;

    public AuthBll(IAuthService auth, IUnitOfWork uow)
    {
        this.auth = auth;
        this.uow = uow;
    }
    public async Task<ServiceResult<AuthTokens>> RegisterAsync(RegisterDto dto, CancellationToken ct)
    {
        var registered = await auth.RegisterUserAsync(dto);
        if (!registered.Success) return ServiceResult<AuthTokens>.From(registered);

        var tokens = await auth.IssueTokensAsync(registered.Data!, Guid.NewGuid(), ct);
        await uow.SaveChangesAsync(ct);
        return ServiceResult<AuthTokens>.Ok(tokens, HttpStatusCode.Created);
    }

    public async Task<ServiceResult<AuthTokens>> LoginAsync(string email, string password, CancellationToken ct)
    {
        var user = await auth.ValidateCredentialsAsync(email, password);
        if (!user.Success) return ServiceResult<AuthTokens>.From(user);

        var tokens = await auth.IssueTokensAsync(user.Data!, Guid.NewGuid(), ct);
        await uow.SaveChangesAsync(ct);
        return ServiceResult<AuthTokens>.Ok(tokens);
    }

    public async Task<ServiceResult<AuthTokens>> RefreshAsync(string refreshToken, CancellationToken ct)
    {
        var result = await auth.RotateAsync(refreshToken, ct);
        await uow.SaveChangesAsync(ct); // also on failure: persists a family revocation after token reuse
        return result;
    }

    public async Task LogoutAsync(string refreshToken, CancellationToken ct)
    {
        await auth.RevokeFamilyAsync(refreshToken, ct);
        await uow.SaveChangesAsync(ct);
    }
}
```

## 8. Thin `AuthController` with the refresh cookie

Setting and clearing the cookie is transport, so it is allowed in the controller (like the job hand-off).

```csharp
// <App>.Api/Controllers/Auth/AuthController.cs (root partial) + one folder per action
using Asp.Versioning;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.RateLimiting;

[ApiController]
[ApiVersion("1.0")]
[Route("api/v{version:apiVersion}/[controller]")]
[EnableRateLimiting("auth")]
public sealed partial class AuthController : ControllerBase
{
    private const string RefreshCookie = "refresh_token";

    private readonly IAuthBll authBll;
    private readonly IResponseService response;

    public AuthController(IAuthBll authBll, IResponseService response)
    {
        this.authBll = authBll;
        this.response = response;
    }

    private void SetRefreshCookie(AuthTokens tokens) =>
        Response.Cookies.Append(RefreshCookie, tokens.RefreshToken, new CookieOptions
        {
            HttpOnly = true,
            Secure = true,
            SameSite = SameSiteMode.Strict,
            Path = "/api/v1/auth",
            Expires = tokens.RefreshTokenExpiresAt,
        });

    private void ClearRefreshCookie() =>
        Response.Cookies.Delete(RefreshCookie, new CookieOptions { Path = "/api/v1/auth", Secure = true, SameSite = SameSiteMode.Strict });

    private IActionResult TokenResponse(ServiceResult<AuthTokens> result)
    {
        if (!result.Success)
            return response.ToActionResult(result);
        SetRefreshCookie(result.Data!);
        return Ok(new AccessTokenResponse(result.Data!.AccessToken, result.Data.AccessTokenExpiresAt));
    }
}

public sealed record AccessTokenResponse(string AccessToken, DateTimeOffset ExpiresAt);
public sealed record LoginRequest(string Email, string Password);
public sealed record RegisterRequest(string Email, string Password, string DisplayName);

// LoginAsync/AuthController.LoginAsync.cs
public sealed partial class AuthController
{
    [HttpPost("login")]
    [AllowAnonymous]
    [ProducesResponseType(typeof(AccessTokenResponse), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status401Unauthorized)]
    public async Task<IActionResult> LoginAsync([FromBody] LoginRequest request, CancellationToken ct) =>
        TokenResponse(await authBll.LoginAsync(request.Email, request.Password, ct));
}

// RefreshAsync/AuthController.RefreshAsync.cs
public sealed partial class AuthController
{
    [HttpPost("refresh")]
    [AllowAnonymous]
    [ProducesResponseType(typeof(AccessTokenResponse), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status401Unauthorized)]
    public async Task<IActionResult> RefreshAsync(CancellationToken ct)
    {
        if (!Request.Cookies.TryGetValue(RefreshCookie, out var refreshToken))
            return Unauthorized();
        var result = await authBll.RefreshAsync(refreshToken, ct);
        if (!result.Success) ClearRefreshCookie();
        return TokenResponse(result);
    }
}

// LogoutAsync/AuthController.LogoutAsync.cs
public sealed partial class AuthController
{
    [HttpPost("logout")]
    [AllowAnonymous] // works with an expired access token; the refresh cookie identifies the session
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    public async Task<IActionResult> LogoutAsync(CancellationToken ct)
    {
        if (Request.Cookies.TryGetValue(RefreshCookie, out var refreshToken))
            await authBll.LogoutAsync(refreshToken, ct);
        ClearRefreshCookie();
        return NoContent();
    }
}
```

Register follows the login shape: `request.Adapt<RegisterDto>()` → `authBll.RegisterAsync` → `TokenResponse` (201 when created if you prefer `Created`).

## 9. Api: authentication, current user, roles, rate limiting

```csharp
// <App>.Api/Auth/JwtAuthenticationExtensions.cs
using System.Text;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.JsonWebTokens;
using Microsoft.IdentityModel.Tokens;

public static class JwtAuthenticationExtensions
{
    public static IServiceCollection AddJwtAuthentication(this IServiceCollection services, IConfiguration config)
    {
        services.AddOptions<JwtOptions>()
            .Bind(config.GetSection(JwtOptions.Section))
            .ValidateDataAnnotations()
            .ValidateOnStart();

        var jwt = config.GetSection(JwtOptions.Section).Get<JwtOptions>()
                  ?? throw new InvalidOperationException("Missing Jwt configuration.");

        services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
            .AddJwtBearer(o =>
            {
                o.MapInboundClaims = false;
                o.TokenValidationParameters = new TokenValidationParameters
                {
                    ValidateIssuer = true,
                    ValidIssuer = jwt.Issuer,
                    ValidateAudience = true,
                    ValidAudience = jwt.Audience,
                    ValidateLifetime = true,
                    ValidateIssuerSigningKey = true,
                    IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(jwt.SigningKey)),
                    ClockSkew = TimeSpan.FromSeconds(30),
                    NameClaimType = JwtRegisteredClaimNames.Sub,
                    RoleClaimType = "role",
                };
            });

        services.AddAuthorization();
        services.AddSingleton(TimeProvider.System);
        return services;
    }
}

// <App>.Api/Auth/ClaimsPrincipalExtensions.cs
using System.Security.Claims;
using Microsoft.IdentityModel.JsonWebTokens;

public static class ClaimsPrincipalExtensions
{
    public static string GetUserId(this ClaimsPrincipal user) =>
        user.FindFirstValue(JwtRegisteredClaimNames.Sub)
        ?? throw new InvalidOperationException("Authenticated user has no 'sub' claim.");
}

// <App>.Api/Auth/RoleSeeder.cs
using Microsoft.AspNetCore.Identity;

public static class RoleSeeder
{
    public static async Task SeedRolesAsync(this WebApplication app)
    {
        using var scope = app.Services.CreateScope();
        var roles = scope.ServiceProvider.GetRequiredService<RoleManager<IdentityRole>>();
        foreach (var role in Roles.All)
            if (!await roles.RoleExistsAsync(role))
                await roles.CreateAsync(new IdentityRole(role));
    }
}
```

In `<App>.Services` registration: `services.AddScoped<ITokenService, TokenService>(); services.AddScoped<IAuthService, AuthService>(); services.AddScoped<IAuthBll, AuthBll>();`

Rate limiting and CORS in `Program.cs` (full file in the `dotnet-new-solution` skill):

```csharp
builder.Services.AddRateLimiter(o =>
{
    o.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
    o.AddPolicy("auth", ctx => RateLimitPartition.GetFixedWindowLimiter(
        ctx.Connection.RemoteIpAddress?.ToString() ?? "unknown",
        _ => new FixedWindowRateLimiterOptions { PermitLimit = 10, Window = TimeSpan.FromMinutes(1) }));
});

builder.Services.AddCors(o => o.AddPolicy("web", p => p
    .WithOrigins(builder.Configuration.GetSection("Cors:Origins").Get<string[]>() ?? [])
    .AllowAnyHeader()
    .AllowAnyMethod()
    .AllowCredentials())); // needed for the refresh cookie
```

## 10. Configuration and secrets

`appsettings.json` (committed, no secrets):

```json
{
  "Jwt": { "Issuer": "https://api.example.com", "Audience": "example-web", "AccessTokenMinutes": 15, "RefreshTokenDays": 14 },
  "Cors": { "Origins": [ "http://localhost:4200" ] }
}
```

Signing key and connection string locally:

```bash
dotnet user-secrets init --project src/<App>.Api
dotnet user-secrets set "Jwt:SigningKey" "<at least 32 random characters>" --project src/<App>.Api
dotnet user-secrets set "ConnectionStrings:Default" "<connection string>" --project src/<App>.Api
```

In production: environment variables (`Jwt__SigningKey`) or a vault. Rotating the key logs everyone out of their access tokens; refresh tokens keep working.
