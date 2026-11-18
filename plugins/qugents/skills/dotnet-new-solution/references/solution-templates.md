# Solution templates

For a new solution named `Shop` (replace the name). **Not compiled in the sandbox (no NuGet access): run `dotnet build` and the `.http` flow once after creating the project.**

## `Directory.Build.props` (repo root)

```xml
<Project>
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <AnalysisLevel>latest-recommended</AnalysisLevel>
    <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
  </PropertyGroup>
</Project>
```

Remove `<TargetFramework>`, `<Nullable>` and `<ImplicitUsings>` from the individual `.csproj` files so they come from here. If `latest-recommended` produces too many warnings in an existing codebase, start with `latest` and tighten later.

## `src/Shop.Api/Program.cs`

**`Program.cs` stays a table of contents.** Every service registration or middleware that needs more than one line of setup lives in an extension method in `src/Shop.Api/Extensions/<Concern>Extensions.cs`, named `Add<Concern>Extension()` / `Use<Concern>Extension()` (the suffix avoids clashing with the framework method it wraps, e.g. `AddApiVersioning`, `UseSwaggerUI`). Plain one-liners (`AddControllers()`, `UseHttpsRedirection()`, `MapControllers()`) stay in `Program.cs`. The order of the `Use…` calls in `Program.cs` is the middleware order, so keep them there and do not hide several middlewares in one extension.

```csharp
using Shop.Api.Extensions;
using Shop.DataAccess;
using Shop.Services;

var builder = WebApplication.CreateBuilder(args);

// Layers
builder.Services
    .AddDataAccess(builder.Configuration)   // DbContext, Identity (AddIdentityCore), UoW, repositories
    .AddServices()                          // BLLs, services, TokenService, cache
    .AddScoped<IResponseService, ResponseService>();

// Auth: Identity + JWT (dotnet-identity-jwt skill)
builder.Services.AddJwtAuthentication(builder.Configuration);

// Web API
builder.Services.AddControllers();
builder.Services.AddProblemDetails();
builder.Services.AddMapsterExtension();       // global Mapster settings; map with Adapt<T>(), no per-type configs
builder.Services.AddApiVersioningExtension(); // version = folder (Controllers/V1 → v1.0), reports supported versions
builder.Services.AddOpenApiExtension();       // built-in document at /openapi/v1.json
builder.Services.AddCorsExtension();          // allow-list from Cors:Origins, credentials for the refresh cookie
builder.Services.AddRateLimiterExtension();   // "default" and "auth" policies

var app = builder.Build();

app.UseExceptionHandler();      // ProblemDetails for unhandled exceptions
app.UseHttpsRedirection();
app.UseApiDocsLoginExtension(); // browser login (HTTP Basic) on /swagger and /openapi outside Development
app.UseOpenApiExtension();      // /openapi/v1.json: open in Development, behind the docs login elsewhere
app.UseSwaggerUIExtension();    // /swagger: same rule
app.UseCorsExtension();
app.UseAuthentication();
app.UseAuthorization();
app.UseRateLimiter();

app.MapControllers().RequireAuthorization(); // secure by default; [AllowAnonymous] opts out

await app.SeedRolesAsync();
await app.RunAsync();
```

`RequireAuthorization()` on `MapControllers()` makes every endpoint require a valid token unless it has `[AllowAnonymous]`, so a forgotten `[Authorize]` fails closed.

To try protected endpoints from Swagger UI, paste the token into a REST client (`Shop.Api.http`) instead; the built-in OpenAPI document has no security scheme until you add a document transformer for it.

### `src/Shop.Api/Extensions/MapsterExtensions.cs`

```csharp
using Mapster;

namespace Shop.Api.Extensions;

public static class MapsterExtensions
{
    /// <summary>
    /// Global Mapster settings only; no per-type mapping configs. Types map by member name with Adapt&lt;T&gt;(),
    /// and every destination member must have a source, so a renamed member fails loudly.
    /// </summary>
    public static IServiceCollection AddMapsterExtension(this IServiceCollection services)
    {
        TypeAdapterConfig.GlobalSettings.Default
            .NameMatchingStrategy(NameMatchingStrategy.Flexible)
            .RequireDestinationMemberSource(true);

        return services;
    }
}
```

