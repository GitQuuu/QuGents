---
name: design-principles
description: Design principles as guidelines, not rules - SOLID, DRY, KISS/YAGNI - applied to an ASP.NET Core N-layer modular monolith with vertical slices, Repository + Unit of Work, and an Angular frontend. Covers when to apply each principle, when to bend it, and how the principles relate to the architecture patterns in dotnet-n-layer. Use when designing or reviewing a class, interface or abstraction, deciding whether to extract shared code, or weighing a refactoring.
---

# Design principles: guidelines, not rules

These principles help you choose between designs. They are **guidelines**: apply them when they make the code easier to change, and bend them when following them would add indirection without a real benefit. When you bend one, it should be a conscious choice you can explain in one sentence (in the hand-off or PR). The project's architecture rules in `dotnet-n-layer` (layers, thin controllers, slices, UoW) are rules; the principles below explain *why* those rules exist and guide everything they do not decide.

## How the patterns fit together

| Pattern | What it gives us | Where it lives |
|---|---|---|
| **N-layer** (logical layers) | Api → Services → DataAccess, dependencies point one way. Each layer has one kind of responsibility | `dotnet-n-layer`: projects and dependency direction |
| **Modular monolith** | Business areas (Auth, Projects, Users …) are modules with clear boundaries inside one deployable. A module talks to another only through its `I<Other>Bll`/`I<Other>Service` interface. A module can later be extracted into its own API | `dotnet-n-layer`: modules, extraction boundary |
| **Vertical slice architecture** | Each use case is a folder (`CreateProjectAsync/`) through controller, BLL and service. Changing one use case touches one slice | `dotnet-n-layer`: vertical slices |
| **Repository + Unit of Work** | Repositories hide EF Core queries per aggregate; the UoW shares one `DbContext` and commits once per use case (in the BLL) | `dotnet-n-layer`: layer responsibilities, templates |
| **N-tier** (physical tiers: browser, API host, database) | Deployment, networking between tiers | DevOps (`devops-github-actions`), not the developer's concern |

Slices and layers are not in conflict: the layers say *which kind* of code goes where, the slice says *which use case* it belongs to.

## SOLID

**S: Single responsibility.** A class has one reason to change. On this stack: a controller handles HTTP, a BLL orchestrates one use case, a Service does domain operations, a repository does queries. Partial classes per operation keep a growing Service readable without splitting it into tiny classes. Bend it: a small helper used by one slice can live in that slice's file.

**O: Open/closed.** Add behaviour by adding code, not by editing stable code. New use case = new slice; new API version = new `V2` folder; new status code = a new case in `ResponseService`. Bend it: do not build plugin points or strategy hierarchies for variations that do not exist yet (YAGNI).

**L: Liskov substitution.** An implementation must honour its interface's contract: same `ServiceResult` statuses, no new exceptions for expected failures, no "not supported" methods. A `<Name>WebService` that replaces a BLL after extraction must behave like it. If a subtype needs to refuse half the interface, the interface is wrong.

**I: Interface segregation.** Small, role-shaped interfaces: `I<Name>Bll` for what the Api needs, `I<Name>Service` for what other modules need, `IUnitOfWork` exposing repositories. Do not add methods to an interface "in case". Bend it: one `I<Name>Bll` per module is fine; do not split it per operation.

**D: Dependency inversion.** Depend on abstractions you own: BLLs on `I<Name>Service` and `IUnitOfWork`, external systems behind interfaces (`IAzureBoardsClient`, `IGitHubClient`, `IClaudeRunner`) so tests can fake them. Inject through constructors. Bend it: no interface for a class that has one implementation, no external dependency and no need to be faked (DTOs, records, pure static helpers, options classes).

## DRY: one source of truth for knowledge

DRY is about **knowledge**, not about lines that look alike.

- A business rule, validation rule, constant or mapping lives in **one** place: rules in the Service, configuration keys and policy names as constants next to their extension (`CorsExtensions.WebPolicy`), shared types in the module that owns them.
- Code that looks similar but changes for different reasons is **not** duplication. Do not merge it:
  - two slices whose handling happens to be alike today;
  - `V1` and `V2` of an endpoint: different contracts, kept apart for backward compatibility;
  - request, DTO and response types with the same fields: they belong to different layers and change independently;
  - test arrangement that repeats for readability (prefer a clear test over a clever shared helper).
- **Rule of three:** extract shared code when the same knowledge appears the third time and the copies would always change together. Put it where the owner is (module service, `Common/`, an extension method), not in a "Utils" dump.

## KISS and YAGNI: the counterweight

- Prefer the simplest design that meets the acceptance criteria. No generic frameworks, base-class hierarchies, events or caching that the task does not need.
- Every abstraction must pay for itself now: a test seam, a real second implementation, or a boundary the architecture requires.
- When a principle pulls toward more indirection and KISS pulls the other way, choose readability and mention the trade-off.

## In a review or hand-off

When you apply or bend a principle in a non-obvious way, say it in one line, e.g. "Kept the two slices separate (DRY bent: they change for different reasons)" or "No interface for `SlugGenerator` (DIP bent: pure function, nothing to fake)". Do not cite principles to justify churn: a refactoring for a principle alone belongs under "Noticed", not in the task.
