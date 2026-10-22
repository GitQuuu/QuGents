# Angular templates

Skeletons for one feature (`products`) talking to the .NET API from the `dotnet-n-layer` skill (`api/v1/products`, `ProblemDetails`, 202 + `jobId`). UI is PrimeNG + Tailwind CSS. Written for Angular 22 and PrimeNG 22 (verified: builds with `strictTemplates`, the spec passes, and Signal Forms bind to PrimeNG inputs at runtime). On older versions, keep the shape and adapt the APIs.

## Contents

1. App config, styles and environment (PrimeNG + Tailwind)
2. ProblemDetails → `ApiError` (error interceptor)
3. Auth: store, interceptor, guard, session restore
4. DTO types
5. API service
6. Feature routes
7. Container component (list: `httpResource` + `p-table` + toast + confirm)
8. Presentational component
9. Edit component (Signal Forms + PrimeNG inputs + write)
10. Job polling (202 + `jobId`)
11. Spec (Vitest + `HttpTestingController`)

---

## 1. App config, styles and environment

`src/app/app.config.ts`

```ts
import { ApplicationConfig } from '@angular/core';
import { provideRouter, withComponentInputBinding } from '@angular/router';
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { providePrimeNG } from 'primeng/config';
import { ConfirmationService, MessageService } from 'primeng/api';
import Aura from '@primeuix/themes/aura';
import { routes } from './app.routes';
import { errorInterceptor } from './core/http/error-interceptor';
import { authInterceptor } from './core/auth/auth-interceptor';
import { environment } from '../environments/environment';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes, withComponentInputBinding()),
    // Order matters: authInterceptor sits closest to the backend, so it sees the raw 401
    // and can refresh before errorInterceptor turns the response into an ApiError.
    provideHttpClient(withInterceptors([errorInterceptor, authInterceptor])),
    providePrimeNG({
      theme: {
        preset: Aura, // brand colors: definePreset(Aura, { semantic: { primary: { ... } } })
        options: {
          darkModeSelector: '.app-dark', // same selector as Tailwind's dark variant in styles.css
          cssLayer: { name: 'primeng', order: 'theme, base, primeng' }, // lets Tailwind utilities win
        },
      },
      license: environment.primengLicense, // PrimeNG 22+: required, otherwise a license banner shows
    }),
    MessageService, // app-wide toasts; <p-toast /> lives once in app.html
    ConfirmationService, // confirm dialogs for destructive actions
  ],
};
```

`src/styles.css` (plain CSS, not SCSS: Tailwind v4 does not run through Sass)

```css
/* PrimeNG component styles sit between Tailwind's base and utilities,
   so utility classes override PrimeNG without !important. */
@layer theme, base, primeng, components, utilities;

@import "tailwindcss";
@import "tailwindcss-primeui"; /* surface/primary colors and p-* variants from the PrimeNG theme */

/* Tailwind's dark: variant follows PrimeNG's darkModeSelector */
@custom-variant dark (&:where(.app-dark, .app-dark *));
```

`.postcssrc.json` (project root)

```json
{
  "plugins": {
    "@tailwindcss/postcss": {}
  }
}
```

`src/environments/environment.ts`

```ts
export const environment = {
  apiUrl: 'http://localhost:5000',
  primengLicense: '', // PrimeUI license key. Public by design (verified offline in the browser), not a secret.
};
```

`src/app/app.html`

```html
<p-toast />
<router-outlet />
```

(`App` imports `RouterOutlet` and `Toast` from `primeng/toast`.)

`src/app/core/http/api-url.ts`

```ts
import { environment } from '../../../environments/environment';

export const API_VERSION = 'v1';

export function apiUrl(path: string): string {
  return `${environment.apiUrl}/api/${API_VERSION}/${path}`;
}
```

## 2. ProblemDetails → `ApiError`

`src/app/core/http/problem-details.ts`

