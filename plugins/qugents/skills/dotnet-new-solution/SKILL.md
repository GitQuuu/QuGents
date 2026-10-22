---
name: dotnet-new-solution
description: Bootstraps a new solution for this stack from scratch with the dotnet and Angular CLIs - Web API (controllers) + Services + DataAccess + NUnit tests, ASP.NET Core Identity with JWT, central package management, user secrets, and an Angular + PrimeNG + Tailwind app. Use when starting a new project or repo, or when asked how to set one up.
---

# New solution: Web API + Identity/JWT + Angular

**Start from `dotnet new webapi --use-controllers`, not from an MVC template with Individual auth.** The MVC + Individual template brings cookie authentication, Razor views, Identity UI pages, static files and a `Controller`-based setup that all have to be removed again, and it is easy to leave something behind (cookie schemes, `AddDefaultIdentity`, `_Layout`). The Web API template has nothing to strip; Identity is added with two packages and `AddIdentityCore`, which gives users, roles, password hashing and lockout without cookies or UI. The result is the same as a well-stripped MVC project, minus the cleanup.

Run the commands from the new repo root. Replace `Shop` with the solution name. Commands work in PowerShell and Bash. Requires the .NET 10 SDK and Node 22.22+ (Angular 22).

## 1. Solution and projects

```bash
dotnet new sln -n Shop                                   # .NET 10 creates Shop.slnx
dotnet new gitignore
dotnet new editorconfig
dotnet new globaljson --sdk-version 10.0.100 --roll-forward latestFeature   # use your installed 10.0.x
dotnet new buildprops                                    # Directory.Build.props (shared settings)
dotnet new packagesprops                                 # Directory.Packages.props (central package versions)

dotnet new webapi --use-controllers -n Shop.Api -o src/Shop.Api
dotnet new classlib -n Shop.Services -o src/Shop.Services
dotnet new classlib -n Shop.DataAccess -o src/Shop.DataAccess
dotnet new nunit -n Shop.Tests -o tests/Shop.Tests

dotnet sln add src/Shop.Api src/Shop.Services src/Shop.DataAccess tests/Shop.Tests

dotnet add src/Shop.Api reference src/Shop.Services
dotnet add src/Shop.Services reference src/Shop.DataAccess
dotnet add tests/Shop.Tests reference src/Shop.Api src/Shop.Services src/Shop.DataAccess
```

Delete the samples: `WeatherForecast.cs`, `Controllers/WeatherForecastController.cs`, and `Class1.cs` in both class libraries.

Central package management (`Directory.Packages.props`) replaces a separate "Packages" project: every version lives in one file, and `dotnet add package` writes it there.

## 2. Packages

```bash
# DataAccess: EF Core + Identity stores
dotnet add src/Shop.DataAccess package Microsoft.EntityFrameworkCore.SqlServer
dotnet add src/Shop.DataAccess package Microsoft.AspNetCore.Identity.EntityFrameworkCore

# Services: mapping + JWT creation
dotnet add src/Shop.Services package Mapster
dotnet add src/Shop.Services package Microsoft.IdentityModel.JsonWebTokens

# Api: JWT validation, versioning, Swagger UI over the built-in OpenAPI document, EF tooling
dotnet add src/Shop.Api package Microsoft.AspNetCore.Authentication.JwtBearer
dotnet add src/Shop.Api package Asp.Versioning.Mvc
dotnet add src/Shop.Api package Swashbuckle.AspNetCore.SwaggerUI
dotnet add src/Shop.Api package Microsoft.EntityFrameworkCore.Design

# Tests
dotnet add tests/Shop.Tests package FakeItEasy
dotnet add tests/Shop.Tests package Microsoft.EntityFrameworkCore.Sqlite

# EF CLI as a local tool (versioned with the repo)
dotnet new tool-manifest
dotnet tool install dotnet-ef
```

Add later, only when a feature needs them: `Hangfire.AspNetCore` + storage (background jobs), `Sentry.AspNetCore` (error tracking).

## 3. Code skeleton

Create, in this order (templates in the referenced skills):

1. `dotnet-n-layer`: `ServiceResult<T>`, `IBaseRepository<T>`/`BaseRepository<T>`, `IUnitOfWork`/`UnitOfWork`, `ResponseService`, `AddDataAccess`/`AddServices` extensions, `ICacheService`.
2. `dotnet-identity-jwt`: `ApplicationUser`, `RefreshToken`, `AppDbContext : IdentityDbContext<ApplicationUser>`, `AddIdentityCore` in `AddDataAccess`, `JwtOptions`, `TokenService`, the Auth module, `AuthController`, `AddJwtAuthentication`, `GetUserId()`, `RoleSeeder`.
3. `Program.cs`, `Directory.Build.props`, `appsettings.json` and `Shop.Api.http` from `references/solution-templates.md`.

## 4. Secrets and database

```bash
dotnet user-secrets init --project src/Shop.Api
dotnet user-secrets set "Jwt:SigningKey" "<at least 32 random characters>" --project src/Shop.Api
dotnet user-secrets set "ConnectionStrings:Default" "Server=localhost;Database=Shop;Trusted_Connection=True;TrustServerCertificate=True" --project src/Shop.Api

dotnet ef migrations add InitialIdentity --project src/Shop.DataAccess --startup-project src/Shop.Api
dotnet ef database update --project src/Shop.DataAccess --startup-project src/Shop.Api
```

## 5. Angular app

```bash
npx @angular/cli@latest new web --style=css --ssr=false --skip-git
cd web
npm install primeng @primeuix/themes @angular/cdk tailwindcss @tailwindcss/postcss postcss tailwindcss-primeui
```

Then follow the `angular-frontend` skill: `.postcssrc.json`, `src/styles.css`, `providePrimeNG` (with license key), interceptors, `AuthStore` + `provideAppInitializer`, `environment.apiUrl`. Point `environment.apiUrl` at the API's dev URL (`https://localhost:<port>` from `launchSettings.json`) and add the Angular dev origin (`http://localhost:4200`) to the API's `Cors:Origins`.

## 6. Done when

- `dotnet build` has no warnings (warnings are errors via `Directory.Build.props`) and `dotnet test` is green.
- `dotnet run --project src/Shop.Api` starts; `/openapi/v1.json` and `/swagger` load in Development.
- The `Shop.Api.http` requests work in order: register → login (cookie set, access token returned) → a protected endpoint with the token → refresh → logout.
- `npx ng build` and `npx ng test --watch=false` pass in `web/`.
- No secrets in committed files (`git grep -n "SigningKey\|Password="` finds only placeholders).

If the repo starts from the `ai-agent-dotnet-template`, apply the same structure inside it instead of creating a new solution file.

## References

- `references/solution-templates.md`: `Program.cs`, `Directory.Build.props`, `appsettings.json`, `appsettings.Development.json` and `Shop.Api.http` for the auth flow.
