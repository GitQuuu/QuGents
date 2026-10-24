---
name: dotnet-new-solution
description: Bootstraps a new solution for this stack from scratch with the dotnet and Angular CLIs - Web API (controllers) + Services + DataAccess + NUnit tests, SQLite (app.db) as the starting database, ASP.NET Core Identity with JWT, central package management, development configuration in appsettings.Development.json, and an Angular + PrimeNG + Tailwind app in its own repository. Use when starting a new project or repo, or when asked how to set one up.
---

# New solution: Web API + Identity/JWT + Angular

**Two repositories.** The backend and the frontend never share a repository:

| Repo | Contains | Local folder |
|---|---|---|
| `<App>` (e.g. `GitQuuu/Shop`) | the .NET solution (sections 1–4) | `C:\_Github\Shop` |
| `<App>-web` (e.g. `GitQuuu/Shop-web`) | the Angular app at the repo root (section 5) | `C:\_Github\Shop-web` (sibling folder) |

Create both repos on GitHub (empty, no README) and clone them side by side before you start.

To work on both in one IDE, **attach** the frontend folder to the backend solution in the IDE (Rider: right-click the solution → *Add* → *Attach Existing Folder*; Visual Studio: *Open Folder* in a second window). Rider keeps this in `.idea/`, so it stays local. Do **not** add the Angular app to the `.slnx` (e.g. as an `.esproj`): the backend CI checks out only the backend repo, so `dotnet restore`/`dotnet build` of the solution would fail on the missing project.

**Start from `dotnet new webapi --use-controllers`, not from an MVC template with Individual auth.** The MVC + Individual template brings cookie authentication, Razor views, Identity UI pages, static files and a `Controller`-based setup that all have to be removed again, and it is easy to leave something behind (cookie schemes, `AddDefaultIdentity`, `_Layout`). The Web API template has nothing to strip; Identity is added with two packages and `AddIdentityCore`, which gives users, roles, password hashing and lockout without cookies or UI. The result is the same as a well-stripped MVC project, minus the cleanup.

Run sections 1–4 from the backend repo root. Replace `Shop` with the solution name. Commands work in PowerShell and Bash. Requires the .NET 10 SDK and Node 22.22+ (Angular 22).

## 1. Solution and projects

```bash
dotnet new sln -n Shop                                   # .NET 10 creates Shop.slnx
dotnet new gitignore
dotnet new editorconfig
dotnet new globaljson --sdk-version 10.0.100 --roll-forward latestFeature   # use your installed 10.0.x
dotnet new buildprops                                    # Directory.Build.props (shared settings)
dotnet new packagesprops                                 # Directory.Packages.props (central package versions)

dotnet new webapi --use-controllers -n Shop.Api -o src/Shop.Api   # then set launchSettings.json to https://localhost:5001
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
# DataAccess: EF Core (SQLite to start, see section 4) + Identity stores
dotnet add src/Shop.DataAccess package Microsoft.EntityFrameworkCore.Sqlite
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
3. `Program.cs`, `Directory.Build.props`, `appsettings.json`, `appsettings.Development.json` and `Shop.Api.http` from `references/solution-templates.md`.

## 4. Database (SQLite `app.db`) and configuration

New solutions start on **SQLite with a local `app.db` file**: no database server to install, and the project owner switches provider when they need to.

- **All development configuration goes in `appsettings.Development.json`**: the connection string `"Data Source=app.db"`, a generated dev JWT signing key and the CORS origin. No user secrets. The file starts with a `// DEV ONLY` comment saying these values must be replaced via environment variables or a vault before production, and `appsettings.json` leaves `SigningKey` and the connection string empty so production cannot start without them.
- Run `dotnet run` and `dotnet ef` **from `src/Shop.Api`**, so the relative path always resolves to `src/Shop.Api/app.db`. Pass `-- --environment Development` to `dotnet ef` so it reads `appsettings.Development.json` (`dotnet run` gets Development from `launchSettings.json`).
- Ignore the database files in git: add `*.db`, `*.db-shm` and `*.db-wal` to `.gitignore`.

```bash
dotnet dev-certs https --trust   # once per machine: the API runs on https://localhost:5001

# generate the dev signing key for appsettings.Development.json
openssl rand -base64 48        # PowerShell: [Convert]::ToBase64String([Security.Cryptography.RandomNumberGenerator]::GetBytes(48))

cd src/Shop.Api
dotnet ef migrations add InitialIdentity --project ../Shop.DataAccess -- --environment Development
dotnet ef database update --project ../Shop.DataAccess -- --environment Development
dotnet run
```