```ts
export interface ProblemDetails {
  status?: number;
  title?: string;
  detail?: string;
  errors?: Record<string, string[]>; // ASP.NET Core validation errors
}

export class ApiError extends Error {
  constructor(
    readonly status: number,
    message: string,
    readonly fieldErrors: Record<string, string[]> = {},
  ) {
    super(message);
  }
}
```

`src/app/core/http/error-interceptor.ts`

```ts
import { HttpErrorResponse, HttpInterceptorFn } from '@angular/common/http';
import { catchError, throwError } from 'rxjs';
import { ApiError, ProblemDetails } from './problem-details';

export const errorInterceptor: HttpInterceptorFn = (req, next) =>
  next(req).pipe(
    catchError((err: unknown) => {
      if (!(err instanceof HttpErrorResponse)) return throwError(() => err);
      const problem = (err.error ?? {}) as ProblemDetails;
      return throwError(
        () => new ApiError(err.status, problem.title ?? problem.detail ?? err.message, problem.errors ?? {}),
      );
    }),
  );
```

## 3. Auth: store, interceptor, guard, session restore

Matches the `dotnet-identity-jwt` skill: access token in the JSON body (kept in memory only), refresh token in an HttpOnly cookie on `/api/v1/auth`, so auth calls use `withCredentials: true`.

`src/app/core/auth/auth-store.ts`

```ts
import { HttpClient } from '@angular/common/http';
import { Injectable, computed, inject, signal } from '@angular/core';
import { Observable, catchError, finalize, map, shareReplay, throwError } from 'rxjs';
import { apiUrl } from '../http/api-url';

interface AccessTokenResponse {
  accessToken: string;
  expiresAt: string;
}

@Injectable({ providedIn: 'root' })
export class AuthStore {
  private readonly http = inject(HttpClient);
  private readonly token = signal<string | null>(null);
  private refreshInFlight: Observable<string> | null = null;

  readonly accessToken = this.token.asReadonly();
  readonly isLoggedIn = computed(() => this.token() !== null);
  private readonly claims = computed(() => decodeClaims(this.token()));
  readonly userId = computed(() => (this.claims()?.['sub'] as string | undefined) ?? null);
  readonly roles = computed(() => toArray(this.claims()?.['role']));

  hasRole(role: string): boolean {
    return this.roles().includes(role);
  }

  login(email: string, password: string): Observable<void> {
    return this.http
      .post<AccessTokenResponse>(apiUrl('auth/login'), { email, password }, { withCredentials: true })
      .pipe(map((r) => this.token.set(r.accessToken)));
  }

  /** One shared request, even when many calls get a 401 at the same time. */
  refresh(): Observable<string> {
    this.refreshInFlight ??= this.http
      .post<AccessTokenResponse>(apiUrl('auth/refresh'), null, { withCredentials: true })
      .pipe(
        map((r) => {
          this.token.set(r.accessToken);
          return r.accessToken;
        }),
        catchError((err: unknown) => {
          this.token.set(null);
          return throwError(() => err);
        }),
        finalize(() => (this.refreshInFlight = null)),
        shareReplay(1),
      );
    return this.refreshInFlight;
  }

  logout(): Observable<void> {
    return this.http
      .post<void>(apiUrl('auth/logout'), null, { withCredentials: true })
      .pipe(finalize(() => this.token.set(null)));
  }
}

function decodeClaims(token: string | null): Record<string, unknown> | null {
  if (!token) return null;
  try {
    const payload = token.split('.')[1].replace(/-/g, '+').replace(/_/g, '/');
    return JSON.parse(atob(payload)) as Record<string, unknown>;
  } catch {
    return null;
  }
}

function toArray(value: unknown): string[] {
  if (Array.isArray(value)) return value.map(String);
  return typeof value === 'string' ? [value] : [];
}
```

Decoding the token is for UX (show/hide by role). The API validates the signature and enforces roles.

