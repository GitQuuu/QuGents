---
name: devops-github-actions
description: Three-branch strategy (development → staging → prod) with GitHub Actions for a backend repo (.NET API) and a separate frontend repo (Angular) - CI on pull requests, deploy per environment to Azure (App Service + Static Web Apps) or MonsterASP (Web Deploy), release and back-merge pull requests, branch rulesets, environment secrets and the bot identity for agent PRs. Use for CI/CD, pipelines, deployments, branch protection, releases, hotfixes, environments or hosting questions.
---

# DevOps: three branches + GitHub Actions

If the repo already has workflows or a different branch model, follow it and say where it differs.

## Two repositories

Backend and frontend live in **separate repositories**: `<App>` (the .NET solution, e.g. `GitQuuu/Shop`) and `<App>-web` (the Angular app at the repo root, e.g. `GitQuuu/Shop-web`). Everything below applies to **each** repo: same branches, rulesets and environment names, but each repo has its own `ci.yml` and `deploy.yml` (backend builds and deploys the API, frontend builds and deploys the SPA). `release-pr.yml`, `back-merge.yml` and the setup script are identical in both.

- Release the **backend first**, then the frontend: the frontend in an environment calls the API in the same environment.
- An API change the frontend depends on must be merged (and deployed to that environment) before the frontend change that uses it. Breaking changes go behind a new API version.
- Connect **both** repos to Azure Boards (Azure Boards GitHub app), so `AB#<id>` and `Fixes AB#<id>` work in either.

## N-tier: the physical tiers

N-tier is about **where things run**; N-layer (Api → Services → DataAccess) is about how the API's code is organized and belongs to the developers (`dotnet-n-layer`). The devops agent owns the tiers.

| Tier | Runs | Azure | MonsterASP |
|---|---|---|---|
| Presentation | Angular static build (frontend repo) | Static Web Apps | static site + `web.config` |
| Application | ASP.NET Core API, incl. the in-process Hangfire server (backend repo) | App Service | API site |
| Data | Database. Starts as SQLite on the API host; later a server database as its own tier | Azure SQL / PostgreSQL Flexible Server | MonsterASP MSSQL |

