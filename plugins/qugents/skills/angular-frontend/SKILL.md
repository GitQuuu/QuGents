---
name: angular-frontend
description: Conventions for Angular frontends (v20+, signals, standalone, zoneless/OnPush) built with PrimeNG components and Tailwind CSS that talk to a versioned ASP.NET Core API, including creating the app in an empty repo. Use when creating, writing, placing or reviewing Angular code, e.g. a feature, component, page layout, PrimeNG table/form/dialog, route, API service, interceptor, or a spec file, or when wiring the frontend to a backend endpoint.
---

# Angular frontend

Targets current Angular (v22: OnPush by default, zoneless, Signal Forms and `resource`/`httpResource` stable, Vitest as test runner) with PrimeNG 22 and Tailwind CSS 4. **Check `@angular/core`, `primeng` and `tailwindcss` in `package.json` first.** If the project is on an older version or uses other conventions (NgModules, `.component.ts` file names, NgRx, Karma/Jasmine), follow the project and say so. Do not migrate unasked.

**Own repository.** The Angular app lives in its own repository (`<App>-web`, app at the repo root), separate from the .NET backend repo (`<App>`, usually a sibling folder). Frontend changes are committed and opened as pull requests in the frontend repo only. Read the backend repo for the API contract (controllers, DTOs), but never change it from a frontend task.

## Principles

1. **Thin components.** A component presents and reacts. Data access, validation rules and transformations live in services or plain functions. This mirrors thin controllers on the backend.
2. **Feature-first folders.** Group by feature, not by type. No top-level `components/`, `services/` folders.
3. **Signals for state.** `signal`, `computed`, `input()`, `output()`, `model()`. RxJS where streams are the natural fit (HTTP writes, polling, websockets); bridge with `toSignal`/`rxResource`.
4. **One API service per backend resource.** It owns URLs and DTO types. Components never build URLs or call `HttpClient` directly.
5. **The backend is the authority.** Client-side validation and role checks are for UX only. The API validates and authorizes again.
6. **PrimeNG for controls, Tailwind for layout.** UI controls come from PrimeNG; spacing, layout, typography and small visual tweaks are Tailwind utilities. No other component library or CSS framework, and almost no hand-written CSS.

## Structure

```
src/
  main.ts
  app/
    app.ts  app.html  app.config.ts  app.routes.ts
    core/                         app-wide singletons: auth, interceptors, error handling, layout
      auth/  auth-store.ts  auth-interceptor.ts  auth-guard.ts
      http/  api-url.ts  problem-details.ts  error-interceptor.ts
    shared/                       reusable presentational UI and pure helpers (no feature logic)
    features/
      products/
        products.routes.ts        lazy routes for the feature
        products-api.ts           API service: one per backend controller/resource
        product.ts                DTO types mirroring the C# DTOs
        product-list/             page (container) component
          product-list.ts  product-list.html  product-list.scss  product-list.spec.ts
        product-edit/
        product-card/             presentational component
  environments/
  styles.css                      Tailwind + PrimeNG layer order (plain CSS)
```

**Naming (Angular v20+ style guide):** hyphenated file names, file named after its main class, no type suffix: `product-list.ts` exports `ProductList`. Tests sit beside the code as `*.spec.ts`. Avoid `utils.ts`, `helpers.ts`, `common.ts`.

## Components

- Standalone (default). Do not set `standalone: true` explicitly in v19+.
- Change detection: leave the default (OnPush in v22). Never opt into `Eager` for new code.
- **Constructor injection, as in .NET.** Components, services, directives and pipes get their dependencies as `private readonly` constructor parameters, not with `inject()`:
  ```ts
  constructor(
    private readonly auth: AuthStore,
    private readonly router: Router,
  ) {}
  ```
  An `InjectionToken` uses `@Inject(API_URL) private readonly apiUrl: string`. Field initializers may use the injected dependencies because `tsconfig.json` has `"useDefineForClassFields": false` (set it when it is missing). `inject()` is used **only** where there is no constructor: functional guards, resolvers and interceptors (`CanActivateFn`, `HttpInterceptorFn`) and `provideAppInitializer`. Tests keep `TestBed.inject()`. If ESLint is configured, turn off `@angular-eslint/prefer-inject`.
  This is the product owner's choice and wins over "follow the project": in a project that still uses `inject()` in classes, new classes use constructor injection, and a class you change is converted completely (no mix inside one class).
