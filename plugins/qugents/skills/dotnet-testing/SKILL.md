---
name: dotnet-testing
description: Unit-testing conventions for .NET with NUnit 4, Arrange-Act-Assert and FakeItEasy, for N-layer solutions (Service, BLL, ResponseService, Hangfire jobs, repositories). Use when writing, fixing or reviewing C# tests, deciding what to test, or running dotnet test.
---

# .NET testing: NUnit + AAA + FakeItEasy

If the test project already has its own conventions (naming, base classes, builders), follow them and say so.

## Stack

| Package | Purpose |
|---|---|
| `NUnit` (4.x), `NUnit3TestAdapter`, `Microsoft.NET.Test.Sdk` | Framework and runner |
| `NUnit.Analyzers` | Catches wrong assert usage at build time |
| `FakeItEasy` (+ `FakeItEasy.Analyzer.CSharp`) | Fakes for interfaces |
| `Microsoft.EntityFrameworkCore.InMemory` or `Microsoft.Data.Sqlite` | Repository/query tests |

No Moq, NSubstitute, xUnit or FluentAssertions unless the project already uses them.

## Layout

The test project mirrors the Services modules. One fixture per operation, one file per fixture:

```
<App>.Tests/
  Modules/Products/
    EditProductAsyncTests.cs           ProductService + ProductBll for EditProductAsync
    ProcessProductImagesJobTests.cs
  Api/
    ResponseServiceTests.cs
  DataAccess/
    ProductRepositoryTests.cs          only custom queries
  TestData/
    ProductBuilder.cs                  builders for entities/DTOs with sensible defaults
```

## Naming

- Fixture: `<Operation>Tests` (or `<Class>Tests` for non-slice classes).
- Test: `<Scenario>_<ExpectedResult>`, because the fixture already names the operation. Examples: `ProductMissing_ReturnsNotFound`, `ServiceFails_DoesNotCommit`, `Saved_ClearsCache`.

## Arrange, Act, Assert

Every test has three blocks marked with comments, in this order, with exactly one Act.

```csharp
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
```

Rules:

- One behavior per test. Several asserts are fine when they describe the same outcome; group them in `Assert.EnterMultipleScope()` (NUnit 4.2+; `Assert.Multiple(() => ...)` on older versions).
- Constraint model only: `Assert.That(actual, Is.EqualTo(expected))`. No `ClassicAssert`.
- No logic in tests: no `if`, loops or try/catch. Use `[TestCase]` for variations.
- Async tests return `Task`, never `async void`. Exceptions: `await Assert.ThatAsync(() => sut.RunAsync(...), Throws.TypeOf<InvalidOperationException>());` (always `await` it; there the Act sits inside the Assert).
- No `Thread.Sleep`, real database, network, file system or clock. Fake `TimeProvider` if time matters.

## Fixture setup (important with NUnit)

NUnit creates **one fixture instance for all tests in the fixture**. Field initializers therefore run once and fakes would leak state between tests. Create fakes and the system under test in `[SetUp]`:

```csharp
[TestFixture]
public sealed class EditProductAsyncTests
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
}
```

(Alternative: `[FixtureLifeCycle(LifeCycle.InstancePerTestCase)]` on the fixture. Pick one style per project.)

## FakeItEasy cheat sheet

```csharp
var repo = A.Fake<IProductRepository>();

// Configure (async methods: pass the value, FakeItEasy wraps it in a completed Task)
A.CallTo(() => repo.GetByIdAsync(A<object>._, A<CancellationToken>._)).Returns(product);
A.CallTo(() => repo.GetByIdAsync(A<object>.That.Matches(id => (Guid)id == productId), A<CancellationToken>._)).Returns(product);
A.CallTo(() => uow.SaveChangesAsync(A<CancellationToken>._)).Returns(1);
A.CallTo(() => blob.DeleteManyAsync(A<IEnumerable<string>>._, A<CancellationToken>._)).ThrowsAsync(new IOException());
A.CallTo(() => clock.GetUtcNow()).Returns(new DateTimeOffset(2026, 1, 1, 0, 0, 0, TimeSpan.Zero));

// Verify interactions that ARE the behavior (commit, cache clear, enqueue, cleanup)
A.CallTo(() => uow.SaveChangesAsync(A<CancellationToken>._)).MustHaveHappenedOnceExactly();
A.CallTo(() => uow.SaveChangesAsync(A<CancellationToken>._)).MustNotHaveHappened();
A.CallTo(() => cache.ClearAll()).MustHaveHappenedOnceExactly();

// Order matters (commit before cache clear)
A.CallTo(() => uow.SaveChangesAsync(A<CancellationToken>._)).MustHaveHappenedOnceExactly()
    .Then(A.CallTo(() => cache.ClearAll()).MustHaveHappenedOnceExactly());
```

- `A<T>._` (or `A<T>.Ignored`) for any value; `A<T>.That.Matches(...)` for a condition.
- Unconfigured calls return defaults (dummies, empty strings, completed tasks). Configure what the test depends on explicitly anyway.
- Do not verify every call. Over-verification makes tests break on harmless refactors.

## What to test per layer

| Layer | Test | Typical cases |
|---|---|---|
| **Service** | Business rules and the `ServiceResult` it returns | Not found → 404, not owner → 403, invalid state → 409/400, happy path maps entity → DTO and stages the change |
| **BLL** | Orchestration | Service fails → returns the failure and **does not commit**; success → commits once, checks row count, clears cache after commit; calls to other modules' interfaces |
| **ResponseService** | Every status-code mapping | `[TestCase(HttpStatusCode.NotFound, typeof(NotFoundObjectResult))]` per case |
| **Jobs** | Failure handling | BLL status ≥ 400 → throws; exception → rethrown; temp blobs deleted only after success |
| **Repositories** | Only custom queries | Filters, includes, `AsNoTracking`. Prefer SQLite in-memory when relational behavior matters (EF InMemory ignores constraints and transactions) |
| **Controllers** | Usually nothing | They are thin. Test only if an action has logic, and then the logic belongs in the BLL instead |

Derive cases from the task's acceptance criteria first, then from every `ServiceResult.Fail` branch in the code.

## Running

```bash
dotnet test
dotnet test --filter "FullyQualifiedName~Modules.Products"
dotnet test --filter "Name~ProductMissing"
```

A test run is only done when it is green and the new tests fail if the behavior they cover is removed.

## References

- `references/test-templates.md`: complete fixtures for a Service, a BLL, `ResponseService`, a Hangfire job, a repository with SQLite in-memory, and a test-data builder.
