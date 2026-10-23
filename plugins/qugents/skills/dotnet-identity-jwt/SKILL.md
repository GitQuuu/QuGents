---
name: dotnet-identity-jwt
description: Authentication and authorization for the ASP.NET Core Web API with ASP.NET Core Identity (users, roles, passwords, lockout) and JWT bearer access tokens with rotating refresh tokens, plus the matching Angular auth store. Use when adding or changing login, registration, refresh/logout, roles, [Authorize] policies, the current-user id, token settings, or the frontend auth flow.
---

# Auth: ASP.NET Core Identity + JWT

Identity owns users, password hashing, roles and lockout. The API issues its own **JWT access tokens** and **refresh tokens**. No cookies for authentication, no Identity UI, no `MapIdentityApi` (its tokens are not JWTs and its endpoints don't fit the layers). If the project already has a working auth setup, follow it and say where it differs.

## Decisions

| Topic | Rule |
|---|---|
| Identity registration | `AddIdentityCore<ApplicationUser>().AddRoles<IdentityRole>().AddEntityFrameworkStores<AppDbContext>()`. Not `AddIdentity` (adds cookie schemes), not `AddDefaultIdentity` (adds UI). |
| User entity | `ApplicationUser : IdentityUser` in DataAccess; `AppDbContext : IdentityDbContext<ApplicationUser>`. |
| Password check | `UserManager.CheckPasswordAsync` + `IsLockedOutAsync` / `AccessFailedAsync` / `ResetAccessFailedCountAsync`. No `SignInManager` (it lives in ASP.NET Core and is cookie-oriented). |
| Access token | JWT, HMAC-SHA256, 15 min, created with `JsonWebTokenHandler`. Claims: `sub` (user id), `email`, `jti`, one `role` claim per role. Nothing sensitive in claims. |
| Validation | `AddJwtBearer` with `MapInboundClaims = false`, issuer/audience/lifetime/signing key validated, `NameClaimType = "sub"`, `RoleClaimType = "role"` (otherwise `[Authorize(Roles)]` silently fails). |
| Refresh token | 64 random bytes, **stored only as a SHA-256 hash**, 14 days, rotated on every use, grouped in a `FamilyId`. Reuse of a rotated token revokes the whole family. Password change and "log out everywhere" revoke all of the user's tokens. |
| Transport | Access token in the JSON body, kept **in memory** by the Angular `AuthStore`. Refresh token in an **HttpOnly, Secure, SameSite=Strict cookie** scoped to `/api/v1/auth`. Never in `localStorage`. |
| CORS | Explicit origins + `AllowCredentials()`; the Angular refresh/login/logout calls use `withCredentials: true`. If app and API end up on different sites (not just subdomains), switch the cookie to `SameSite=None` and keep `Secure`. **Development:** API on `https://localhost:5001`, Angular dev server on **`https://localhost:4200`**. Browsers treat `http://localhost:4200` → `https://localhost:5001` as cross-site and drop the `SameSite=Strict` cookie, so refresh silently fails if Angular runs on plain HTTP (verified in Chromium). |
| Settings | `Jwt:Issuer`, `Jwt:Audience`, `Jwt:SigningKey` (≥ 32 bytes) bound to `JwtOptions` with `ValidateOnStart`. During development the key (a generated dev-only key) lives in `appsettings.Development.json` under a `// DEV ONLY` comment; `appsettings.json` keeps it empty. In production a new key comes from environment variables or a vault, never from the repo. |
| Authorization | `[Authorize]` on every controller by default; `[AllowAnonymous]` only on register/login/refresh. Roles via `[Authorize(Roles = Roles.Admin)]` (constants, no magic strings) or policies for anything beyond a role. Ownership checks stay in the Service. |
| Current user | Controllers read `User.GetUserId()` (the `sub` claim) and `User.IsInRole(...)` and pass them to the BLL as parameters. The BLL never sees `HttpContext` or `ClaimsPrincipal`. |
| Rate limiting | A strict `auth` policy (e.g. 10 requests/min per IP) on the auth controller. |
| Errors | Wrong email or password, unknown user and locked-out user all return the same 401 "Invalid email or password." (no user enumeration). Registration errors from `IdentityResult` return 400 with the messages. |

## Where things live (N-layer)

```
<App>.DataAccess/
  Entities/ApplicationUser.cs  Entities/RefreshToken.cs
  AppDbContext.cs                              IdentityDbContext<ApplicationUser> + DbSet<RefreshToken>
  Repositories/Auth/IRefreshTokenRepository.cs  (on IUnitOfWork as RefreshTokens)
  DependencyInjection.cs                       AddDbContext + AddIdentityCore + repositories
<App>.Services/Modules/Auth/
  IAuthBll.cs  AuthBll.cs  IAuthService.cs  AuthService.cs
  ITokenService.cs  TokenService.cs  JwtOptions.cs  Roles.cs  AuthTokens.cs
  RegisterAsync/  LoginAsync/  RefreshAsync/  LogoutAsync/     one slice per operation
<App>.Api/
  Controllers/Auth/AuthController.cs + one folder per action (thin; also sets/clears the refresh cookie)
  Auth/JwtAuthenticationExtensions.cs          AddJwtAuthentication(configuration)
  Auth/ClaimsPrincipalExtensions.cs            GetUserId()
  Auth/RoleSeeder.cs                           creates roles at startup
```

Endpoints: `POST api/v1/auth/register`, `POST api/v1/auth/login`, `POST api/v1/auth/refresh`, `POST api/v1/auth/logout`. Login and refresh return `{ accessToken, expiresAt }` and set the refresh cookie.

## Gotchas

- **`UserManager` saves immediately.** Its write methods (`CreateAsync`, `AddToRoleAsync`, `AccessFailedAsync`…) call `SaveChanges` on the shared scoped `DbContext`, which also flushes anything the UoW has staged. Call them before staging other changes in the same use case, or keep auth use cases free of other writes.
- Refresh-token lookups must be **tracked** (no `AsNoTracking`) because the BLL revokes/rotates the entity.
- Commit in `RefreshAsync` even on the reuse-detected failure path, so the family revocation is saved.
- With `MapInboundClaims = false` the user id is `sub`, not `ClaimTypes.NameIdentifier`. Keep `GetUserId()` and `NameClaimType` in sync.
- Changing roles takes effect at the next access token (≤ 15 min). Revoke refresh tokens if a role removal must apply immediately.

## Angular side

`AuthStore` (signals) holds the access token in memory, exposes `isLoggedIn`, `roles` and `hasRole()`, and shares one in-flight `refresh()` between concurrent 401s. On app start, `provideAppInitializer` calls `refresh()` once to restore the session from the cookie. The auth interceptor adds the bearer header and never tries to refresh on `/auth/` calls. Templates: the `angular-frontend` skill's `references/angular-templates.md`, section 3.

## Testing

- `AuthService`: fake `UserManager<ApplicationUser>` (FakeItEasy can fake it; pass `A.Fake<IUserStore<ApplicationUser>>()` and nulls to the constructor) or use EF InMemory with real Identity.
- Cases: wrong password → 401 and `AccessFailedAsync` called; locked out → 401; success → tokens issued and refresh hash stored; refresh with unknown/expired token → 401; refresh with revoked token → 401 **and** family revoked + committed; rotation sets `ReplacedByTokenHash`; logout revokes the family.
- `TokenService`: the created token validates with the same parameters as `AddJwtBearer` and carries `sub`, `role` and the expected expiry.

## References

- `references/auth-templates.md`: C# for entities, `JwtOptions`, `TokenService`, `AuthService`/`AuthBll` (login, refresh with rotation and reuse detection), the thin `AuthController` with the refresh cookie, `AddJwtAuthentication`, role seeding and `GetUserId()`. Read it before writing auth code.