- Order: inputs/outputs/queries, state signals, computed, constructor, methods.
- `readonly` on `input()`, `output()`, `model()` and queries. `protected` on members used only by the template.
- Templates: built-in control flow (`@if`, `@for (item of items(); track item.id)`, `@switch`, `@defer`). `[class.x]`/`[style.x]`, not `NgClass`/`NgStyle`. Keep expressions simple; move logic into `computed`.
- Event handlers named for the action (`saveProduct()`), not the trigger (`onClick()`).
- **Container vs presentational:** route-level (container) components load data and call services. Presentational components get data via `input()` and emit via `output()`, and inject nothing feature-specific.

## UI: PrimeNG + Tailwind CSS

**Packages:** `primeng`, `@primeuix/themes`, `@angular/cdk`, `tailwindcss`, `@tailwindcss/postcss`, `postcss`, `tailwindcss-primeui`. Optional: `primeicons` for `pi pi-*` icons.

**License (check before adding or upgrading):** PrimeNG 22+ needs a PrimeUI license key in `providePrimeNG({ license })`, otherwise every page shows an "invalid license" banner. The Community License is free for organizations under USD 1M revenue, under 5 developers and under 10 employees (renewed yearly); others need a paid per-developer Commercial license. PrimeNG 21 and older stay MIT but only support Angular 21. Never hide the banner with CSS or code; configure the key. The key is public by design and lives in `environment.ts`.

**Setup** (full files in the templates):
- `.postcssrc.json` with the `@tailwindcss/postcss` plugin.
- `src/styles.css` (plain CSS; Tailwind v4 does not run through Sass): `@layer theme, base, primeng, components, utilities;`, then `@import "tailwindcss";` and `@import "tailwindcss-primeui";`, plus `@custom-variant dark (&:where(.app-dark, .app-dark *));`.
- `providePrimeNG({ theme: { preset: Aura, options: { darkModeSelector: '.app-dark', cssLayer: { name: 'primeng', order: 'theme, base, primeng' } } }, license })` in `app.config.ts`. Without `cssLayer`, PrimeNG styles are unlayered and beat every Tailwind utility.
- `MessageService` and `ConfirmationService` provided once in `app.config.ts`; `<p-toast />` once in `app.html`.

**Using components:**
- Import standalone classes per component: `Button` from `primeng/button`, `InputText`, `InputNumber`, `Select`, `DatePicker`, `Dialog`, `ConfirmDialog`, `Toast`, `Tag`; `TableModule` for `p-table`.
- Templates use template refs: `<ng-template #header>`, `#body let-row`, `#emptymessage`. Not the old `pTemplate="..."`.
- Events are PrimeNG outputs, e.g. `(onClick)` on `p-button`.
- **Pick the PrimeNG component first** (table, select, date picker, dialog, toast, confirm, tag, skeleton, progress bar) before building your own.
- **Styling order:** component props (`severity`, `size`, `outlined`, `text`) → Tailwind classes via `class`/`styleClass` → `pt` (pass-through) for inner elements → theme preset (`definePreset`) for brand-wide changes. Never `::ng-deep`, `!important` or overriding `.p-*` classes in CSS.
- Colors: prefer the theme-aware `primary-*` and `surface-*` utilities from `tailwindcss-primeui` over raw palette colors, so dark mode and the preset keep working.
- Feedback: success and API errors as toasts (`MessageService.add`), destructive actions behind `ConfirmationService.confirm`, loading states with `[loading]` on `p-table`/`p-button` or `p-skeleton`.
- Lists: `p-table` with `dataKey`; server paging/sorting via `[lazy]="true"` and `(onLazyLoad)` mapped to the API's paging parameters.
- Accessibility: every input has a `<label for>`/`inputId` pair; icon-only buttons get `ariaLabel`.

