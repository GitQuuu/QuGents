---
name: security-best-practices
description: Security baseline for ASP.NET Core Web APIs and Angular frontends - OWASP API Security Top 10 (2023) and ASVS level 2 mapped to this stack, configuration hardening (HTTPS/HSTS, headers, CORS, cookies, JWT, Data Protection), Angular (XSS, CSP, token handling, open redirects), dependency and secret checks, severities and the findings format. Use when threat-modelling a backlog item, reviewing a change for security, hardening configuration, or auditing a solution.
---

# Security best practices: .NET Web API + Angular

Line-level code review checks (injection, null handling, async) are in the `secure-code-review` skill; load it too when reviewing a diff. This skill is the security baseline: what must be true for the system to be safe, and how to check it.

## Severity

| Severity | Meaning | Blocks merge |
|---|---|---|
| **Critical** | Exploitable now without special access: auth bypass, another user's data (IDOR), secret in code or logs, injection, RCE | Yes |
| **High** | Exploitable with some effort or a common precondition: missing role check on an admin endpoint, mass assignment of a privileged field, SSRF to internal hosts, known-vulnerable package (High/Critical CVE) in a used path | Yes |
| **Medium** | Defence in depth missing or limited impact: missing security header, verbose error, weak rate limit, no max page size | No, list it in the PR |
| **Low** | Hardening or hygiene | No |

Every finding needs a concrete exploit scenario ("an authenticated user calls `PUT api/v1/projects/{otherUsersId}` and …"). No scenario, no finding; put it under Questions.

## OWASP API Security Top 10 (2023) on this stack

| Risk | What must be true here |
|---|---|
| **API1 Broken object level authorization** | Every endpoint that takes an id checks that the caller may access *that* object, in the Service (ownership or role), and returns 404/403 via `ServiceResult`. The owner comes from `User.GetUserId()`, never from the request body or query |
| **API2 Broken authentication** | JWT validation: issuer, audience, lifetime and signing key (≥ 256-bit, from configuration) all validated, small `ClockSkew`, expected algorithm only. Refresh tokens: random, hashed at rest, rotated, reuse detection, HttpOnly + Secure + SameSite=Strict cookie with a narrow `Path`. Identity lockout on. Same message and timing for unknown user and wrong password. Auth endpoints rate limited |
| **API3 Broken object property level authorization** | Response types expose only what the caller may see (never entities, password hashes, stamps, tokens, internal ids of others). Request types contain only fields the caller may set: no `Id`, `OwnerId`, `Role`, `IsActive`, `IsAdmin` from the body unless that endpoint is meant to change it. `dto.Adapt(entity)` maps every same-named member, so the request/DTO type must not have members that must not be changed |
| **API4 Unrestricted resource consumption** | List endpoints are paged with a maximum page size. Request body and upload size limits. Rate limits on expensive or abusable endpoints. `HttpClient` calls have timeouts. Background jobs have time and cost limits |
| **API5 Broken function level authorization** | Default-deny fallback policy (`RequireAuthenticatedUser`); admin actions have `[Authorize(Roles = …)]` or a policy; `[AllowAnonymous]` only where intended and listed in the PR |
| **API6 Unrestricted access to sensitive business flows** | Flows that can be abused in bulk (registration, invitations, connection tests, job starts) have rate limits or approval |
| **API7 Server-side request forgery** | A URL or host that comes from a user (an organization URL, a repository) is parsed and checked against an allow-list (e.g. `dev.azure.com`, `github.com`, `api.github.com`), `https` only, no redirects followed to other hosts, never to localhost, private or link-local addresses |
| **API8 Security misconfiguration** | See **Configuration hardening** |
| **API9 Improper inventory management** | Every endpoint is versioned (`Controllers/V<n>`); no debug, test or sample endpoints outside Development; Swagger and the Hangfire dashboard are not public in production |
| **API10 Unsafe consumption of APIs** | Responses from external APIs are validated before use, errors from them are mapped (not passed through raw), tokens for them are never logged |

## Configuration hardening (API)

- `UseHttpsRedirection`; `UseHsts` outside Development.
- Headers on every response: `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY` (or CSP `frame-ancestors 'none'`), `Referrer-Policy: no-referrer`. Add them in one `Use<…>Extension()`.
- CORS: explicit origins from configuration; never `AllowAnyOrigin` together with credentials; methods and headers only as needed.
- Errors: `ProblemDetails` without exception messages or stack traces outside Development; no `UseDeveloperExceptionPage` in production.
- Secrets: only the `// DEV ONLY` values in `appsettings.Development.json`; production values come from the host or a vault. Startup fails when a required secret is missing.
- Tokens and credentials the system stores (PATs, API keys) are encrypted with ASP.NET Core Data Protection, keys persisted outside the repo, never logged and never returned (show at most the last 4 characters).
- Logging: no passwords, tokens, cookies, `Authorization` headers or full request bodies of auth endpoints.

## Angular

- No `bypassSecurityTrust*` and no `[innerHTML]` with data from users or APIs, unless sanitized and justified in the PR.
- The access token lives in memory only (never `localStorage`/`sessionStorage`); the refresh token is only in the HttpOnly cookie. `withCredentials` only on requests to the API origin.
- Route guards are UX; the API enforces every rule. Never trust the client.
- Redirects after login (`returnUrl`) accept relative in-app paths only (open redirect).
- No secrets or tokens in `environment*.ts`; they ship to the browser.
- A Content-Security-Policy is set by the host (or a `<meta>` tag) for production: no `unsafe-eval`; `connect-src` limited to the API origin.

## Dependencies and secrets

Run when packages or lock files change, and in an audit:

```bash
dotnet list package --vulnerable --include-transitive   # backend: High/Critical = finding
npm audit --omit=dev --audit-level=high                  # frontend
```

Secrets in a diff: look for keys, tokens, connection strings and private keys (`grep -nEi "(api[_-]?key|secret|password|token|connectionstring|BEGIN .*PRIVATE KEY)"`), and use `gitleaks detect` when it is installed. A real secret that reached git is **Critical** even after removal: it must be rotated.

## ASVS level 2 (selection used here)

Authentication (V2), session management (V3), access control (V4), validation and encoding (V5), error handling and logging (V7), data protection (V8), communications (V9), API (V13), configuration (V14). In plan mode, turn the relevant ones into testable acceptance criteria.

## Findings format

```markdown
## Security review: <task or scope>
Verdict: PASS | FAIL (any Critical or High)

### Critical / High
1. **[API1] Other users' projects can be edited** — `src/…/ProjectService.EditProjectAsync.cs:22`
   Scenario: an authenticated non-owner sends PUT api/v1/projects/{id} and the project is changed.
   Fix: load with `GetByIdForOwnerAsync(id, userId)` and return 404 when not found.

### Medium / Low
- [API8] No `X-Content-Type-Options` header — add in `SecurityHeadersExtensions`.

### Checked
OWASP API1–API10 for the touched endpoints, configuration, dependencies (`dotnet list package --vulnerable`: none), secrets in diff: none.
```
