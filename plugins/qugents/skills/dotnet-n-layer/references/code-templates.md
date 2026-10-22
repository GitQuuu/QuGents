# Code templates

Skeletons for one module (`Products`) and one use case (`EditProductAsync`). Replace names, keep the shape. If the codebase already defines a type shown here, use the existing one.

Packages assumed: `Asp.Versioning.Mvc`, `Mapster`, `Microsoft.EntityFrameworkCore`, `Hangfire.Core`, `Sentry`, `NUnit`, `FakeItEasy`.

## Contents

1. Shared: `ServiceResult<T>`
2. Api: `ResponseService`
3. Api: controller (root + action partial + request)
4. Services: BLL (interface, root, slice)
5. Services: Service (interface, root, slice)
6. DataAccess: `IBaseRepository<T>`, aggregate repository, `IUnitOfWork`
7. DI extensions + `Program.cs`
8. Background job hand-off (controller → Hangfire job → BLL)
9. Tests
10. Extraction: `<Name>WebService`

---

## 1. `ServiceResult<T>` (Services, e.g. `Common/ServiceResult.cs`)

```csharp
using System.Net;

public sealed class ServiceResult<T>
{
    public bool Success { get; init; }
    public HttpStatusCode StatusCode { get; init; }
    public string? Message { get; init; }
    public T? Data { get; init; }

    public static ServiceResult<T> Ok(T data, HttpStatusCode status = HttpStatusCode.OK) =>
        new() { Success = true, StatusCode = status, Data = data };

    public static ServiceResult<T> Fail(HttpStatusCode status, string message) =>
        new() { Success = false, StatusCode = status, Message = message };

    /// Pass a failure from another result type up the chain unchanged.
    public static ServiceResult<T> From<TOther>(ServiceResult<TOther> failed) =>
        Fail(failed.StatusCode, failed.Message ?? "Operation failed.");
}
```

## 2. `ResponseService` (Api, it produces MVC types)

```csharp
using System.Net;
using Microsoft.AspNetCore.Mvc;

public interface IResponseService
{
    IActionResult ToActionResult<T>(ServiceResult<T> result);
}

public sealed class ResponseService : IResponseService
{
    public IActionResult ToActionResult<T>(ServiceResult<T> result) => result.StatusCode switch
    {
        HttpStatusCode.OK         => new OkObjectResult(result.Data),
        HttpStatusCode.Created    => new ObjectResult(result.Data) { StatusCode = StatusCodes.Status201Created },
        HttpStatusCode.Accepted   => new AcceptedResult(location: (string?)null, value: result.Data),
        HttpStatusCode.NoContent  => new NoContentResult(),
        HttpStatusCode.BadRequest => new BadRequestObjectResult(Problem(result)),
        HttpStatusCode.NotFound   => new NotFoundObjectResult(Problem(result)),
        HttpStatusCode.Forbidden  => new ForbidResult(),
        HttpStatusCode.Conflict   => new ConflictObjectResult(Problem(result)),
        _                         => new ObjectResult(Problem(result)) { StatusCode = (int)result.StatusCode },
    };

    private static ProblemDetails Problem<T>(ServiceResult<T> result) =>
        new() { Status = (int)result.StatusCode, Title = result.Message };
}
```

Every new case gets a unit test (section 9).

## 3. Controller

`Controllers/Products/ProductsController.cs`: root partial, fields and constructor only.

```csharp
using Asp.Versioning;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.RateLimiting;

[ApiController]
[ApiVersion("1.0")]
[Route("api/v{version:apiVersion}/[controller]")]
[EnableRateLimiting("default")]
public sealed partial class ProductsController : ControllerBase
{
    private readonly IProductBll _productBll;
    private readonly IResponseService _response;

    public ProductsController(IProductBll productBll, IResponseService response)
    {
        _productBll = productBll;
        _response = response;
    }
}
```

`Controllers/Products/EditProductAsync/ProductsController.EditProductAsync.cs`: one action, thin.

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using Mapster;

