# Test templates

Templates for the `Products` module from the `dotnet-n-layer` skill. Replace names, keep the shape.

**Start with section 0** (API behaviour tests through HTTP with a real SQLite database). Sections 2, 3 and 5 build the system under test by hand with faked repositories and `IUnitOfWork`; they are **legacy**, kept for projects that already test that way. For new code, use them only for boundary fakes (third-party clients, external processes).

## 0. API behaviour test (preferred)

`TestData/ApiFactory.cs`: the real API in memory with a temp SQLite database; only boundaries are replaced.

```csharp
using System.Net.Http.Headers;
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Mvc.Testing;
using Microsoft.AspNetCore.TestHost;
using Microsoft.Data.Sqlite;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Time.Testing;

public sealed class ApiFactory : WebApplicationFactory<Program>
{
    public const string JwtIssuer = "shop-tests";
    public const string JwtAudience = "shop-tests-web";
    public const string JwtSigningKey = "test-only-signing-key-0123456789-abcdefghijklmnop";

    private readonly string _dir = Path.Combine(Path.GetTempPath(), "shop-tests-" + Guid.NewGuid().ToString("N"));

    public FakeTimeProvider Clock { get; } = new(new DateTimeOffset(2026, 1, 1, 12, 0, 0, TimeSpan.Zero));

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        Directory.CreateDirectory(_dir);
        builder.ConfigureAppConfiguration((_, c) => c.AddInMemoryCollection(new Dictionary<string, string?>
        {
            ["ConnectionStrings:Default"] = $"Data Source={Path.Combine(_dir, "app.db")}",  // real SQLite, migrated on start
            ["Jwt:Issuer"] = JwtIssuer,
            ["Jwt:Audience"] = JwtAudience,
            ["Jwt:SigningKey"] = JwtSigningKey,
        }));
        builder.ConfigureTestServices(services =>
        {
            services.AddSingleton<TimeProvider>(Clock);                    // boundary: the clock
            services.AddSingleton<IPaymentGateway, FakePaymentGateway>();  // boundary: a third-party API
        });
    }

    /// <summary>An HttpClient signed in as the given user (token minted with the test signing key).</summary>
    public HttpClient ClientFor(string userId, params string[] roles)
    {
        var client = CreateClient();
        client.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", TestJwt.Create(userId, roles));
        return client;
    }

    protected override void Dispose(bool disposing)
    {
        base.Dispose(disposing);
        if (!disposing) return;
        SqliteConnection.ClearAllPools();
        try { Directory.Delete(_dir, recursive: true); } catch (IOException) { /* best effort */ }
    }
}
```

`Api/Products/EditProductTests.cs`: behaviour from the caller's point of view.

```csharp
[TestFixture]
public sealed class EditProductTests
{
    private ApiFactory _api = null!;

    [SetUp]
    public void SetUp() => _api = new ApiFactory();

    [TearDown]
    public void TearDown() => _api.Dispose();

    [Test]
    public async Task Owner_ValidRequest_StoresNewName()
    {
        // Arrange
        var owner = _api.ClientFor("owner-1");
        var created = await CreateProductAsync(owner, "Old name");

        // Act
        var response = await owner.PutAsJsonAsync($"api/v1/products/{created.Id}", new { name = "New name" });

        // Assert
        var stored = await owner.GetFromJsonAsync<ProductResponse>($"api/v1/products/{created.Id}");
        using (Assert.EnterMultipleScope())
        {
            Assert.That(response.StatusCode, Is.EqualTo(HttpStatusCode.OK));
            Assert.That(stored!.Name, Is.EqualTo("New name"));
        }
    }

    [Test]
    public async Task OtherUser_Returns404AndLeavesProductUnchanged()
    {
        // Arrange
        var created = await CreateProductAsync(_api.ClientFor("owner-1"), "Old name");

        // Act
        var response = await _api.ClientFor("user-2").PutAsJsonAsync($"api/v1/products/{created.Id}", new { name = "Hacked" });

        // Assert
        var stored = await _api.ClientFor("owner-1").GetFromJsonAsync<ProductResponse>($"api/v1/products/{created.Id}");
        using (Assert.EnterMultipleScope())
        {
            Assert.That(response.StatusCode, Is.EqualTo(HttpStatusCode.NotFound));
            Assert.That(stored!.Name, Is.EqualTo("Old name"));
        }
    }

    private static async Task<ProductResponse> CreateProductAsync(HttpClient client, string name)
    {
        var response = await client.PostAsJsonAsync("api/v1/products", new { name });
        response.EnsureSuccessStatusCode();
        return (await response.Content.ReadFromJsonAsync<ProductResponse>())!;
    }
}
```