`src/app/core/auth/auth-interceptor.ts`

```ts
import { HttpErrorResponse, HttpInterceptorFn, HttpRequest } from '@angular/common/http';
import { inject } from '@angular/core';
import { catchError, switchMap, throwError } from 'rxjs';
import { AuthStore } from './auth-store';

const withToken = (req: HttpRequest<unknown>, token: string | null) =>
  token ? req.clone({ setHeaders: { Authorization: `Bearer ${token}` } }) : req;

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const auth = inject(AuthStore);
  const isAuthCall = req.url.includes('/auth/'); // login/refresh/logout: never retry with a refresh
  return next(withToken(req, auth.accessToken())).pipe(
    catchError((err: unknown) => {
      if (err instanceof HttpErrorResponse && err.status === 401 && !isAuthCall) {
        return auth.refresh().pipe(switchMap((token) => next(withToken(req, token))));
      }
      return throwError(() => err);
    }),
  );
};
```

Restore the session on page load (the access token lives only in memory). In `app.config.ts` providers:

```ts
import { inject, provideAppInitializer } from '@angular/core';
import { firstValueFrom } from 'rxjs';
import { AuthStore } from './core/auth/auth-store';

// inside providers: [...]
provideAppInitializer(() => firstValueFrom(inject(AuthStore).refresh()).catch(() => undefined)),
```

`src/app/core/auth/auth-guard.ts`

```ts
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';
import { AuthStore } from './auth-store';

/** UX only. The API enforces authentication and roles again. */
export const authGuard: CanActivateFn = () =>
  inject(AuthStore).isLoggedIn() || inject(Router).createUrlTree(['/login']);

export const roleGuard =
  (role: string): CanActivateFn =>
  () =>
    inject(AuthStore).hasRole(role) || inject(Router).createUrlTree(['/login']);
```

`src/app/core/auth/auth-store.spec.ts`

```ts
import { afterEach, beforeEach, describe, expect, it } from 'vitest';
import { TestBed } from '@angular/core/testing';
import { provideHttpClient } from '@angular/common/http';
import { HttpTestingController, provideHttpClientTesting } from '@angular/common/http/testing';
import { firstValueFrom } from 'rxjs';
import { AuthStore } from './auth-store';

const tokenWith = (claims: object) => `x.${btoa(JSON.stringify(claims))}.y`;

describe('AuthStore', () => {
  let store: AuthStore;
  let http: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({ providers: [provideHttpClient(), provideHttpClientTesting()] });
    store = TestBed.inject(AuthStore);
    http = TestBed.inject(HttpTestingController);
  });

  afterEach(() => http.verify());

  it('shares one refresh request between concurrent callers', async () => {
    // Arrange
    const token = tokenWith({ sub: 'u1', role: ['Admin', 'User'] });

    // Act
    const first = firstValueFrom(store.refresh());
    const second = firstValueFrom(store.refresh());
    const req = http.expectOne((r) => r.url.endsWith('/api/v1/auth/refresh'));
    req.flush({ accessToken: token, expiresAt: '2030-01-01T00:00:00Z' });

    // Assert
    expect(req.request.withCredentials).toBe(true);
    expect(await first).toBe(token);
    expect(await second).toBe(token);
    expect(store.userId()).toBe('u1');
    expect(store.hasRole('Admin')).toBe(true);
  });

  it('clears the token when refresh fails', async () => {
    // Arrange
    const result = firstValueFrom(store.refresh()).catch(() => 'failed');

    // Act
    http.expectOne((r) => r.url.endsWith('/api/v1/auth/refresh')).flush(null, { status: 401, statusText: 'Unauthorized' });

    // Assert
    expect(await result).toBe('failed');
    expect(store.isLoggedIn()).toBe(false);
  });
});
```

## 4. DTO types

`src/app/features/products/product.ts`: mirrors the C# DTOs (camelCase).

