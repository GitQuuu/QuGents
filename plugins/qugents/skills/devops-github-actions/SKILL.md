---
name: devops-github-actions
description: Three-branch strategy (development → staging → prod) with GitHub Actions - CI on pull requests, deploy per environment to Azure (App Service + Static Web Apps) or MonsterASP (Web Deploy), release and back-merge pull requests, branch rulesets, environment secrets and the bot identity for agent PRs. Use for CI/CD, pipelines, deployments, branch protection, releases, hotfixes, environments or hosting questions.
---

# DevOps: three branches + GitHub Actions

If the repo already has workflows or a different branch model, follow it and say where it differs.

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
- **Agents open PRs to `development` only.** Promotions to `staging` and `prod` are always the PO's decision; the devops agent may prepare the release PR but never merges it.
- `development` is the GitHub default branch, so `Fixes AB#<id>` in a merged PR closes the Azure Boards work item (Azure Boards GitHub app).
- Only `prod` requires a pull request (with a green `ci`). `development` and `staging` accept direct pushes, but none of the three can be force-pushed or deleted (rulesets below). Agents still always deliver through PRs to `development`.
- A hotfix merged into `prod` always gets a back-merge PR `prod → development` so the fix is not lost at the next release.

## Workflows (templates in `references/workflows.md`)

| File | Trigger | Does |
|---|---|---|
| `.github/workflows/ci.yml` | PR to the three branches; reusable (`workflow_call`) | restore, build, test (.NET), `npm ci`, `ng build`, `ng test`. Job name `ci` is the required status check |
| `.github/workflows/deploy.yml` | push to the three branches | runs `ci`, then deploys to the environment named like the branch, to the target in repo variable `DEPLOY_TARGET` |
| `.github/workflows/release-pr.yml` | manual (`workflow_dispatch`) | opens the release PR `development → staging` or `staging → prod` with a changelog of merged PRs and `AB#` ids |
| `.github/workflows/back-merge.yml` | push to `prod` | opens/updates a PR `prod → development` when `prod` has commits `development` lacks |

## Deploy targets (pluggable)

Repo variable `DEPLOY_TARGET` = `azure` or `monsterasp`. Everything environment-specific lives in **GitHub Environments** (`development`, `staging`, `prod`) as variables and secrets, never in the repo.

| Target | API | Angular | Environment secrets / variables |
|---|---|---|---|
| `azure` | App Service via `azure/login` (OIDC, no passwords) + `azure/webapps-deploy` | Azure Static Web Apps | secrets `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`, `AZURE_STATIC_WEB_APPS_API_TOKEN`; vars `AZURE_WEBAPP_NAME` |
| `monsterasp` | Web Deploy with `rasmusbuchholdt/simply-web-deploy` (Windows runner) | a second MonsterASP site with the static build + `web.config` SPA rewrite | secrets `MONSTERASP_API_SERVER`, `_SITE`, `_USERNAME`, `_PASSWORD` and the same four `MONSTERASP_WEB_*` for the Angular site (each from that site's *Visual Studio access* card) |

Shared environment variables: `NG_CONFIGURATION` (`dev`, `staging`, `production`). Repo variables: `API_PROJECT` (e.g. `Shop.Api`), `WEB_DIST` (e.g. `web/dist/web/browser`). App settings for each environment (`Jwt__SigningKey`, `ConnectionStrings__Default`, `Cors__Origins__0`) are set in the host (App Service configuration / MonsterASP control panel), not in workflows. This is where the `// DEV ONLY` values from `appsettings.Development.json` are replaced.

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

Run it once per repo with an admin token. Raise prod approvals to 1 only when someone other than the PR author can approve.

## Bot identity for agent PRs (important)

GitHub does not let you approve your own pull request, and agent PRs opened with **your** token look like your own work (and PRs created with `GITHUB_TOKEN` do not trigger `ci`). Give the agents their own identity:
- a **GitHub App** (best: per-repo installation, short-lived tokens), or
- a **machine user** with a fine-grained PAT limited to the repo (Contents RW, Pull requests RW, Metadata R).

Then you review and approve as yourself.

## What the devops agent does

- New repo: create `development` and `staging`/`prod` branches from the initial commit, set `development` as default, add the four workflows, Angular configurations, `web.config` (MonsterASP), and print the list of secrets/variables the PO must create per environment.
- Backlog tasks about pipelines, deployment, Docker, health checks or environments.
- On request: prepare a release PR (or trigger `release-pr.yml`) and summarize what is going out.
- Never: merge or approve PRs, push to the three branches, store secrets in the repo, weaken a ruleset or required check.

Validate workflow changes with `actionlint` when available.