The helper only performs setup through the public API; the expected values stay literal in the test.

Common usings (or put them in a `GlobalUsings.cs` in the test project):

```csharp
using System.Net;
using FakeItEasy;
using NUnit.Framework;
```

## 1. Test-data builder

`TestData/ProductBuilder.cs`: valid defaults, override only what the test is about.

```csharp
public sealed class ProductBuilder
{
    private Guid _id = Guid.NewGuid();
    private string _ownerId = "user-1";
    private string _name = "Chair";
    private decimal _price = 100m;

    public ProductBuilder WithId(Guid id) { _id = id; return this; }
    public ProductBuilder OwnedBy(string ownerId) { _ownerId = ownerId; return this; }
    public ProductBuilder WithPrice(decimal price) { _price = price; return this; }

    public Product Build() => new() { Id = _id, OwnerId = _ownerId, Name = _name, Price = _price };
}
```

## 2. Service fixture

`Modules/Products/EditProductAsyncTests.cs` (Service part)

```csharp
[TestFixture]
public sealed class EditProductServiceTests
{
    private const string UserId = "user-1";
    private IUnitOfWork _uow = null!;
    private IProductRepository _products = null!;
    private ProductService _sut = null!;

    [SetUp]
    public void SetUp()
    {
        _uow = A.Fake<IUnitOfWork>();
        _products = A.Fake<IProductRepository>();
        A.CallTo(() => _uow.Products).Returns(_products);
        _sut = new ProductService(_uow);
    }

    private static EditProductDto ValidDto(Guid? id = null) => new(id ?? Guid.NewGuid(), "Table", 250m);

    [Test]
    public async Task ProductMissing_ReturnsNotFound()
    {
        // Arrange
        A.CallTo(() => _products.GetByIdAsync(A<object>._, A<CancellationToken>._)).Returns((Product?)null);

        // Act
        var result = await _sut.EditProductAsync(ValidDto(), UserId, CancellationToken.None);

        // Assert
        using (Assert.EnterMultipleScope())
        {
            Assert.That(result.Success, Is.False);
            Assert.That(result.StatusCode, Is.EqualTo(HttpStatusCode.NotFound));
        }
    }

    [Test]
    public async Task NotOwner_ReturnsForbidden_AndDoesNotUpdate()
    {
        // Arrange
        var product = new ProductBuilder().OwnedBy("someone-else").Build();
        A.CallTo(() => _products.GetByIdAsync(A<object>._, A<CancellationToken>._)).Returns(product);

        // Act
        var result = await _sut.EditProductAsync(ValidDto(product.Id), UserId, CancellationToken.None);

        // Assert
        Assert.That(result.StatusCode, Is.EqualTo(HttpStatusCode.Forbidden));
        A.CallTo(() => _products.Update(A<Product>._)).MustNotHaveHappened();
    }

    [Test]
    public async Task Owner_UpdatesProduct_AndReturnsDto()
    {
        // Arrange
        var product = new ProductBuilder().OwnedBy(UserId).Build();
        A.CallTo(() => _products.GetByIdAsync(A<object>._, A<CancellationToken>._)).Returns(product);
        var dto = ValidDto(product.Id);

        // Act
        var result = await _sut.EditProductAsync(dto, UserId, CancellationToken.None);

        // Assert
        using (Assert.EnterMultipleScope())
        {
            Assert.That(result.Success, Is.True);
            Assert.That(result.Data!.Name, Is.EqualTo(dto.Name));
            Assert.That(result.Data.Price, Is.EqualTo(dto.Price));
        }
        A.CallTo(() => _products.Update(product)).MustHaveHappenedOnceExactly();
    }
}
```