**SQLite limits to design around** (EF Core can store these types but not compare or order them in SQL, so such queries throw):
- Use `DateTime` in UTC for entity timestamps, not `DateTimeOffset`.
- Don't filter or sort on `decimal` columns in queries (prices, amounts) until the move to a server database, or store minor units as `long` (øre/cents).

**Switching database later** (e.g. SQL Server or PostgreSQL):
1. Replace the package in DataAccess (`Microsoft.EntityFrameworkCore.SqlServer` or `Npgsql.EntityFrameworkCore.PostgreSQL`) and `UseSqlite` with `UseSqlServer`/`UseNpgsql` in `AddDataAccess`.
2. Put the new dev connection string in `appsettings.Development.json`; production gets its own via `ConnectionStrings__Default` (environment variable or vault).
3. Migrations are provider-specific: delete the `Migrations` folder and create a fresh `Initial` migration (data in `app.db` is not migrated).
4. The `DateTime`/`decimal` limits above no longer apply.

## 5. Angular app (frontend repo)

Run from the root of the **frontend repo** (`Shop-web`). The app is created in the repo root, not in a subfolder. The folder may already hold `CLAUDE.md`, `.github/`, `.claude/` and `scripts/`, but **not** `README.md` or `.gitignore`: `ng new` stops with "A merge conflicted on path /README.md". Add those after `ng new` (append `certs/` to its `.gitignore`):

```bash
npx @angular/cli@latest new shop-web --directory . --style=css --ssr=false --skip-git
npm install primeng @primeuix/themes @angular/cdk tailwindcss @tailwindcss/postcss postcss tailwindcss-primeui
```

Then follow the `angular-frontend` skill: `.postcssrc.json`, `src/styles.css`, `providePrimeNG` (with license key), interceptors, `AuthStore` + `provideAppInitializer`.

**Ports and HTTPS in development:**

| Part | URL |
|---|---|
| API | `https://localhost:5001` (`launchSettings.json`) |
| Angular dev server | `https://localhost:4200` |

- `environment.apiUrl = 'https://localhost:5001'` and `Cors:Origins = ["https://localhost:4200"]` in `appsettings.Development.json`.
- The Angular dev server must run on HTTPS: with `http://localhost:4200` the browser treats the API as cross-site and never sends the `SameSite=Strict` refresh cookie. Reuse the trusted ASP.NET Core dev certificate:

```bash
# from the frontend repo root
dotnet dev-certs https --export-path ./certs/localhost.pem --format Pem --no-password   # writes localhost.pem + localhost.key
```

In `angular.json` → `projects.shop-web.architect.serve.options`: `"ssl": true, "sslCert": "certs/localhost.pem", "sslKey": "certs/localhost.key"`. Add `certs/` to the frontend repo's `.gitignore`.

Give the frontend repo its own `CLAUDE.md` (stack, commands `npx ng serve` / `npx ng build` / `npx ng test --watch=false`, and the same `## Azure DevOps` section as the backend repo, see the `ado-backlog` skill).

## 6. Done when

- `dotnet build` has no warnings (warnings are errors via `Directory.Build.props`) and `dotnet test` is green.
- `dotnet run` (from `src/Shop.Api`) listens on **https://localhost:5001**; `app.db` sits in `src/Shop.Api` and is git-ignored; `https://localhost:5001/openapi/v1.json` and `https://localhost:5001/swagger` load.
- The `Shop.Api.http` requests work in order: register → login (cookie set, access token returned) → a protected endpoint with the token → refresh → logout.
- In the frontend repo, `npx ng build` and `npx ng test --watch=false` pass, and `npx ng serve` runs on **https://localhost:4200** and talks to the API on https://localhost:5001.
- Dev values exist only in `appsettings.Development.json` with the `// DEV ONLY` comment; `appsettings.json` has empty `SigningKey` and connection string.

**Before production** (not part of the bootstrap, but say it in the hand-off): generate a new signing key and set `Jwt__SigningKey`, `ConnectionStrings__Default` and `Cors__Origins` through environment variables or a vault. The dev key in the repo must never be used outside development.

## 7. Branches and CI/CD

After the first commit in each repo, hand over to the `devops` agent (skill `devops-github-actions`): in **both** repos it creates `development`/`staging`/`prod`, sets `development` as default branch, adds the backend or frontend variant of the CI and deploy workflows, the release and back-merge workflows and the rulesets, and lists the secrets/variables to create per repo and environment.

If the repo starts from the `ai-agent-dotnet-template`, apply the same structure inside it instead of creating a new solution file.

## References

- `references/solution-templates.md`: `Program.cs`, `Directory.Build.props`, `appsettings.json`, `appsettings.Development.json` and `Shop.Api.http` for the auth flow.