```ts
export interface ProductDto {
  id: string;
  name: string;
  price: number;
}

export interface EditProductRequest {
  name: string;
  price: number;
}

export interface JobAccepted {
  jobId: string;
}
```

## 5. API service

`src/app/features/products/products-api.ts`: the only place that knows the products URLs.

```ts
import { HttpClient } from '@angular/common/http';
import { Injectable, inject } from '@angular/core';
import { Observable } from 'rxjs';
import { apiUrl } from '../../core/http/api-url';
import { EditProductRequest, JobAccepted, ProductDto } from './product';

@Injectable({ providedIn: 'root' })
export class ProductsApi {
  private readonly http = inject(HttpClient);
  private readonly baseUrl = apiUrl('products');

  /** For httpResource in a component: `httpResource<ProductDto[]>(() => api.listUrl(page()))`. */
  listUrl(page: number): string {
    return `${this.baseUrl}?page=${page}`;
  }

  byIdUrl(id: string): string {
    return `${this.baseUrl}/${id}`;
  }

  edit(id: string, body: EditProductRequest): Observable<ProductDto> {
    return this.http.put<ProductDto>(`${this.baseUrl}/${id}`, body);
  }

  archive(id: string): Observable<ProductDto> {
    return this.http.post<ProductDto>(`${this.baseUrl}/${id}/archive`, null);
  }

  uploadImages(id: string, files: File[]): Observable<JobAccepted> {
    const form = new FormData();
    files.forEach((file) => form.append('files', file));
    return this.http.post<JobAccepted>(`${this.baseUrl}/${id}/images`, form);
  }
}
```

## 6. Feature routes

`src/app/features/products/products.routes.ts`

```ts
import { Routes } from '@angular/router';
import { roleGuard } from '../../core/auth/auth-guard';

export default [
  { path: '', loadComponent: () => import('./product-list/product-list').then((m) => m.ProductList) },
  {
    path: ':id/edit',
    canActivate: [roleGuard('Admin')],
    loadComponent: () => import('./product-edit/product-edit').then((m) => m.ProductEdit),
  },
] satisfies Routes;
```

In `app.routes.ts`: `{ path: 'products', loadChildren: () => import('./features/products/products.routes') }`.

## 7. Container component

`src/app/features/products/product-list/product-list.ts`

```ts
import { Component, computed, inject, signal } from '@angular/core';
import { httpResource } from '@angular/common/http';
import { RouterLink } from '@angular/router';
import { firstValueFrom } from 'rxjs';
import { ConfirmationService, MessageService } from 'primeng/api';
import { Button } from 'primeng/button';
import { ConfirmDialog } from 'primeng/confirmdialog';
import { TableModule } from 'primeng/table';
import { ApiError } from '../../../core/http/problem-details';
import { ProductsApi } from '../products-api';
import { ProductDto } from '../product';

@Component({
  selector: 'app-product-list',
  imports: [RouterLink, Button, ConfirmDialog, TableModule],
  templateUrl: './product-list.html',
})
export class ProductList {
  private readonly api = inject(ProductsApi);
  private readonly messages = inject(MessageService);
  private readonly confirmation = inject(ConfirmationService);

  protected readonly page = signal(1);
  protected readonly products = httpResource<ProductDto[]>(() => this.api.listUrl(this.page()));
  protected readonly productList = computed(() => this.products.value() ?? []);

  protected showNextPage(): void {
    this.page.update((p) => p + 1);
  }

  protected confirmArchive(product: ProductDto): void {
    this.confirmation.confirm({
      header: 'Archive product',
      message: `Archive "${product.name}"?`,
      accept: () => this.archiveProduct(product.id),
    });
  }

  private async archiveProduct(id: string): Promise<void> {
    try {
      await firstValueFrom(this.api.archive(id));
      this.messages.add({ severity: 'success', summary: 'Product archived' });
      this.products.reload();
    } catch (err) {
      this.messages.add({
        severity: 'error',
        summary: 'Could not archive the product',
        detail: err instanceof ApiError ? err.message : undefined,
      });
    }
  }
}
```