### `src/Shop.Api/Extensions/ApiVersioningExtensions.cs`

```csharp
using Asp.Versioning;
using Asp.Versioning.Conventions;

namespace Shop.Api.Extensions;

public static class ApiVersioningExtensions
{
    /// <summary>
    /// The folder is the version: Controllers/V1/… (namespace …Controllers.V1.…) is v1.0, Controllers/V2/… is v2.0.
    /// Controllers carry no [ApiVersion]. Routes are api/v{version:apiVersion}/[controller].
    /// </summary>
    public static IServiceCollection AddApiVersioningExtension(this IServiceCollection services)
    {
        services
            .AddApiVersioning(o =>
            {
                o.DefaultApiVersion = new ApiVersion(1, 0);
                o.AssumeDefaultVersionWhenUnspecified = true;
                o.ReportApiVersions = true;
                o.ApiVersionReader = new UrlSegmentApiVersionReader();
            })
            .AddMvc(o => o.Conventions.Add(new VersionByNamespaceConvention()));

        return services;
    }
}
```

### `src/Shop.Api/Extensions/OpenApiExtensions.cs`

API docs (OpenAPI document + Swagger UI) run in **every environment**:
- **Development:** open, no login.
- **Other environments:** behind a login using the browser's built-in username/password popup (HTTP Basic, `WWW-Authenticate: Basic`). The credentials are `ApiDocs__Username` and `ApiDocs__Password`, set in the host's configuration and never in the repo.
- **Not configured:** without the credentials the docs are not served at all (fail closed).

Basic sends the password with every request, so the login comes **after** `UseHttpsRedirection`. The password must be long and random (at least 16 characters; the app refuses to start with a shorter one), which makes guessing impractical without a rate limit. The login is independent of Identity/JWT; app users cannot open the docs with their account.

```csharp
using System.Security.Cryptography;
using System.Text;

namespace Shop.Api.Extensions;

/// <summary>Login for the API docs outside Development. Set via ApiDocs__Username / ApiDocs__Password, never in the repo.</summary>
public sealed class ApiDocsOptions
{
    public const string Section = "ApiDocs";
    public const int MinPasswordLength = 16;

    public string Username { get; init; } = "";
    public string Password { get; init; } = "";

    public bool IsConfigured => Username.Length > 0 && Password.Length > 0;
}

public static class OpenApiExtensions
{
    public const string DocumentPath = "/openapi/v1.json";

    public static IServiceCollection AddOpenApiExtension(this IServiceCollection services)
    {
        services.AddOpenApi();

        return services;
    }

    /// <summary>
    /// Browser login (HTTP Basic) in front of /swagger and /openapi outside Development.
    /// Call after UseHttpsRedirection and before UseOpenApiExtension / UseSwaggerUIExtension.
    /// </summary>
    public static WebApplication UseApiDocsLoginExtension(this WebApplication app)
    {
        if (app.Environment.IsDevelopment())
        {
            return app;
        }

        var options = GetOptions(app);
        if (!options.IsConfigured)
        {
            return app; // docs are not mapped at all, see DocsEnabled
        }

        if (options.Password.Length < ApiDocsOptions.MinPasswordLength)
        {
            throw new InvalidOperationException(
                $"ApiDocs:Password must be at least {ApiDocsOptions.MinPasswordLength} characters.");
        }

        app.UseWhen(context => IsDocsPath(context.Request.Path), docs => docs.Use(async (context, next) =>
        {
            if (IsAuthorized(context.Request.Headers.Authorization, options))
            {
                await next(context);
                return;
            }

            context.Response.StatusCode = StatusCodes.Status401Unauthorized;
            context.Response.Headers.WWWAuthenticate = "Basic realm=\"API docs\", charset=\"UTF-8\"";
        }));

        return app;
    }

    public static WebApplication UseOpenApiExtension(this WebApplication app)
    {
        if (DocsEnabled(app))
        {
            app.MapOpenApi().AllowAnonymous(); // JWT does not apply here; the docs login protects it
        }

        return app;
    }

    public static WebApplication UseSwaggerUIExtension(this WebApplication app)
    {
        if (DocsEnabled(app))
        {
            app.UseSwaggerUI(o => o.SwaggerEndpoint(DocumentPath, "Shop API v1"));
        }

        return app;
    }

    private static bool DocsEnabled(WebApplication app) =>
        app.Environment.IsDevelopment() || GetOptions(app).IsConfigured;

    private static ApiDocsOptions GetOptions(WebApplication app) =>
        app.Configuration.GetSection(ApiDocsOptions.Section).Get<ApiDocsOptions>() ?? new ApiDocsOptions();

    private static bool IsDocsPath(PathString path) =>
        path.StartsWithSegments("/swagger") || path.StartsWithSegments("/openapi");

    private static bool IsAuthorized(string? header, ApiDocsOptions options)
    {
        const string prefix = "Basic ";
        if (header is null || !header.StartsWith(prefix, StringComparison.OrdinalIgnoreCase))
        {
            return false;
        }

        string credentials;
        try
        {
            credentials = Encoding.UTF8.GetString(Convert.FromBase64String(header[prefix.Length..].Trim()));
        }
        catch (FormatException)
        {
            return false;
        }

        var separator = credentials.IndexOf(':');
        if (separator < 0)
        {
            return false;
        }

        // '&' (not '&&') so a wrong username costs as much time as a wrong password
        return FixedTimeEquals(credentials[..separator], options.Username)
             & FixedTimeEquals(credentials[(separator + 1)..], options.Password);
    }

    private static bool FixedTimeEquals(string a, string b) =>
        CryptographicOperations.FixedTimeEquals(
            SHA256.HashData(Encoding.UTF8.GetBytes(a)),
            SHA256.HashData(Encoding.UTF8.GetBytes(b)));
}
```

