---
name: dotnet-testing
description: Testing conventions for .NET (NUnit 4, Arrange-Act-Assert) in N-layer solutions - behaviour tests through public APIs (HTTP endpoints via WebApplicationFactory, module interfaces), real databases (SQLite, Testcontainers for server databases), fakes only at process boundaries (FakeItEasy), bug-first failing tests, property-based tests (FsCheck) and mutation testing (Stryker.NET), plus how to run tests efficiently. Use when writing, fixing or reviewing C# tests, deciding what to test, or running dotnet test.
---

# .NET testing: behaviour first, real dependencies, NUnit + AAA

If the test project already has its own conventions (naming, base classes, builders), follow them and say so.

## Principles (the rules everything below follows)

1. **Assert behaviour from the user's point of view.** A test describes what a caller observes: the HTTP status and body, the data that is (or is not) there afterwards, the cookie that is set, the message a user sees. Not which private method ran.
2. **Couple to public APIs, not to code structure.** Test through the HTTP endpoint (`WebApplicationFactory`) or a module's public interface (`I<Name>Bll`). Renaming a Service, splitting a class or moving a repository method must not break a test as long as the behaviour is the same. Test an internal class directly only when it is a unit with its own contract and a large input space (parser, validator, token or slug generator, mapping rules), and then through its public members.
3. **Avoid mocks whenever possible.** Use the real thing: real EF Core with SQLite, the real DI container, real Identity, real Mapster, real `ResponseService`. Fakes are only for **process boundaries you do not own**: third-party HTTP APIs (Azure DevOps, GitHub), external processes (`claude`), the clock (`FakeTimeProvider`), randomness, email/SMS. Never fake your own repositories, `IUnitOfWork`, services or BLLs.
4. **Integration tests with Testcontainers.** When a dependency runs as a container (SQL Server, PostgreSQL, Redis, Azurite, a WireMock server for a third-party API), start it with Testcontainers instead of faking it. With SQLite the real SQLite engine (temp file per test host) is the database; no container needed.
5. **Reproduce a bug in a failing test before fixing it.** The test fails on the current code for the reported reason, then the fix makes it pass. It stays as a regression test.
6. **Keep logic out of the tests.** See **Arrange, Act, Assert**.
7. **Find missing tests with mutation and property-based testing.** Property-based tests (FsCheck) for code with a large input space; mutation testing (Stryker.NET) on the changed code to find behaviour no test notices.

## Stack

| Package | Purpose |
|---|---|
| `NUnit` (4.x), `NUnit3TestAdapter`, `Microsoft.NET.Test.Sdk`, `NUnit.Analyzers` | Framework, runner, assert analyzers |
| `Microsoft.AspNetCore.Mvc.Testing` | `WebApplicationFactory<Program>`: the real API in memory, tests call it over HTTP |
| `Microsoft.Data.Sqlite` | Real SQLite database per test host (temp file) |
| `Testcontainers` (+ `Testcontainers.MsSql` / `Testcontainers.PostgreSql` / …) | Real infrastructure in Docker for integration tests (needs Docker on the machine and the CI runner) |
| `Microsoft.Extensions.TimeProvider.Testing` | `FakeTimeProvider` |
| `FakeItEasy` | Fakes for process boundaries only (third-party clients, `IClaudeRunner`, …) |
| `FsCheck.NUnit` | Property-based tests |
| `dotnet-stryker` (local tool) | Mutation testing |

No Moq, NSubstitute, xUnit or FluentAssertions unless the project already uses them. No EF Core InMemory provider (it ignores constraints and transactions).

## Which tests to write

