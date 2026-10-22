# Review checklist

Check in this order. Report each finding as `path:line`, the rule it breaks, and the smallest fix. Keep real violations apart from taste.

## 1. Dependency direction

- [ ] `<App>.DataAccess.csproj` has no `ProjectReference` to Services or Api.
- [ ] `<App>.Services.csproj` has no `ProjectReference` to Api, and no `using` of Api namespaces.
- [ ] No layer is skipped: Api never uses `IUnitOfWork`, repositories or `DbContext`.
- [ ] Modules talk through `I<Other>Bll`/`I<Other>Service` only, never another module's repository or internals.
- [ ] API only: controllers are `[ApiController]` + `ControllerBase`; no `View()`, `.cshtml`, Razor Pages, Blazor or HTML responses.

Quick checks:

```bash
grep -rn --include=*.csproj "ProjectReference" .
grep -rn "IUnitOfWork\|DbContext\|Repository" --include=*.cs <App>.Api/Controllers
grep -rn "Microsoft.AspNetCore.Mvc\|IActionResult\|ControllerBase\|HttpContext" --include=*.cs <App>.Services
grep -rln --include=*.cs ": Controller\b\|View(\|RazorPages\|AddControllersWithViews" . ; find . -name "*.cshtml" -o -name "*.razor"
```

## 2. Thin controllers

Smells, each one a finding:

- [ ] More than one BLL call in an action, or any call to `I<Name>Service`.
- [ ] `if`/`switch` on business data (prices, ownership, state) instead of on transport concerns.
- [ ] Loops, LINQ over domain data, or aggregation in the action.
- [ ] `SaveChangesAsync`, cache calls, or entity types in the controller.
- [ ] `try/catch` around the BLL call for expected failures.
- [ ] Action body longer than ~6 lines (job hand-off excepted).
- [ ] Request model not beside the action, or a new action added to the root partial.

## 3. Slice shape

- [ ] New use case has its own operation folder in the controller and in the service module.
- [ ] Root partials (`<Name>Bll.cs`, `<Name>Service.cs`, `<Name>Controller.cs`) hold only fields and the constructor.
- [ ] The method is on `I<Name>Bll` (and `I<Name>Service` if new domain work).
- [ ] Everything new is registered in the DI extension.

## 4. Results and errors

- [ ] Expected failures return `ServiceResult<T>`; exceptions only for unexpected faults.
- [ ] Any new status code has a `ResponseService` case and a test.
- [ ] `[ProducesResponseType]` covers every status the BLL can return.

## 5. Persistence

- [ ] Services do not call `SaveChangesAsync`; the BLL commits once per use case.
- [ ] Row count checked where the codebase does; cache invalidated after a successful write.
- [ ] Entities do not leave the Services layer.
- [ ] Read-only queries use `AsNoTracking()`.

## 6. HTTP and security

- [ ] `api/v{version:apiVersion}/[controller]`, `[ApiVersion]`, `[ApiController]`, `[EnableRateLimiting]`.
- [ ] `[Authorize]` and role checks on protected actions. Ownership checks happen in the Service, not the controller.
- [ ] User context (id, alias, roles) is passed to the BLL as parameters.

## 7. Background jobs

- [ ] Payload is a serializable record; no `IFormFile`, streams or `HttpContext`.
- [ ] `[AutomaticRetry]` + `[DisableConcurrentExecution]` on the job method.
- [ ] Sentry capture with tags, then rethrow; throw when BLL status ≥ 400.
- [ ] Temp blobs cleaned up best-effort after success, not before retries are exhausted.

## 8. Tests

- [ ] One test file per operation under `<App>.Tests/Modules/<Name>/`.
- [ ] Failure paths covered (not found, forbidden, nothing saved) and the BLL does not commit when a service fails.

## Common drift (known tensions)

Report these when they matter to the task. Do not fix them unasked.

- **BLL inherits `ControllerBase` or uses `IHttpContextAccessor`.** Couples the BLL to ASP.NET and blocks extraction. New BLL code takes user context as parameters and returns `ServiceResult<T>`/DTOs; the controller translates.
- **Mixed target frameworks**, a CI runtime that differs from the pinned SDK, or a `global.json` with `rollForward`. Check before using newer C#/.NET features.
- **Secrets or DSNs in `Program.cs`.** Move to configuration or user secrets.
- **Machine-specific `<HintPath>`** in project files. Replace with package or project references.
- **One-off import or migration code** inside core service modules. Move to a separate tool or module.
- **`CacheService.ClearAll()` with a static key set.** Simple, but wrong once there is more than one instance. Mention it if scaling out is on the table.