## 3. BLL fixture

```csharp
[TestFixture]
public sealed class EditProductBllTests
{
    private IUnitOfWork _uow = null!;
    private IProductService _service = null!;
    private ICacheService _cache = null!;
    private ProductBll _sut = null!;

    [SetUp]
    public void SetUp()
    {
        _uow = A.Fake<IUnitOfWork>();
        _service = A.Fake<IProductService>();
        _cache = A.Fake<ICacheService>();
        _sut = new ProductBll(_uow, _service, _cache);
    }

    private static EditProductDto Dto() => new(Guid.NewGuid(), "Table", 250m);

    [Test]
    public async Task ServiceFails_ReturnsFailure_AndDoesNotCommit()
    {
        // Arrange
        A.CallTo(() => _service.EditProductAsync(A<EditProductDto>._, A<string>._, A<CancellationToken>._))
            .Returns(ServiceResult<ProductDto>.Fail(HttpStatusCode.Forbidden, "Not allowed."));

        // Act
        var result = await _sut.EditProductAsync(Dto(), "user-1", CancellationToken.None);

        // Assert
        Assert.That(result.StatusCode, Is.EqualTo(HttpStatusCode.Forbidden));
        A.CallTo(() => _uow.SaveChangesAsync(A<CancellationToken>._)).MustNotHaveHappened();
        A.CallTo(() => _cache.ClearAll()).MustNotHaveHappened();
    }

    [Test]
    public async Task Saved_CommitsOnce_ThenClearsCache()
    {
        // Arrange
        var dto = new ProductDto(Guid.NewGuid(), "Table", 250m);
        A.CallTo(() => _service.EditProductAsync(A<EditProductDto>._, A<string>._, A<CancellationToken>._))
            .Returns(ServiceResult<ProductDto>.Ok(dto));
        A.CallTo(() => _uow.SaveChangesAsync(A<CancellationToken>._)).Returns(1);

        // Act
        var result = await _sut.EditProductAsync(Dto(), "user-1", CancellationToken.None);

        // Assert
        Assert.That(result.Data, Is.EqualTo(dto));
        A.CallTo(() => _uow.SaveChangesAsync(A<CancellationToken>._)).MustHaveHappenedOnceExactly()
            .Then(A.CallTo(() => _cache.ClearAll()).MustHaveHappenedOnceExactly());
    }

    [Test]
    public async Task NothingSaved_ReturnsConflict_AndKeepsCache()
    {
        // Arrange
        A.CallTo(() => _service.EditProductAsync(A<EditProductDto>._, A<string>._, A<CancellationToken>._))
            .Returns(ServiceResult<ProductDto>.Ok(new ProductDto(Guid.NewGuid(), "Table", 250m)));
        A.CallTo(() => _uow.SaveChangesAsync(A<CancellationToken>._)).Returns(0);

        // Act
        var result = await _sut.EditProductAsync(Dto(), "user-1", CancellationToken.None);

        // Assert
        Assert.That(result.StatusCode, Is.EqualTo(HttpStatusCode.Conflict));
        A.CallTo(() => _cache.ClearAll()).MustNotHaveHappened();
    }
}
```

## 4. ResponseService fixture

```csharp
using Microsoft.AspNetCore.Mvc;

[TestFixture]
public sealed class ResponseServiceTests
{
    private ResponseService _sut = null!;

    [SetUp]
    public void SetUp() => _sut = new ResponseService();

    [TestCase(HttpStatusCode.BadRequest, typeof(BadRequestObjectResult))]
    [TestCase(HttpStatusCode.NotFound, typeof(NotFoundObjectResult))]
    [TestCase(HttpStatusCode.Forbidden, typeof(ForbidResult))]
    [TestCase(HttpStatusCode.Conflict, typeof(ConflictObjectResult))]
    public void Failure_MapsToMatchingActionResult(HttpStatusCode status, Type expected)
    {
        // Arrange
        var result = ServiceResult<object>.Fail(status, "x");

        // Act
        var actionResult = _sut.ToActionResult(result);

        // Assert
        Assert.That(actionResult, Is.TypeOf(expected));
    }

    [Test]
    public void Ok_ReturnsOkWithData()
    {
        // Arrange
        var result = ServiceResult<string>.Ok("data");

        // Act
        var actionResult = _sut.ToActionResult(result);

        // Assert
        Assert.That(actionResult, Is.TypeOf<OkObjectResult>());
        Assert.That(((OkObjectResult)actionResult).Value, Is.EqualTo("data"));
    }
}
```

