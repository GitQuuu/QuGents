---
name: secure-code-review
description: Code-review checklist for correctness, security, robustness and performance in ASP.NET Core + EF Core APIs and Angular frontends, with severity levels and a findings format. Use when reviewing a diff, pull request or file for bugs and vulnerabilities (not layer structure, which the dotnet-n-layer skill covers).
---

# Secure code review: .NET API + Angular

Review what changed, in the context it runs in. Read enough surrounding code to be sure: the caller, the interface, the DI registration, the test. A finding you have not verified is a **Question**, not a defect.

## Severity

| Level | Meaning | Examples |
|---|---|---|
| **Blocker** | Must be fixed before merge: security hole, data loss/corruption, crash on a normal path, broken contract | Missing authorization, IDOR, SQL injection, secret in code, unhandled null on happy path |
| **Major** | Wrong behavior in a realistic case, or a serious maintainability/performance problem | Missing error branch, N+1 on a list endpoint, race on shared state, swallowed exception |
| **Minor** | Works, but fragile or unclear | Misleading name, missing `CancellationToken`, magic number, weak test |
| **Nit** | Taste. Mention at most a few, never block on them | Wording, ordering |

Formatting and style that a linter/formatter enforces are not findings.

## Checklist: ASP.NET Core API

**Authentication and authorization**
- [ ] Every new action has `[Authorize]` (or a deliberate `[AllowAnonymous]` with a reason).
- [ ] Role/policy checks match the requirement.
- [ ] **Ownership/IDOR:** a resource fetched by id is checked against the current user/tenant in the Service before it is read or changed.
- [ ] User identity comes from claims, never from the request body or query.

**Input and binding**
- [ ] Request models are DTOs, never entities (mass assignment). Only intended fields can be set.
- [ ] Validation on request models (`[Required]`, `[MaxLength]`, `[Range]`), plus business validation in the Service.
- [ ] File uploads: size limit, content-type/extension allow-list, no use of the client file name as a path.
- [ ] Paging limits enforced server-side (no unbounded `pageSize`).

**Data access (EF Core)**
- [ ] No string-concatenated SQL. `FromSqlRaw`/`ExecuteSqlRaw` only with parameters; prefer `FromSql($"...")` interpolation.
- [ ] Read-only queries use `AsNoTracking()`.
- [ ] No N+1: no queries inside loops; needed navigations included or projected (`Select` to DTO).
- [ ] Lists are paged or bounded.
- [ ] One `SaveChangesAsync` per use case; partial failures cannot leave half-written data (transaction if several saves are unavoidable).

**Async and resources**
- [ ] No `.Result`, `.Wait()`, `async void` (except event handlers).
- [ ] `CancellationToken` passed through to EF and `HttpClient`.
- [ ] `HttpClient` from `IHttpClientFactory`/named clients, never `new HttpClient()` per call.
- [ ] Disposables disposed (`using`/`await using`).
- [ ] No mutable static state shared across requests; scoped vs singleton lifetimes correct (no scoped service captured by a singleton).

**Errors and logging**
- [ ] Expected failures return `ServiceResult`; exceptions are not swallowed (`catch { }`) or used for control flow.
- [ ] Error responses do not leak stack traces, SQL or internal ids (ProblemDetails with a safe title).
- [ ] No passwords, tokens, personal data or full request bodies in logs/Sentry.

**Secrets and configuration**
- [ ] No connection strings with credentials, keys, DSNs or tokens in code or committed `appsettings.*.json` (a local SQLite `Data Source=app.db` is fine). `*.db` files are git-ignored.
- [ ] CORS is an explicit allow-list, not `AllowAnyOrigin` with credentials.
- [ ] JWT validation keeps issuer, audience, lifetime and signing key validation on; `MapInboundClaims = false` with `NameClaimType = "sub"` and `RoleClaimType = "role"`.
- [ ] Identity via `AddIdentityCore` (no cookie schemes/UI); login gives the same 401 for unknown user, wrong password and lockout, and calls `AccessFailedAsync` on failure.
- [ ] Refresh tokens are stored hashed, rotated on use, revoked as a family on reuse, and sent only in an HttpOnly + Secure cookie scoped to the auth path. Details: `dotnet-identity-jwt` skill.

**Background jobs**
- [ ] Payloads hold no secrets, streams or `HttpContext`; jobs re-check authorization-relevant state if time has passed.
- [ ] Idempotent under retry (`[AutomaticRetry]` will run it again).

## Checklist: Angular

- [ ] No `bypassSecurityTrust*` or `[innerHTML]` with data from users or the API, unless sanitized.
- [ ] No secrets in `environment*.ts` (they ship to the browser). The PrimeUI license key is the exception: public by design.
- [ ] PrimeNG 22+: license configured via `providePrimeNG({ license })`; the license banner is not hidden with CSS/JS.
- [ ] No `::ng-deep` or `!important` overrides of PrimeNG internals; use props, Tailwind classes, `pt` or the theme preset.
- [ ] Tokens not logged; storage choice matches the app's threat model (avoid `localStorage` for refresh tokens where possible).
- [ ] Guards are UX only; the matching API endpoint is protected.
- [ ] Subscriptions end (`takeUntilDestroyed`, `async` pipe, `firstValueFrom`), no leaks from `setInterval`/polling.
- [ ] Errors from the API are surfaced to the user, not swallowed; 401/403 handled centrally.
- [ ] `@for` has a stable `track` key; no heavy work in template expressions.
- [ ] Types match the backend DTOs; no `any` where a DTO type exists.

## Tests

- [ ] The change has tests for its new branches (especially failure and authorization paths).
- [ ] Tests would fail if the behavior were removed (not just asserting mocks were called).
- [ ] No test was weakened or deleted to make the build pass.

## Output format

Start with a verdict, then findings ordered by severity:

```
Verdict: Request changes | Approve with comments | Approve

Blocker
1. src/App.Api/Controllers/Orders/GetOrderAsync/OrdersController.GetOrderAsync.cs:14
   IDOR: the order is returned without checking it belongs to the caller.
   Fix: pass userId to IOrderBll.GetOrderAsync and return 404 from OrderService when OwnerId differs.

Major
2. ...

Questions
- path:line — what you could not verify and what would settle it.
```

Each finding: `path:line`, what is wrong and the concrete consequence, the smallest fix. Group repeated instances of the same problem into one finding with all locations. If there are no findings at a level, omit it. Say explicitly when the change looks good.