Tests (API behaviour, `WebApplicationFactory` with environment `Production` and test credentials): `/swagger` without credentials → 401 with a `WWW-Authenticate: Basic` header; wrong password → 401; right credentials → 200 for `/swagger/index.html` and `/openapi/v1.json`; without `ApiDocs` settings → 404; Development → 200 without credentials.

### `src/Shop.Api/Extensions/CorsExtensions.cs`

```csharp
using Microsoft.AspNetCore.Cors.Infrastructure;

namespace Shop.Api.Extensions;

public static class CorsExtensions
{
    public const string WebPolicy = "web";
    public const string OriginsKey = "Cors:Origins";

    /// <summary>Allow-list for the Angular app from Cors:Origins. Credentials allowed (refresh-token cookie).</summary>
    public static IServiceCollection AddCorsExtension(this IServiceCollection services)
    {
        services.AddCors();
        services.AddOptions<CorsOptions>().Configure<IConfiguration>((options, configuration) =>
        {
            var origins = configuration.GetSection(OriginsKey).Get<string[]>() ?? [];
            options.AddPolicy(WebPolicy, policy => policy
                .WithOrigins(origins)
                .AllowAnyHeader()
                .AllowAnyMethod()
                .AllowCredentials());
        });

        return services;
    }

    public static IApplicationBuilder UseCorsExtension(this IApplicationBuilder app) =>
        app.UseCors(WebPolicy);
}
```

### `src/Shop.Api/Extensions/RateLimiterExtensions.cs`

```csharp
using System.Threading.RateLimiting;

namespace Shop.Api.Extensions;

public static class RateLimiterExtensions
{
    public static IServiceCollection AddRateLimiterExtension(this IServiceCollection services)
    {
        services.AddRateLimiter(o =>
        {
            o.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
            o.AddPolicy("default", ctx => RateLimitPartition.GetTokenBucketLimiter(
                ctx.Connection.RemoteIpAddress?.ToString() ?? "unknown",
                _ => new TokenBucketRateLimiterOptions
                {
                    TokenLimit = 100,
                    TokensPerPeriod = 50,
                    ReplenishmentPeriod = TimeSpan.FromSeconds(10),
                    QueueLimit = 0,
                }));
            o.AddPolicy("auth", ctx => RateLimitPartition.GetFixedWindowLimiter(
                ctx.Connection.RemoteIpAddress?.ToString() ?? "unknown",
                _ => new FixedWindowRateLimiterOptions { PermitLimit = 10, Window = TimeSpan.FromMinutes(1) }));
        });

        return services;
    }
}
```