| Level | Through | Use for |
|---|---|---|
| **API behaviour tests** (default) | HTTP via `WebApplicationFactory`, real DI, real SQLite (or a Testcontainers database), fakes only at boundaries | Every endpoint: acceptance criteria, auth (401/403/404 for other users' data), validation (400 with field errors), conflicts, side effects visible through the API or the database, cookies and headers |
| **Module behaviour tests** | `I<Name>Bll` from the real container, real database | Behaviour without an endpoint: Hangfire jobs, polling, orchestration across modules. Assert results and stored state, not calls |
| **Unit tests of a contract** | The public members of a self-contained unit | Parsers, validators, generators, pure rules with many inputs; add property-based tests here |
| **User flows** (Web tasks) | The running app in a browser via Playwright MCP | The task's flow as a user does it (see the `tester` agent) |

Do not write separate tests per layer for the same behaviour (Service test + BLL test + endpoint test for one rule). One test at the highest level that shows the behaviour is enough; go lower only for many input variations.

## Layout and naming

Organize by **behaviour**, not by class:

```
<App>.Tests/
  Api/Products/
    EditProductTests.cs             PUT api/v1/products/{id}: what a client observes
  Modules/Products/
    ProcessProductImagesJobTests.cs behaviour without an endpoint
  Units/
    SlugTests.cs                    contract + property tests of a self-contained unit
  TestData/
    ApiFactory.cs                   WebApplicationFactory: temp SQLite, boundary fakes, test JWTs
    ProductBuilder.cs               builders with sensible defaults
```

- Fixture: the behaviour or endpoint (`EditProductTests`), not a class name.
- Test: `<Scenario>_<ExpectedResult>` in the caller's words: `OtherUsersProduct_Returns404`, `ValidRequest_StoresNewName`, `ExpiredRefreshCookie_Returns401AndClearsCookie`.

## Arrange, Act, Assert

Every test has three blocks marked with comments, in this order, with exactly one Act.

```csharp
[Test]
public async Task OtherUsersProduct_Returns404()
{
    // Arrange
    var product = await _api.CreateProductAsOwnerAsync("owner-1");
    var client = _api.ClientFor("user-2");

    // Act
    var response = await client.PutAsJsonAsync($"api/v1/products/{product.Id}", new { name = "New name" });

    // Assert
    using (Assert.EnterMultipleScope())
    {
        Assert.That(response.StatusCode, Is.EqualTo(HttpStatusCode.NotFound));
        Assert.That((await _api.GetProductAsync(product.Id)).Name, Is.EqualTo(product.Name));
    }
}
```

Rules:

- One behavior per test. Several asserts are fine when they describe the same outcome; group them in `Assert.EnterMultipleScope()` (NUnit 4.2+; `Assert.Multiple(() => ...)` on older versions).
- Constraint model only: `Assert.That(actual, Is.EqualTo(expected))`. No `ClassicAssert`.
- **No logic in tests:** no `if`, `switch`, loops, try/catch or computed expected values (no re-implementing the rule to get the expected result). Write the expected value literally; use `[TestCase]` for variations; move setup into builders and helpers named after what they do.
- Async tests return `Task`, never `async void`. Exceptions: `await Assert.ThatAsync(() => sut.RunAsync(...), Throws.TypeOf<InvalidOperationException>());` (always `await` it; there the Act sits inside the Assert).
- No `Thread.Sleep` and no real clock: use `FakeTimeProvider` when time matters. The database is real (see **Fakes only at the boundaries**); third-party networks and external processes are not.

## Fixture setup (important with NUnit)

NUnit creates **one fixture instance for all tests in the fixture**, so field initializers run once and state leaks between tests. Create the test host and per-test data in `[SetUp]` and dispose in `[TearDown]`:

```csharp
[TestFixture]
public sealed class EditProductTests
{
    private ApiFactory _api = null!;

    [SetUp]
    public void SetUp() => _api = new ApiFactory();       // fresh temp SQLite database, boundary fakes, test JWTs

    [TearDown]
    public void TearDown() => _api.Dispose();
}
```

If a fresh host per test is too slow for a large fixture, share one host per fixture (`[OneTimeSetUp]`) and give each test its own data (unique ids/emails) instead of resetting the database. (Alternative style: `[FixtureLifeCycle(LifeCycle.InstancePerTestCase)]`. Pick one style per project.)

## Fakes only at the boundaries

Fake a dependency only when it crosses a process or network boundary you do not own, and replace it in the test host (`ApiFactory.ConfigureTestServices`), not by building the system under test by hand.

- Third-party HTTP APIs: a fake `HttpMessageHandler` or a WireMock server (Testcontainers) that answers like the real API; assert what *your* API returns, not which URL was called (unless the request shape is the behaviour, e.g. auth headers).
- External processes (`claude`) and the clock: FakeItEasy / `FakeTimeProvider`.
- Verifying a call on a fake is fine only when the call is the behaviour (an email is sent, a job is enqueued) and it is not observable otherwise.

### FakeItEasy cheat sheet (for boundary fakes)

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

## Testcontainers

```csharp
// One container per fixture (or per test assembly with [SetUpFixture]); each test gets clean data.
private readonly PostgreSqlContainer _db = new PostgreSqlBuilder().WithImage("postgres:17-alpine").Build();

[OneTimeSetUp]
public Task StartAsync() => _db.StartAsync();

[OneTimeTearDown]
public ValueTask StopAsync() => _db.DisposeAsync();

// Point the test host at it: ApiFactory overrides ConnectionStrings:Default with _db.GetConnectionString().
```

Use it once the project uses a server database or another containerizable dependency. CI runners (`ubuntu-latest`) have Docker; on a developer machine Docker Desktop (or another Docker engine) must be running.

## Bug first

1. Write a test that reproduces the reported behaviour through the public API and **fails on the current code for that reason** (run it and show the failure).
2. Fix the code; the same test now passes without being changed.
3. Keep it as a regression test named after the behaviour, not the ticket.

## Property-based tests (FsCheck)

For units with a large input space (parsers, validators, mappers, generators): state a property that holds for all inputs and let FsCheck generate them.

```csharp
[Property]
public bool Slug_ContainsOnlyLowercaseLettersDigitsAndDashes(NonNull<string> title) =>
    Slug.From(title.Get).All(c => char.IsAsciiLetterLower(c) || char.IsAsciiDigit(c) || c == '-');
```

Good properties: round-trips (`Parse(Format(x)) == x`), invariants (output always valid), idempotence, "never throws for any input". A shrunk counter-example that fails becomes a normal `[TestCase]` regression test.

## Mutation testing (Stryker.NET)

Mutation testing changes the production code (flips conditions, removes calls) and checks that a test fails. A **surviving mutant** is behaviour no test notices: write the missing test, or explain why the mutant is equivalent.

```bash
dotnet tool install dotnet-stryker              # once, as a local tool
cd tests/<App>.Tests
dotnet stryker --since:development              # only mutate code changed on this branch
```

Run it once per task on the changed code (not in fix rounds, not on the whole solution), and report surviving mutants that matter.

## Running

**Run the full suite once per task, not after every step.** While building and fixing, run only the tests of what you touched; the main session runs the full suite once before the commit, and CI runs it on the PR.

| When | Command |
|---|---|
| Developer, tester and fix rounds | `dotnet build`, then `dotnet test --no-build --filter "FullyQualifiedName~Tests.Modules.<Module>\|FullyQualifiedName~Tests.Api.<Name>"` (the module's tests plus the endpoint's Api tests) |
| One test or case | `dotnet test --no-build --filter "Name~<Scenario>"` |
| Before the commit (main session, once) | `dotnet build` and the full `dotnet test --no-build` |
| Angular, while working | `npx ng test --watch=false --include="src/app/features/<feature>/**/*.spec.ts"` |
| Angular, before the commit (once) | `npx ng build` and the full `npx ng test --watch=false` |

A change to shared code (`Common/`, DataAccess base classes, DI extensions, `Program.cs`, middleware) affects everything: then the developer runs the full suite too.


A test run is only done when it is green and the new tests fail if the behavior they cover is removed.

## References

- `references/test-templates.md`: builders, `ResponseService`, a Hangfire job, a repository with SQLite, and FakeItEasy fixtures. The Service/BLL fixtures with faked repositories are **legacy**: for new code prefer API behaviour tests with real SQLite (principles 2–3).
