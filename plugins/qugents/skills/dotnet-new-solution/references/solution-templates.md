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

```csharp
using System.Threading.RateLimiting;
using Asp.Versioning;

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
builder.Services
    .AddApiVersioning(o =>
    {
        o.DefaultApiVersion = new ApiVersion(1, 0);
        o.AssumeDefaultVersionWhenUnspecified = true;
        o.ReportApiVersions = true;
    })
    .AddMvc();
builder.Services.AddOpenApi(); // built-in document at /openapi/v1.json

builder.Services.AddCors(o => o.AddPolicy("web", p => p
    .WithOrigins(builder.Configuration.GetSection("Cors:Origins").Get<string[]>() ?? [])
    .AllowAnyHeader()
    .AllowAnyMethod()
    .AllowCredentials())); // refresh-token cookie

builder.Services.AddRateLimiter(o =>
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

var app = builder.Build();

app.UseExceptionHandler(); // ProblemDetails for unhandled exceptions
if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();
    app.UseSwaggerUI(o => o.SwaggerEndpoint("/openapi/v1.json", "Shop API v1"));
}

app.UseHttpsRedirection();
app.UseCors("web");
app.UseAuthentication();
app.UseAuthorization();
app.UseRateLimiter();

app.MapControllers().RequireAuthorization(); // secure by default; [AllowAnonymous] opts out

await app.SeedRolesAsync();
await app.RunAsync();
```

`RequireAuthorization()` on `MapControllers()` makes every endpoint require a valid token unless it has `[AllowAnonymous]`, so a forgotten `[Authorize]` fails closed.

To try protected endpoints from Swagger UI, paste the token into a REST client (`Shop.Api.http`) instead; the built-in OpenAPI document has no security scheme until you add a document transformer for it.

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
  // PRODUCTION: set ConnectionStrings__Default, Jwt__SigningKey and Cors__Origins via environment variables or a vault.
  "ConnectionStrings": { "Default": "" },
  "Jwt": {
    "Issuer": "https://localhost:5001",
    "Audience": "shop-web",
    "SigningKey": "",
    "AccessTokenMinutes": 15,
    "RefreshTokenDays": 14
  },
  "Cors": { "Origins": [] }
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