`product-list.html`: PrimeNG for controls, Tailwind for layout.

```html
<section class="flex flex-col gap-4 p-4">
  <h1 class="text-2xl font-semibold">Products</h1>

  @if (products.error()) {
    <p class="text-red-600" role="alert">Could not load products.</p>
    <p-button label="Try again" severity="secondary" (onClick)="products.reload()" />
  } @else {
    <p-table [value]="productList()" [loading]="products.isLoading()" dataKey="id">
      <ng-template #header>
        <tr>
          <th>Name</th>
          <th>Price</th>
          <th class="w-0"></th>
        </tr>
      </ng-template>
      <ng-template #body let-product>
        <tr>
          <td>{{ product.name }}</td>
          <td>{{ product.price }}</td>
          <td class="flex gap-2">
            <p-button label="Edit" size="small" severity="secondary" [routerLink]="[product.id, 'edit']" />
            <p-button label="Archive" size="small" severity="danger" [outlined]="true" (onClick)="confirmArchive(product)" />
          </td>
        </tr>
      </ng-template>
      <ng-template #emptymessage>
        <tr>
          <td colspan="3" class="text-center text-surface-500">No products yet.</td>
        </tr>
      </ng-template>
    </p-table>
    <p-button label="Next page" [text]="true" (onClick)="showNextPage()" />
  }
</section>
<p-confirmdialog />
```

## 8. Presentational component

`src/app/features/products/product-card/product-card.ts`: inputs in, outputs out, nothing feature-specific injected.

```ts
import { Component, computed, input, output } from '@angular/core';
import { Button } from 'primeng/button';
import { Tag } from 'primeng/tag';
import { ProductDto } from '../product';

@Component({
  selector: 'app-product-card',
  imports: [Button, Tag],
  template: `
    <article class="flex items-center justify-between rounded-lg border border-surface-200 p-4 dark:border-surface-700">
      <div class="flex flex-col gap-1">
        <h3 class="font-medium">{{ product().name }}</h3>
        <span class="text-surface-500">{{ product().price }}</span>
      </div>
      @if (isExpensive()) {
        <p-tag value="Premium" severity="warn" />
      }
      <p-button label="Archive" size="small" [text]="true" (onClick)="archive.emit(product().id)" />
    </article>
  `,
})
export class ProductCard {
  readonly product = input.required<ProductDto>();
  readonly archive = output<string>();

  protected readonly isExpensive = computed(() => this.product().price > 10_000);
}
```

## 9. Edit component (Signal Forms + PrimeNG inputs + write)

`src/app/features/products/product-edit/product-edit.ts`

```ts
import { Component, inject, input, signal } from '@angular/core';
import { Router } from '@angular/router';
import { form, FormField, maxLength, min, required } from '@angular/forms/signals';
import { firstValueFrom } from 'rxjs';
import { MessageService } from 'primeng/api';
import { Button } from 'primeng/button';
import { InputNumber } from 'primeng/inputnumber';
import { InputText } from 'primeng/inputtext';
import { ApiError } from '../../../core/http/problem-details';
import { ProductsApi } from '../products-api';
import { EditProductRequest } from '../product';

@Component({
  selector: 'app-product-edit',
  imports: [FormField, Button, InputNumber, InputText],
  templateUrl: './product-edit.html',
})
export class ProductEdit {
  private readonly api = inject(ProductsApi);
  private readonly router = inject(Router);
  private readonly messages = inject(MessageService);

  readonly id = input.required<string>(); // route param via withComponentInputBinding()

  protected readonly model = signal<EditProductRequest>({ name: '', price: 0 });
  protected readonly productForm = form(this.model, (p) => {
    required(p.name);
    maxLength(p.name, 200); // mirrors [MaxLength(200)] on the C# request
    min(p.price, 0);
  });
  protected readonly saving = signal(false);

  protected async saveProduct(): Promise<void> {
    if (this.productForm().invalid()) return;
    this.saving.set(true);
    try {
      await firstValueFrom(this.api.edit(this.id(), this.model()));
      this.messages.add({ severity: 'success', summary: 'Product saved' });
      await this.router.navigate(['/products']);
    } catch (err) {
      this.messages.add({
        severity: 'error',
        summary: 'Could not save the product',
        detail: err instanceof ApiError ? err.message : undefined,
      });
    } finally {
      this.saving.set(false);
    }
  }
}
```

