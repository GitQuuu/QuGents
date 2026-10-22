---
name: dotnet-n-layer
description: Conventions for ASP.NET Core N-layer modular monoliths (API → Services → Data Access) with thin controllers and feature-sliced service modules (Controller → BLL → Service → Unit of Work/Repository). Use when writing, placing or reviewing C# in such a solution, e.g. adding an endpoint or use case, a controller action, BLL, service, repository, DTO, DI registration or Hangfire job.
---

# .NET N-layer: API → Service → DAL

Names here (`ServiceResult<T>`, `ResponseService`, `IUnitOfWork`, `I<Name>Bll`...) are conventions of this architecture. If the codebase already has its own version, use that one and follow its shape. If the code has drifted from this description, trust the code and say so.

## The backend is always a Web API

The backend is a JSON Web API and nothing else. All UI lives in the separate Angular app.

- `<App>.Api` is an ASP.NET Core Web API: controllers with `[ApiController]` that inherit `ControllerBase` (not `Controller`) and return DTOs or `ProblemDetails` as JSON through `ResponseService`.
- **No server-rendered UI:** no MVC views (`View()`, `.cshtml`), Razor Pages, Blazor, `ViewBag`/`TempData`, or HTML responses. If a task asks for a screen, the backend part is an endpoint and the screen is built in Angular.
- Controllers, not Minimal APIs, so the thin-controller and slice structure holds (unless the project already uses Minimal APIs consistently).
- Every endpoint is versioned (`api/v{version}/...`) and described in OpenAPI/Swagger through `[ProducesResponseType]`.
- Stateless: ASP.NET Core Identity for users/roles and JWT bearer access tokens with rotating refresh tokens (the `dotnet-identity-jwt` skill), no server-side session state. CORS is an explicit allow-list for the Angular app's origins.
- The API knows nothing about the frontend: no Angular paths, no UI wording in responses beyond `ProblemDetails` titles.

## Projects and dependency direction

```
<App>.Api           composition root, controllers, middleware, ResponseService
   │ references
   ▼
<App>.Services      modules: BLL (use-case orchestration) + Service (domain ops), DTOs, jobs
   │ references
   ▼
<App>.DataAccess    DbContext, entities, repositories, Unit of Work
<App>.Packages      optional: central NuGet references (or Directory.Packages.props)
<App>.Tests         mirrors the Services modules

web/ (or <App>.Web) Angular app: separate project, talks to <App>.Api over HTTP only
```

Call chain: `Controller → I<Name>Bll → I<Name>Service → IUnitOfWork → repository → DbContext`.

**Hard rules**

1. Dependencies point down only. Never upward, never skipping a layer.
2. DataAccess does not reference Services or Api. Services does not reference Api: no controller types, no `IActionResult`, no Api namespaces.
3. Controllers depend on `I<Name>Bll` only. They never touch `IUnitOfWork`, repositories, `DbContext` or `I<Name>Service`.
4. A module reaches another module only through its interface (`I<Other>Bll`/`I<Other>Service`), never through its repositories or internals.
5. Expected failures return `ServiceResult<T>` and flow through `ResponseService`. Throw only for unexpected faults.
6. Services stage changes. The BLL commits once with `IUnitOfWork.SaveChangesAsync()` and then invalidates the cache.
7. Entities never leave the Services layer. Controllers return DTOs.
8. Every new BLL, Service, repository or job is registered in the layer's `IServiceCollection` extension. It does not work until it is.
9. The backend serves JSON only. No MVC views, Razor Pages, Blazor or HTML responses; UI belongs in the Angular app.

## Thin controllers

A controller action does exactly this and nothing more:

1. Binds and validates the request (`[ApiController]` gives an automatic 400 on an invalid model).
2. Reads transport context (user id, roles from claims) and passes it on as **parameters**.
3. Maps the request model to a service DTO (`request.Adapt<XxxDto>()`).
4. Calls **one** `I<Name>Bll` method.
5. Translates the `ServiceResult<T>` with `ResponseService`.

The only exception is the background-job hand-off. There the controller may upload raw input to blob storage, enqueue a job with `IBackgroundJobClient` and return `202 Accepted` with the `jobId`.

**A controller must not:**