## 5. Hangfire job fixture

```csharp
[TestFixture]
public sealed class ProcessProductImagesJobTests
{
    private IProductBll _bll = null!;
    private IBlobStorage _blobs = null!;
    private ProcessProductImagesJob _sut = null!;
    private ProcessProductImagesJobDto _payload = null!;

    [SetUp]
    public void SetUp()
    {
        _bll = A.Fake<IProductBll>();
        _blobs = A.Fake<IBlobStorage>();
        _sut = new ProcessProductImagesJob(_bll, _blobs);
        _payload = new ProcessProductImagesJobDto(Guid.NewGuid(), ["tmp/a.jpg"], "user-1", IsAdmin: false);
    }

    [Test]
    public async Task BllReturnsError_Throws_AndKeepsBlobsForRetry()
    {
        // Arrange
        A.CallTo(() => _bll.ProcessProductImagesJobAsync(_payload, A<CancellationToken>._))
            .Returns(ServiceResult<bool>.Fail(HttpStatusCode.BadRequest, "Bad image."));

        // Act + Assert (for exceptions, the act sits inside the assert; always await ThatAsync)
        await Assert.ThatAsync(() => _sut.RunAsync(_payload, CancellationToken.None), Throws.TypeOf<InvalidOperationException>());
        A.CallTo(() => _blobs.DeleteManyAsync(A<IEnumerable<string>>._, A<CancellationToken>._)).MustNotHaveHappened();
    }

    [Test]
    public async Task Success_DeletesTempBlobs()
    {
        // Arrange
        A.CallTo(() => _bll.ProcessProductImagesJobAsync(_payload, A<CancellationToken>._))
            .Returns(ServiceResult<bool>.Ok(true));

        // Act
        await _sut.RunAsync(_payload, CancellationToken.None);

        // Assert
        A.CallTo(() => _blobs.DeleteManyAsync(_payload.BlobPaths, A<CancellationToken>._)).MustHaveHappenedOnceExactly();
    }
}
```

The job calls `SentrySdk` statically. In tests Sentry is not initialized, so the calls are no-ops.

## 6. Repository fixture (SQLite in-memory)

Use this for custom queries where relational behavior matters. EF InMemory is fine for simple filters.

```csharp
using Microsoft.Data.Sqlite;
using Microsoft.EntityFrameworkCore;

[TestFixture]
public sealed class ProductRepositoryTests
{
    private SqliteConnection _connection = null!;
    private AppDbContext _db = null!;
    private ProductRepository _sut = null!;

    [SetUp]
    public void SetUp()
    {
        _connection = new SqliteConnection("DataSource=:memory:");
        _connection.Open();
        _db = new AppDbContext(new DbContextOptionsBuilder<AppDbContext>().UseSqlite(_connection).Options);
        _db.Database.EnsureCreated();
        _sut = new ProductRepository(_db);
    }

    [TearDown]
    public void TearDown()
    {
        _db.Dispose();
        _connection.Dispose();
    }

    [Test]
    public async Task GetByOwnerAsync_ReturnsOnlyThatOwnersProducts()
    {
        // Arrange
        _db.Products.AddRange(
            new ProductBuilder().OwnedBy("a").Build(),
            new ProductBuilder().OwnedBy("a").Build(),
            new ProductBuilder().OwnedBy("b").Build());
        await _db.SaveChangesAsync();

        // Act
        var result = await _sut.GetByOwnerAsync("a");

        // Assert
        Assert.That(result, Has.Count.EqualTo(2));
        Assert.That(result, Has.All.Property(nameof(Product.OwnerId)).EqualTo("a"));
    }
}
```