`product-edit.html`: `[formField]` works on `pInputText` and on PrimeNG form components such as `p-inputnumber`.

```html
<form class="flex max-w-md flex-col gap-4 p-4" (submit)="$event.preventDefault(); saveProduct()">
  <div class="flex flex-col gap-1">
    <label for="name">Name</label>
    <input pInputText id="name" [formField]="productForm.name" />
    @if (productForm.name().invalid() && productForm.name().touched()) {
      <small class="text-red-600">Name is required (max 200 characters).</small>
    }
  </div>

  <div class="flex flex-col gap-1">
    <label for="price">Price</label>
    <p-inputnumber inputId="price" [formField]="productForm.price" mode="currency" currency="DKK" locale="da-DK" />
  </div>

  <p-button type="submit" label="Save" [loading]="saving()" />
</form>
```

## 10. Job polling (202 + `jobId`)

`src/app/core/http/poll-job.ts`

```ts
import { HttpClient } from '@angular/common/http';
import { Observable, switchMap, takeWhile, timer } from 'rxjs';
import { apiUrl } from './api-url';

export interface JobStatus {
  jobId: string;
  state: 'Enqueued' | 'Processing' | 'Succeeded' | 'Failed';
  message?: string;
}

export function pollJob(http: HttpClient, jobId: string, intervalMs = 2000): Observable<JobStatus> {
  return timer(0, intervalMs).pipe(
    switchMap(() => http.get<JobStatus>(apiUrl(`jobs/${jobId}`))),
    takeWhile((s) => s.state !== 'Succeeded' && s.state !== 'Failed', true), // emit the final state too
  );
}
```

In a component: `uploadImages()` → `jobId` → `pollJob(...)` → set a `jobState` signal from each emission (show it with `p-progressbar`/`p-tag`); `takeUntilDestroyed()` to stop when the component goes away.

## 11. Spec (Vitest + `HttpTestingController`)

`src/app/features/products/products-api.spec.ts`

```ts
import { afterEach, beforeEach, describe, expect, it } from 'vitest';
import { TestBed } from '@angular/core/testing';
import { provideHttpClient } from '@angular/common/http';
import { HttpTestingController, provideHttpClientTesting } from '@angular/common/http/testing';
import { firstValueFrom } from 'rxjs';
import { ProductsApi } from './products-api';

describe('ProductsApi', () => {
  let api: ProductsApi;
  let http: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({ providers: [provideHttpClient(), provideHttpClientTesting()] });
    api = TestBed.inject(ProductsApi);
    http = TestBed.inject(HttpTestingController);
  });

  afterEach(() => http.verify());

  it('edit sends PUT with the body to the versioned products endpoint', async () => {
    // Arrange
    const body = { name: 'Chair', price: 100 };

    // Act
    const result = firstValueFrom(api.edit('42', body));
    const req = http.expectOne((r) => r.url.endsWith('/api/v1/products/42'));
    req.flush({ id: '42', ...body });

    // Assert
    expect(req.request.method).toBe('PUT');
    expect(req.request.body).toEqual(body);
    expect(await result).toEqual({ id: '42', ...body });
  });
});
```

Component specs that render PrimeNG components need `providePrimeNG({ theme: 'none' })` and `MessageService`/`ConfirmationService` in the TestBed providers.