## Talking to the .NET API

- Base URL from `environment.apiUrl` plus the version segment: `${apiUrl}/api/v1/products`. Keep it in one place (`core/http/api-url.ts`).
- DTO types mirror the C# DTOs (JSON is camelCase). When a backend DTO changes, update the TS type in the same task. Dates arrive as ISO strings; convert at the edge if needed.
- **Reads:** `httpResource()` (or `rxResource`) in the component or a store service. It exposes `value()`, `isLoading()`, `error()` and `reload()`.
- **Writes:** methods on the API service returning `Observable<T>`. The component subscribes once, or uses `firstValueFrom` in an `async` handler.
- **Errors:** the API returns `ProblemDetails` (`{ status, title, errors? }`). A functional error interceptor maps them to a typed `ApiError`. 400 with `errors` maps to field errors on the form; 401 triggers refresh or login; 403 shows "not allowed"; 404/409 are shown in context. Never swallow errors silently.
- **Auth** (backend: `dotnet-identity-jwt`): `AuthStore` keeps the JWT access token **in memory only** (signals) and exposes `isLoggedIn`, `userId`, `roles`, `hasRole()`. The refresh token is an HttpOnly cookie the browser sends to `/api/v1/auth/*`, so login/refresh/logout use `withCredentials: true`. `provideAppInitializer` calls `refresh()` once to restore the session after a reload. A functional `HttpInterceptorFn` adds the bearer token and, on 401 outside `/auth/`, waits for one shared refresh and retries. Guards are functional (`CanActivateFn`) and check login/roles for UX only. Never put tokens in `localStorage`/`sessionStorage`.
- **Background jobs (202 + `jobId`):** the API service returns the `jobId`; a small polling helper (`timer` + `switchMap` to the jobs endpoint, `takeWhile` until done/failed) drives a status signal in the component.

Register HTTP in `app.config.ts`: `provideHttpClient(withInterceptors([authInterceptor, errorInterceptor]))`. In v22 `fetch` is the default backend.

## Forms

- v22+: **Signal Forms** (`form()` from `@angular/forms/signals`, `[formField]` in the template, validators like `required`, `email`, `minLength` in the schema callback). `[formField]` works on `pInputText` and on PrimeNG form components such as `p-inputnumber`, `p-select`, `p-datepicker`.
- Older versions or existing reactive forms: **typed reactive forms** (`FormBuilder.nonNullable`). Do not mix both in one form.
- Mirror the backend's DataAnnotations (required, max length, ranges) for UX, and map server-side 400 field errors back onto the form.

## Routing

- Lazy per feature: `{ path: 'products', loadChildren: () => import('./features/products/products.routes') }`, and `loadComponent` for single pages.
- Route params via `input()` with `withComponentInputBinding()`.
- Guards and resolvers are functions.

## Security

- Never call `bypassSecurityTrust*` on data that came from users or the API. Never bind untrusted HTML with `[innerHTML]` unless sanitized.
- No secrets in `environment.ts`; it ships to the browser. (The PrimeUI license key is public by design and belongs there.)
- Do not log tokens or personal data to the console.

## Testing