- branch on business rules (`if (product.Price > ...)`) or loop over data;
- call more than one BLL method, or orchestrate anything;
- inject `IUnitOfWork`, repositories, `DbContext`, `I<Name>Service` or `ICacheService`;
- call `SaveChangesAsync`, touch the cache, or return entities;
- use `try/catch` for expected failures (that is `ServiceResult`'s job).

Rule of thumb: an action body is 3–6 lines. When it grows, the extra logic belongs in the BLL.

```csharp
// Good: thin
public async Task<IActionResult> EditProductAsync(Guid id, [FromBody] EditProductRequest request, CancellationToken ct)
{
    var dto = request.Adapt<EditProductDto>() with { Id = id };
    var result = await _productBll.EditProductAsync(dto, User.GetUserId(), ct);
    return _response.ToActionResult(result);
}

// Bad: business logic, data access and commit in the controller
public async Task<IActionResult> EditProductAsync(Guid id, EditProductRequest request)
{
    var product = await _uow.Products.GetByIdAsync(id);           // rule 3
    if (product is null) return NotFound();
    if (product.OwnerId != User.GetUserId()) return Forbid();     // business rule
    request.Adapt(product);
    await _uow.SaveChangesAsync();                                 // rule 6
    return Ok(product);                                            // rule 7
}
```

## Layer responsibilities

| Layer | Owns | Never |
|---|---|---|
| **Controller** (Api) | Route, version, auth attributes, binding, mapping, one BLL call, result translation | Business rules, data access, commits, caching |
| **BLL** (Services) | One use case end to end: calls services (own and other modules' interfaces), commits via UoW, checks row count, invalidates cache, returns `ServiceResult<T>` | `HttpContext`, `IActionResult`, `ControllerBase`, direct `DbContext` |
| **Service** (Services) | Focused operations and queries on entities via `IUnitOfWork`; expected failures as `ServiceResult<T>` | `SaveChangesAsync`, HTTP types, other modules' repositories |
| **Repository / UoW** (DataAccess) | Queries, CRUD, includes; `IUnitOfWork` exposes repositories + one `SaveChangesAsync` | Business rules, references upward |

The BLL is the extraction boundary. A module can later move into its own API: copy the controller to a new project, and in the monolith swap `I<Name>Bll` for a `<Name>WebService` that implements the same interface over `HttpClient`. That only works if the BLL takes user context as parameters and returns `ServiceResult<T>`/DTOs, never `IActionResult`.

## Vertical slices from the service layer down

Each use case is a folder named after the operation. The class is split with `partial`. The root files hold only fields and the constructor; never grow them.

```
<App>.Api/Controllers/Products/
  ProductsController.cs                          fields + ctor + class attributes
  EditProductAsync/
    ProductsController.EditProductAsync.cs       the action
    EditProductRequest.cs                        request model beside the action

<App>.Services/Modules/Products/
  IProductBll.cs   ProductBll.cs                 interface + root partial (fields, ctor)
  IProductService.cs   ProductService.cs
  EditProductAsync/
    ProductBll.EditProductAsync.cs
    ProductService.EditProductAsync.cs
    EditProductDto.cs   ProductDto.cs
  InitCreateProductImagesProcessAsync/
    ProductBll.ProcessProductImagesJobAsync.cs
    ProcessProductImagesJob.cs   ProcessProductImagesJobDto.cs

<App>.Tests/Modules/Products/
  EditProductAsyncTests.cs                       one file per operation
```

## Recipe: add a use case

1. Find a comparable slice in the same module and copy its shape.
2. Create the operation folder in the controller and in the service module.
3. Add the method to `I<Name>Bll` (and `I<Name>Service` if a new domain operation is needed).
4. Implement the partials in the folder: Service does the work and returns `ServiceResult<T>`; BLL orchestrates, commits, invalidates cache.
5. Write the thin action with HTTP conventions (below).
6. If a new status code appears, add the case to `ResponseService` and a unit test for it.
7. Register anything new in the DI extension method.
8. Add `<Operation>Tests.cs` in the test project following the `dotnet-testing` skill (NUnit, AAA, FakeItEasy).

## HTTP conventions

- Route `api/v{version:apiVersion}/[controller]` with `[ApiVersion]` and `[ApiController]`.
- `[EnableRateLimiting]` on the controller; `[Authorize]` and role checks on protected actions.
- `[ProducesResponseType]` for every outcome the BLL can return.
- Slow work returns `202 Accepted` + `jobId`; a jobs endpoint exposes status for polling.

## Background jobs (Hangfire)

- Payloads are serializable records: blob paths, ids, `userId`, `alias`, role flags. Never `IFormFile`, streams or `HttpContext`.
- The job calls `I<Name>Bll.Process…JobAsync`. It throws when the BLL returns status ≥ 400 so Hangfire records the failure.
- `[AutomaticRetry]` and `[DisableConcurrentExecution]` on the job method. Capture to Sentry with tags, then rethrow.
- Clean up temporary blobs best-effort after success. Do not delete them before the last retry.

## Other patterns in use

| Pattern | Rule |
|---|---|
| Result object | `ServiceResult<T>`: success flag, `HttpStatusCode`, message or data |
| Response translator | `ResponseService` maps status code → `IActionResult` (`NotFound`, `Forbid`, `Conflict`...) |
| Mapping | Mapster `Adapt<T>()`, configured centrally. Request → DTO in the controller, entity ↔ DTO in services |
| Repository | Generic `IBaseRepository<T>` (CRUD, predicate queries, `IncludeReferenceAsync`) plus one repository per aggregate |
| Unit of Work | Scoped: one request shares one `DbContext` across all repositories |
| Cache-aside | `ICacheService` over `IMemoryCache`; the BLL calls `ClearAll()` after writes |
| Adapters | Blob storage, email, image hosting behind interfaces; named `HttpClient`s configured centrally |
| DI | One `Add<Layer>()` extension per project, called from `Program.cs` |
| Entities | Anemic. Business rules live in Services and the BLL. Timestamps as UTC `DateTime` |
| Database | SQLite (`Data Source=app.db`) until the project switches provider. Check the provider in `AddDataAccess`; on SQLite, queries cannot compare/order `DateTimeOffset` or `decimal` |
| Auth | ASP.NET Core Identity (`AddIdentityCore`, no cookies/UI) + JWT bearer + refresh tokens; `[Authorize]` by default, user id from `sub` passed to the BLL. Details: `dotnet-identity-jwt` |

## References

- `references/code-templates.md`: C# skeletons for every layer (controller, BLL, service, `ServiceResult`, `ResponseService`, UoW, repository, DI, Hangfire job, test, extraction `WebService`). Read it before writing new code in this architecture.
- `references/review-checklist.md`: ordered review checklist, thin-controller smells and common drift. Read it when reviewing a change or a codebase.
- Related skills: `dotnet-identity-jwt` (auth), `dotnet-new-solution` (starting a new solution), `dotnet-testing` (tests).