- **One set of tiers per environment.** `development`, `staging` and `prod` each have their own frontend, API and database. Never share a database between environments.
- **Only neighbours talk.** Browser → API over HTTPS (the API's CORS origin is the frontend URL of the same environment); API → database over an allow-listed or private connection. The browser never reaches the database.
- **Configuration per tier.** Frontend: API URL through the build configuration (`environment.<env>.ts`, `NG_CONFIGURATION`). API: connection strings and secrets in the host's configuration. Database: least privilege for the app user (no schema changes in `prod`); migrations run as a separate deploy step.
- **Deploy order:** data (migrations) → API → frontend, so a new frontend never calls an API that is not there yet.
- **SQLite caveat:** with SQLite the data tier lives on the API host, so the API cannot scale out to more than one instance and the data is lost if the host's disk is replaced. Move to a server database before scaling or before real production data.
- **Health per tier:** `/health` on the API (extend with a database check when the data tier is separate); the frontend host serves `index.html` for every route.

## Branch model

```
feature/*, agent/*  ──PR──►  development  ──PR──►  staging  ──PR──►  prod
                              (dev env)            (test env)         (production)
hotfix/*  ──PR──►  prod  ──back-merge PR──►  development
```

| Branch | Environment | Who merges | Receives PRs from |
|---|---|---|---|
| `development` (GitHub default branch) | `development` | PO / developer | `feature/*`, `agent/*` |
| `staging` | `staging` | PO | `development` only (release PR) |
| `prod` | `prod` | PO | `staging` (release PR) or `hotfix/*` |

Rules:
- **Agents open PRs to `development` only.** Promotions to `staging` and `prod` are always the PO's decision; the devops agent may prepare the release PR but never merges it. The only agent merge is `ado-deliver` merging a PBI-approved Task into `development` (merge commit, no squash) after green `ci` and a verifier PASS.
- `development` is the GitHub default branch, so `Fixes AB#<id>` in a merged PR closes the Azure Boards work item (Azure Boards GitHub app).
- Only `prod` requires a pull request (with a green `ci`). `development` and `staging` accept direct pushes, but none of the three can be force-pushed or deleted (rulesets below). Agents still always deliver through PRs to `development`.
- A hotfix merged into `prod` always gets a back-merge PR `prod → development` so the fix is not lost at the next release.

## Workflows (templates in `references/workflows.md`)

| File | Trigger | Does |
|---|---|---|
| `.github/workflows/ci.yml` | PR to the three branches; reusable (`workflow_call`) | backend: `dotnet restore`, `build`, `test`. Frontend: `npm ci`, `ng build`, `ng test`. Job name `ci` is the required status check in both |
| `.github/workflows/deploy.yml` | push to the three branches | runs `ci`, then deploys (backend: the API; frontend: the static build) to the environment named like the branch, to the target in repo variable `DEPLOY_TARGET` |
| `.github/workflows/release-pr.yml` | manual (`workflow_dispatch`) | opens the release PR `development → staging` or `staging → prod` with a changelog of merged PRs and `AB#` ids |
| `.github/workflows/back-merge.yml` | push to `prod` | opens/updates a PR `prod → development` when `prod` has commits `development` lacks |

## Deploy targets (pluggable)

Repo variable `DEPLOY_TARGET` = `azure` or `monsterasp`. Everything environment-specific lives in **GitHub Environments** (`development`, `staging`, `prod`) as variables and secrets, never in the repo.

| Target | Backend repo (API) | Frontend repo (Angular) |
|---|---|---|
| `azure` | App Service via `azure/login` (OIDC, no passwords) + `azure/webapps-deploy`. Secrets `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`; variable `AZURE_WEBAPP_NAME` | Azure Static Web Apps. Secret `AZURE_STATIC_WEB_APPS_API_TOKEN` |
| `monsterasp` | Web Deploy with `rasmusbuchholdt/simply-web-deploy` (Windows runner). Secrets `MONSTERASP_API_SERVER`, `_SITE`, `_USERNAME`, `_PASSWORD` | A second MonsterASP site with the static build + `web.config` SPA rewrite. Secrets `MONSTERASP_WEB_SERVER`, `_SITE`, `_USERNAME`, `_PASSWORD` (each from that site's *Visual Studio access* card) |

Backend repo variable: `API_PROJECT` (e.g. `Shop.Api`). Frontend repo variable: `WEB_DIST` (e.g. `dist/shop-web/browser`), and per environment `NG_CONFIGURATION` (`dev`, `staging`, `production`). Backend app settings for each environment (`Jwt__SigningKey`, `ConnectionStrings__Default`, `Cors__Origins__0` = the frontend's URL in that environment) are set in the host (App Service configuration / MonsterASP control panel), not in workflows. This is where the `// DEV ONLY` values from `appsettings.Development.json` are replaced.

Angular gets one build configuration per environment (`dev`, `staging`, `production`) with `fileReplacements` for `environment.ts`, so each build points at its own API URL.

## Database migrations on deploy

- `development`/`staging` on SQLite: the API may apply migrations at startup behind a setting (`Database:MigrateOnStartup=true` in those environments only).
- Server databases (`prod`, or once the project leaves SQLite): generate an idempotent script in CI (`dotnet ef migrations script --idempotent`) and apply it as a deploy step or by hand. Never `MigrateOnStartup` in `prod`.

## Branch rulesets

`references/workflows.md` has a `gh api` script (idempotent, safe to re-run) that creates two rulesets:

| Ruleset | Branches | Rules |
|---|---|---|
| `branch-safety` | `development`, `staging` | no deletion, no force push |
| `prod-protection` | `prod` | PR required, required status check `ci`, approvals = script argument (default 0), no deletion, no force push |

Run it once in **each** repo with an admin token (on Windows from cmd: `scripts\setup-github-repo.cmd <owner/repo>`). Raise prod approvals to 1 only when someone other than the PR author can approve.

## Bot identity for agent PRs (important)

GitHub does not let you approve your own pull request, and agent PRs opened with **your** token look like your own work (and PRs created with `GITHUB_TOKEN` do not trigger `ci`). Give the agents their own identity:
- a **GitHub App** (best: per-repo installation, short-lived tokens), or
- a **machine user** with a fine-grained PAT limited to the backend and frontend repos (Contents RW, Pull requests RW, Metadata R).

Then you review and approve as yourself.

## What the devops agent does

- New project: in **both** repos create `development`, `staging` and `prod` from the initial commit, set `development` as default and add the four workflows (backend or frontend variant) plus the setup script. In the frontend repo also the Angular configurations and the SPA fallback files. Print the list of secrets/variables the PO must create per repo and environment.
- Backlog tasks about pipelines, deployment, Docker, health checks or environments.
- On request: prepare a release PR (or trigger `release-pr.yml`) and summarize what is going out.
- Never: merge or approve PRs, push to the three branches, store secrets in the repo, weaken a ruleset or required check.

Validate workflow changes with `actionlint` when available.