## `src/Shop.Api/Properties/launchSettings.json`

The backend always starts on **https://localhost:5001**. Replace the template's generated profiles (random ports) with one profile; `dotnet run` uses the first profile.

```json
{
  "$schema": "https://json.schemastore.org/launchsettings.json",
  "profiles": {
    "https": {
      "commandName": "Project",
      "dotnetRunMessages": true,
      "launchBrowser": false,
      "applicationUrl": "https://localhost:5001",
      "environmentVariables": { "ASPNETCORE_ENVIRONMENT": "Development" }
    }
  }
}
```

HTTPS only, on purpose: the refresh-token cookie is `Secure`. Trust the ASP.NET Core dev certificate once per machine with `dotnet dev-certs https --trust`.

## Configuration

During development **all configuration lives in `appsettings.Development.json`** (committed), including the dev signing key and the SQLite connection string. No user secrets. `appsettings.json` holds only non-secret defaults and leaves the production values empty, so a production start without them fails fast (`ValidateOnStart`).

ASP.NET Core's JSON configuration accepts `//` comments, so the files carry the production reminder themselves.

`src/Shop.Api/appsettings.json`

```jsonc
{
  "Logging": { "LogLevel": { "Default": "Information", "Microsoft.AspNetCore": "Warning" } },
  "AllowedHosts": "*",
  // PRODUCTION: set ConnectionStrings__Default, Jwt__SigningKey, Cors__Origins and ApiDocs__Username/ApiDocs__Password
  // via environment variables or a vault. Without ApiDocs the Swagger UI and OpenAPI document are off.
  "ConnectionStrings": { "Default": "" },
  "Jwt": {
    "Issuer": "https://localhost:5001",
    "Audience": "shop-web",
    "SigningKey": "",
    "AccessTokenMinutes": 15,
    "RefreshTokenDays": 14
  },
  "Cors": { "Origins": [] },
  "ApiDocs": { "Username": "", "Password": "" }
}
```

`src/Shop.Api/appsettings.Development.json`

```jsonc
{
  // DEV ONLY: local development values, committed on purpose so the project runs after a clone.
  // Before the project goes to production: generate a NEW signing key and supply Jwt__SigningKey,
  // ConnectionStrings__Default and Cors__Origins through environment variables or a vault.
  // Never reuse this signing key outside development.
  "ConnectionStrings": { "Default": "Data Source=app.db" },
  "Jwt": { "SigningKey": "<random dev key, at least 32 characters, generated per project>" },
  "Cors": { "Origins": [ "https://localhost:4200" ] }
}
```

Generate the dev key once per project and paste it in:

```bash
# PowerShell
[Convert]::ToBase64String([Security.Cryptography.RandomNumberGenerator]::GetBytes(48))
# Bash
openssl rand -base64 48
```

## `src/Shop.Api/Shop.Api.http`

Replace the template's sample file. Run the requests top to bottom. Named-request syntax (`{{login.response.body.$.accessToken}}`) works in Visual Studio and VS Code REST Client; in Rider, paste the access token from the login response instead.

```http
@api = https://localhost:5001/api/v1
@email = admin@example.com
@password = Correct-Horse-Battery-1

### Register
POST {{api}}/auth/register
Content-Type: application/json

{ "email": "{{email}}", "password": "{{password}}", "displayName": "Admin" }

### Login (sets the refresh_token cookie, returns the access token)
# @name login
POST {{api}}/auth/login
Content-Type: application/json

{ "email": "{{email}}", "password": "{{password}}" }

### Protected endpoint with the access token
GET {{api}}/products
Authorization: Bearer {{login.response.body.$.accessToken}}

### Refresh (uses the cookie from login)
POST {{api}}/auth/refresh

### Logout
POST {{api}}/auth/logout
```

The cookie has `Secure` and `Path=/api/v1/auth`, so refresh/logout only work over HTTPS (which is why the API only listens on https://localhost:5001).