public sealed partial class ProductsController
{
    [HttpPut("{id:guid}")]
    [Authorize]
    [ProducesResponseType(typeof(ProductDto), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    [ProducesResponseType(StatusCodes.Status403Forbidden)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> EditProductAsync(Guid id, [FromBody] EditProductRequest request, CancellationToken ct)
    {
        var dto = request.Adapt<EditProductDto>() with { Id = id };
        var result = await _productBll.EditProductAsync(dto, User.GetUserId(), ct);
        return _response.ToActionResult(result);
    }
}
```

`Controllers/Products/EditProductAsync/EditProductRequest.cs`

```csharp
using System.ComponentModel.DataAnnotations;

public sealed record EditProductRequest(
    [property: Required, MaxLength(200)] string Name,
    [property: Range(0, 1_000_000)] decimal Price);
```

`User.GetUserId()` is a small `ClaimsPrincipal` extension in the Api project that reads the JWT `sub` claim (see the `dotnet-identity-jwt` skill). The BLL receives the value, never the principal.

## 4. BLL

`Modules/Products/IProductBll.cs`

```csharp
public interface IProductBll
{
    Task<ServiceResult<ProductDto>> EditProductAsync(EditProductDto dto, string userId, CancellationToken ct);
}
```

`Modules/Products/ProductBll.cs`: root partial.

```csharp
public sealed partial class ProductBll : IProductBll
{
    private readonly IUnitOfWork _uow;
    private readonly IProductService _productService;
    private readonly ICacheService _cache;

    public ProductBll(IUnitOfWork uow, IProductService productService, ICacheService cache)
    {
        _uow = uow;
        _productService = productService;
        _cache = cache;
    }
}
```

`Modules/Products/EditProductAsync/ProductBll.EditProductAsync.cs`: orchestrate, commit, invalidate.

```csharp
using System.Net;

public sealed partial class ProductBll
{
    public async Task<ServiceResult<ProductDto>> EditProductAsync(EditProductDto dto, string userId, CancellationToken ct)
    {
        var edited = await _productService.EditProductAsync(dto, userId, ct);
        if (!edited.Success)
            return edited;

        var rows = await _uow.SaveChangesAsync(ct);
        if (rows == 0)
            return ServiceResult<ProductDto>.Fail(HttpStatusCode.Conflict, "Nothing was saved.");

        _cache.ClearAll();
        return edited;
    }
}
```

## 5. Service

`Modules/Products/IProductService.cs`

```csharp
public interface IProductService
{
    Task<ServiceResult<ProductDto>> EditProductAsync(EditProductDto dto, string userId, CancellationToken ct);
}
```

`Modules/Products/ProductService.cs`: root partial.

```csharp
public sealed partial class ProductService : IProductService
{
    private readonly IUnitOfWork _uow;

    public ProductService(IUnitOfWork uow) => _uow = uow;
}
```

`Modules/Products/EditProductAsync/ProductService.EditProductAsync.cs`: do the work, do not commit.

```csharp
using System.Net;
using Mapster;

public sealed partial class ProductService
{
    public async Task<ServiceResult<ProductDto>> EditProductAsync(EditProductDto dto, string userId, CancellationToken ct)
    {
        var product = await _uow.Products.GetByIdAsync(dto.Id, ct);
        if (product is null)
            return ServiceResult<ProductDto>.Fail(HttpStatusCode.NotFound, "Product not found.");
        if (product.OwnerId != userId)
            return ServiceResult<ProductDto>.Fail(HttpStatusCode.Forbidden, "Not allowed.");

        dto.Adapt(product);          // map onto the tracked entity
        _uow.Products.Update(product);

        return ServiceResult<ProductDto>.Ok(product.Adapt<ProductDto>());
    }
}
```

`Modules/Products/EditProductAsync/EditProductDto.cs` and `ProductDto.cs`

```csharp
public sealed record EditProductDto(Guid Id, string Name, decimal Price);
public sealed record ProductDto(Guid Id, string Name, decimal Price);
```

## 6. Data access

`Repositories/IBaseRepository.cs`

```csharp
using System.Linq.Expressions;

public interface IBaseRepository<T> where T : class
{
    Task<T?> GetByIdAsync(object id, CancellationToken ct = default);
    Task<IReadOnlyList<T>> FindAsync(Expression<Func<T, bool>> predicate, CancellationToken ct = default);
    Task AddAsync(T entity, CancellationToken ct = default);
    void Update(T entity);
    void Remove(T entity);
    Task IncludeReferenceAsync<TProperty>(T entity, Expression<Func<T, TProperty?>> navigation, CancellationToken ct = default)
        where TProperty : class;
}
```

`Repositories/BaseRepository.cs`

```csharp
using System.Linq.Expressions;
using Microsoft.EntityFrameworkCore;

public class BaseRepository<T> : IBaseRepository<T> where T : class
{
    protected readonly AppDbContext Db;
    protected readonly DbSet<T> Set;

    public BaseRepository(AppDbContext db)
    {
        Db = db;
        Set = db.Set<T>();
    }

    public async Task<T?> GetByIdAsync(object id, CancellationToken ct = default) =>
        await Set.FindAsync([id], ct);

    public async Task<IReadOnlyList<T>> FindAsync(Expression<Func<T, bool>> predicate, CancellationToken ct = default) =>
        await Set.Where(predicate).ToListAsync(ct);

    public async Task AddAsync(T entity, CancellationToken ct = default) => await Set.AddAsync(entity, ct);
    public void Update(T entity) => Set.Update(entity);
    public void Remove(T entity) => Set.Remove(entity);

    public Task IncludeReferenceAsync<TProperty>(T entity, Expression<Func<T, TProperty?>> navigation, CancellationToken ct = default)
        where TProperty : class =>
        Db.Entry(entity).Reference(navigation).LoadAsync(ct);
}
```

`Repositories/Products/IProductRepository.cs`: aggregate-specific queries only.

```csharp
public interface IProductRepository : IBaseRepository<Product>
{
    Task<IReadOnlyList<Product>> GetByOwnerAsync(string ownerId, CancellationToken ct = default);
}

public sealed class ProductRepository(AppDbContext db) : BaseRepository<Product>(db), IProductRepository
{
    public async Task<IReadOnlyList<Product>> GetByOwnerAsync(string ownerId, CancellationToken ct = default) =>
        await Set.Where(p => p.OwnerId == ownerId).AsNoTracking().ToListAsync(ct);
}
```

`UnitOfWork/IUnitOfWork.cs`: the only door to data for the Services layer.

```csharp
public interface IUnitOfWork
{
    IProductRepository Products { get; }
    Task<int> SaveChangesAsync(CancellationToken ct = default);
}

public sealed class UnitOfWork(AppDbContext db, IProductRepository products) : IUnitOfWork
{
    public IProductRepository Products { get; } = products;
    public Task<int> SaveChangesAsync(CancellationToken ct = default) => db.SaveChangesAsync(ct);
}
```

Everything is registered **scoped**, so the repositories and the UoW share one `DbContext` per request.

## 7. DI

One extension per project. `Program.cs` only calls them.

```csharp
// <App>.DataAccess/DependencyInjection.cs
public static class DataAccessRegistration
{
    public static IServiceCollection AddDataAccess(this IServiceCollection services, IConfiguration config)
    {
        services.AddDbContext<AppDbContext>(o => o.UseSqlServer(config.GetConnectionString("Default")));
        services.AddScoped<IUnitOfWork, UnitOfWork>();
        services.AddScoped<IProductRepository, ProductRepository>();
        return services;
    }
}

// <App>.Services/DependencyInjection.cs
public static class ServicesRegistration
{
    public static IServiceCollection AddServices(this IServiceCollection services)
    {
        services.AddSingleton<ICacheService, CacheService>();
        services.AddProductModule();
        return services;
    }

    private static IServiceCollection AddProductModule(this IServiceCollection services)
    {
        services.AddScoped<IProductBll, ProductBll>();
        services.AddScoped<IProductService, ProductService>();
        services.AddScoped<ProcessProductImagesJob>();
        return services;
    }
}

// <App>.Api/Program.cs
builder.Services
    .AddDataAccess(builder.Configuration)
    .AddServices()
    .AddScoped<IResponseService, ResponseService>();
```

Connection strings, Sentry DSN and JWT keys come from configuration or user secrets, never from source.

## 8. Background job hand-off

Controller action: validate, store raw input, enqueue, return 202. Blob storage and `IBackgroundJobClient` are the only extra dependencies a controller may take. Add `IBlobStorage _blobStorage` and `IBackgroundJobClient _jobs` to the root partial's fields and constructor.

```csharp
public sealed partial class ProductsController
{
    [HttpPost("{id:guid}/images")]
    [Authorize]
    [ProducesResponseType(StatusCodes.Status202Accepted)]
    public async Task<IActionResult> InitCreateProductImagesProcessAsync(
        Guid id, [FromForm] UploadProductImagesRequest request, CancellationToken ct)
    {
        var blobPaths = await _blobStorage.UploadTempAsync(request.Files, ct);
        var payload = new ProcessProductImagesJobDto(id, blobPaths, User.GetUserId(), IsAdmin: User.IsInRole("Admin"));
        var jobId = _jobs.Enqueue<ProcessProductImagesJob>(job => job.RunAsync(payload, CancellationToken.None));
        return Accepted(new { jobId });
    }
}
```

Payload: serializable, explicit user context.

```csharp
public sealed record ProcessProductImagesJobDto(Guid ProductId, IReadOnlyList<string> BlobPaths, string UserId, bool IsAdmin);
```

Job (Services, in the operation folder):

```csharp
using Hangfire;

public sealed class ProcessProductImagesJob(IProductBll productBll, IBlobStorage blobStorage)
{
    [AutomaticRetry(Attempts = 3)]
    [DisableConcurrentExecution(timeoutInSeconds: 600)]
    public async Task RunAsync(ProcessProductImagesJobDto payload, CancellationToken ct)
    {
        try
        {
            var result = await productBll.ProcessProductImagesJobAsync(payload, ct);
            if ((int)result.StatusCode >= 400)
                throw new InvalidOperationException($"Job failed with {(int)result.StatusCode}: {result.Message}");
        }
        catch (Exception ex)
        {
            SentrySdk.CaptureException(ex, scope =>
            {
                scope.SetTag("job", nameof(ProcessProductImagesJob));
                scope.SetTag("productId", payload.ProductId.ToString());
            });
            throw; // let Hangfire record the failure and retry
        }

        // Best-effort cleanup after success only, so retries still find their input.
        try { await blobStorage.DeleteManyAsync(payload.BlobPaths, ct); }
        catch (Exception ex) { SentrySdk.CaptureException(ex); }
    }
}
```

## 9. Tests

Tests follow the `dotnet-testing` skill: NUnit 4, Arrange-Act-Assert, FakeItEasy, one fixture per operation in `<App>.Tests/Modules/<Name>/`. Full fixtures for Service, BLL, `ResponseService`, jobs and repositories are in that skill's `references/test-templates.md`. The minimum for a new use case:

```csharp
[TestFixture]
public sealed class EditProductAsyncTests
{
    private IUnitOfWork _uow = null!;
    private IProductService _service = null!;
    private ProductBll _sut = null!;

    [SetUp] // NUnit shares one fixture instance across tests: create fakes here, not in field initializers
    public void SetUp()
    {
        _uow = A.Fake<IUnitOfWork>();
        _service = A.Fake<IProductService>();
        _sut = new ProductBll(_uow, _service, A.Fake<ICacheService>());
    }

    [Test]
    public async Task ServiceFails_DoesNotCommit()
    {
        // Arrange
        A.CallTo(() => _service.EditProductAsync(A<EditProductDto>._, A<string>._, A<CancellationToken>._))
            .Returns(ServiceResult<ProductDto>.Fail(HttpStatusCode.Forbidden, "Not allowed."));

        // Act
        await _sut.EditProductAsync(new EditProductDto(Guid.NewGuid(), "x", 1), "user-1", CancellationToken.None);

        // Assert
        A.CallTo(() => _uow.SaveChangesAsync(A<CancellationToken>._)).MustNotHaveHappened();
    }
}
```

## 10. Extraction: `<Name>WebService`

When the Products module moves to its own API, the monolith keeps `IProductBll` and swaps the implementation. Controllers do not change.

```csharp
using System.Net.Http.Json;

public sealed class ProductWebService(HttpClient http) : IProductBll
{
    public async Task<ServiceResult<ProductDto>> EditProductAsync(EditProductDto dto, string userId, CancellationToken ct)
    {
        // userId travels in the forwarded bearer token, not in the body.
        var response = await http.PutAsJsonAsync($"api/v1/products/{dto.Id}", dto, ct);
        if (!response.IsSuccessStatusCode)
            return ServiceResult<ProductDto>.Fail(response.StatusCode, await response.Content.ReadAsStringAsync(ct));

        var data = await response.Content.ReadFromJsonAsync<ProductDto>(ct);
        return ServiceResult<ProductDto>.Ok(data!);
    }
}

// Registration in the monolith:
services.AddHttpClient<IProductBll, ProductWebService>(c => c.BaseAddress = new Uri(config["Services:Products"]!));
```