- Vitest (default from v21) with `TestBed`. Follow the project's runner if it still uses Karma/Jasmine.
- Arrange/Act/Assert, as in the backend tests. Assert what the **user sees and does**: find elements by role, label or visible text and interact like a user (click, type), not by CSS classes, component internals or private signals. No logic in specs.
- Fake only the boundary: HTTP with `HttpTestingController`. Use the real services, stores and child components otherwise.
- Property-based tests with `fast-check` for pure functions with many inputs (formatters, validators, parsers); mutation testing with StrykerJS (`npx stryker run`) when it is set up.
- Real user flows are checked in a browser with Playwright MCP by the tester (see the `tester` agent).
- HTTP: `provideHttpClient()` + `provideHttpClientTesting()` and `HttpTestingController` (`expectOne`, `flush`, `verify`).
- Test API services (URL, method, body, error mapping), stores/services with logic, and container components' behavior. Presentational components: only when they hold logic.
- Component specs that render PrimeNG need `providePrimeNG({ theme: 'none' })` plus `MessageService`/`ConfirmationService` in the TestBed providers. Flush pending `httpResource` requests with `HttpTestingController` before `await fixture.whenStable()`.

## New app in an empty repo

A Web task that bootstraps the app ("set up the Angular project") starts in a repo that holds at most `README.md`, `.gitignore`, `CLAUDE.md`, `.github/`, `.claude/`, `scripts/` or an IDE folder. Create the app **in the repo root**, never in a subfolder. Three things make `ng new` fail when run by an agent, so follow these steps exactly (Git Bash):

1. **Check Node.** `node -v` must be a version the current Angular CLI supports (an even LTS release). If it is not, stop and report it; do not install Node yourself.
2. **Move conflicting files aside.** `ng new` creates `README.md`, `.gitignore` and `.editorconfig` and stops with "A merge conflicted on path /README.md" if they exist:
   ```bash
   for f in README.md .gitignore .editorconfig; do [ -f "$f" ] && mv "$f" "$f.pre-ng"; done
   ```
3. **Create the app without prompts.** The CLI asks about analytics, AI tools and more; without a terminal the questions hang or fail, so switch them all off. Name the app in kebab-case after the repo (`ChinaBoxSPA` → `china-box-spa`):
   ```bash
   export NG_CLI_ANALYTICS=false
   npx -y @angular/cli@latest new <app-name> --directory . --style=css --routing --ssr=false --skip-git --skip-install --interactive=false --defaults
   ```
4. **Install with a long timeout.** `npm install` can take several minutes. Run it as its own Bash call with `timeout: 600000`; with the default timeout it is killed halfway:
   ```bash
   npm install
   npm install tailwindcss @tailwindcss/postcss postcss               # always
   npm install primeng @primeuix/themes @angular/cdk tailwindcss-primeui   # only when a PrimeNG component is used
   ```
5. **Merge the files back.** Keep the repo's own `README.md` (rewrite it if the task asks for one), append the lines of `.gitignore.pre-ng` that Angular's `.gitignore` lacks, then delete the `*.pre-ng` files.
6. **Constructor injection:** add `"useDefineForClassFields": false` to `compilerOptions` in `tsconfig.json` (see Components). If `ng lint` is added later, turn off `@angular-eslint/prefer-inject`.
7. **Set up styling** from `references/angular-templates.md`: `.postcssrc.json`, `src/styles.css`, and `providePrimeNG` only if PrimeNG was installed.
8. **Prove it works:** `npx ng build` and `npx ng test --watch=false` pass. Commit `package-lock.json`.

HTTPS for the dev server (needed only when the app calls an API with a refresh cookie) is in the `dotnet-new-solution` skill, section 5.

## Commands

```bash
npm ci
npx ng build
npx ng test --watch=false
npx ng serve         # https://localhost:4200 (HTTPS needed for the refresh cookie, see dotnet-new-solution)
npx ng lint          # if configured
```

Development URLs: Angular on `https://localhost:4200`, API on `https://localhost:5001` (`environment.apiUrl`).

## References

- `references/angular-templates.md`: skeletons for PrimeNG + Tailwind setup (`app.config.ts`, `styles.css`, `.postcssrc.json`), an API service, DTO types, a list page with `p-table` + confirm + toast, a presentational component, an edit form with Signal Forms on PrimeNG inputs, routes, interceptors, ProblemDetails mapping, job polling, and specs. Read it before writing new frontend code.
