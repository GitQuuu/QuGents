# Workflow and setup templates

Validated with `actionlint` (incl. shellcheck). Replace `Shop.Api`, `web` and paths with the project's names. Action versions are current majors at the time of writing; pin to a commit SHA if the repo's policy requires it.

## Contents

1. `ci.yml`
2. `deploy.yml` (Azure or MonsterASP)
3. `release-pr.yml`
4. `back-merge.yml`
5. Repo setup script: branches, default branch, rulesets, environments
6. Angular build configurations per environment
7. SPA fallback files (`web.config` for MonsterASP, `staticwebapp.config.json` for Azure)
8. GitHub Environments checklist

---

## 1. `.github/workflows/ci.yml`

```yaml
name: ci

on:
  pull_request:
    branches: [development, staging, prod]
  workflow_call:

permissions:
  contents: read

jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-dotnet@v4
        with:
          global-json-file: global.json

      - run: dotnet restore
      - run: dotnet build --no-restore -c Release
      - run: dotnet test --no-build -c Release

      - uses: actions/setup-node@v4
        with:
          node-version: '24'
          cache: npm
          cache-dependency-path: web/package-lock.json

      - run: npm ci
        working-directory: web
      - run: npx ng build --configuration production
        working-directory: web
      - run: npx ng test --watch=false
        working-directory: web
```

## 2. `.github/workflows/deploy.yml`

```yaml
name: deploy

on:
  push:
    branches: [development, staging, prod]

concurrency:
  group: deploy-${{ github.ref_name }}
  cancel-in-progress: false

permissions:
  contents: read

jobs:
  ci:
    uses: ./.github/workflows/ci.yml

  deploy-azure:
    needs: ci
    if: vars.DEPLOY_TARGET == 'azure'
    runs-on: ubuntu-latest
    environment: ${{ github.ref_name }}   # GitHub Environment named like the branch
    permissions:
      contents: read
      id-token: write                     # OIDC login to Azure, no stored passwords
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-dotnet@v4
        with:
          global-json-file: global.json
      - run: dotnet publish "src/${{ vars.API_PROJECT }}" -c Release -o publish/api

      - uses: actions/setup-node@v4
        with:
          node-version: '24'
          cache: npm
          cache-dependency-path: web/package-lock.json
      - run: npm ci
        working-directory: web
      - run: npx ng build --configuration "${{ vars.NG_CONFIGURATION }}"
        working-directory: web

      - uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy API to App Service
        uses: azure/webapps-deploy@v3
        with:
          app-name: ${{ vars.AZURE_WEBAPP_NAME }}
          package: publish/api

      - name: Deploy Angular to Static Web Apps
        uses: Azure/static-web-apps-deploy@v1
        with:
          azure_static_web_apps_api_token: ${{ secrets.AZURE_STATIC_WEB_APPS_API_TOKEN }}
          action: upload
          app_location: ${{ vars.WEB_DIST }}
          skip_app_build: true
          skip_api_build: true

  deploy-monsterasp:
    needs: ci
    if: vars.DEPLOY_TARGET == 'monsterasp'
    runs-on: windows-latest                # Web Deploy (msdeploy) needs Windows
    environment: ${{ github.ref_name }}
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-dotnet@v4
        with:
          global-json-file: global.json
      - run: dotnet publish "src/${{ vars.API_PROJECT }}" -c Release -r win-x86 --self-contained false -o publish/api

      - uses: actions/setup-node@v4
        with:
          node-version: '24'
          cache: npm
          cache-dependency-path: web/package-lock.json
      - run: npm ci
        working-directory: web
      - run: npx ng build --configuration "${{ vars.NG_CONFIGURATION }}"
        working-directory: web
      - name: Collect Angular build
        shell: pwsh
        run: |
          New-Item -ItemType Directory -Force publish/web | Out-Null
          Copy-Item -Recurse -Force "${{ vars.WEB_DIST }}/*" publish/web

      - name: Deploy API (Web Deploy)
        uses: rasmusbuchholdt/simply-web-deploy@2.2.0
        with:
          website-name: ${{ secrets.MONSTERASP_API_SITE }}
          server-computer-name: ${{ secrets.MONSTERASP_API_SERVER }}
          server-username: ${{ secrets.MONSTERASP_API_USERNAME }}
          server-password: ${{ secrets.MONSTERASP_API_PASSWORD }}
          source-path: '\publish\api\'

      - name: Deploy Angular (Web Deploy)
        uses: rasmusbuchholdt/simply-web-deploy@2.2.0
        with:
          website-name: ${{ secrets.MONSTERASP_WEB_SITE }}
          server-computer-name: ${{ secrets.MONSTERASP_WEB_SERVER }}
          server-username: ${{ secrets.MONSTERASP_WEB_USERNAME }}
          server-password: ${{ secrets.MONSTERASP_WEB_PASSWORD }}
          source-path: '\publish\web\'
```

## 3. `.github/workflows/release-pr.yml`

Run it from the Actions tab ("Run workflow" → target `staging` or `prod`). It needs the `BOT_TOKEN` secret (the bot identity's token): pull requests created with the built-in `GITHUB_TOKEN` do not trigger `ci`, so they could never pass the required check.

```yaml
name: release-pr

on:
  workflow_dispatch:
    inputs:
      target:
        description: Promote to
        type: choice
        options: [staging, prod]
        required: true

permissions:
  contents: read

jobs:
  release-pr:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Open or update the release PR
        env:
          # Bot identity token: PRs made with GITHUB_TOKEN do not trigger the ci workflow.
          GH_TOKEN: ${{ secrets.BOT_TOKEN }}
          TARGET: ${{ inputs.target }}
        run: |
          if [ "$TARGET" = "staging" ]; then SOURCE=development; else SOURCE=staging; fi
          git fetch origin "$SOURCE" "$TARGET"
          LOG=$(git log --no-merges --format='- %s' "origin/$TARGET..origin/$SOURCE")
          if [ -z "$LOG" ]; then
            echo "Nothing to release from $SOURCE to $TARGET."
            exit 0
          fi
          IDS=$(echo "$LOG" | grep -o 'AB#[0-9]\+' | sort -u | tr '\n' ' ' || true)
          BODY=$(printf '## Release %s -> %s\n\n%s\n\nWork items: %s\n' "$SOURCE" "$TARGET" "$LOG" "$IDS")
          EXISTING=$(gh pr list --base "$TARGET" --head "$SOURCE" --state open --json number --jq '.[0].number')
          if [ -n "$EXISTING" ]; then
            gh pr edit "$EXISTING" --body "$BODY"
          else
            gh pr create --base "$TARGET" --head "$SOURCE" \
              --title "Release $SOURCE -> $TARGET ($(date -u +%Y-%m-%d))" --body "$BODY"
          fi
```

## 4. `.github/workflows/back-merge.yml`

```yaml
name: back-merge

on:
  push:
    branches: [prod]

permissions:
  contents: read

jobs:
  back-merge:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Open a back-merge PR when prod has commits development lacks
        env:
          GH_TOKEN: ${{ secrets.BOT_TOKEN }}
        run: |
          git fetch origin development prod
          if [ -z "$(git log --no-merges --format=%H origin/development..origin/prod)" ]; then
            echo "development already contains everything from prod."
            exit 0
          fi
          EXISTING=$(gh pr list --base development --head prod --state open --json number --jq '.[0].number')
          if [ -z "$EXISTING" ]; then
            gh pr create --base development --head prod \
              --title "Back-merge prod -> development" \
              --body "Brings hotfixes from prod back into development."
          fi
```

## 5. Repo setup (once, with an admin token; safe to re-run)

`scripts/setup-github-repo.sh`:

```bash
#!/usr/bin/env bash
# Usage: scripts/setup-github-repo.sh <owner/repo> [prod-approvals]
# Needs: gh CLI logged in as a repo admin (gh auth login) and a local clone with at least one commit.
# Windows: run from Git Bash. Safe to run again: existing branches, rulesets and environments are kept or updated.
#
# Result:
#   development  default branch, no PR required, no deletion, no force push
#   staging      no PR required, no deletion, no force push
#   prod         PR + green `ci` required (+ [prod-approvals] approvals, default 0), no deletion, no force push
set -euo pipefail
REPO="${1:?Usage: $0 <owner/repo> [prod-approvals]}"
APPROVALS="${2:-0}"

gh auth status >/dev/null 2>&1 || { echo "Run 'gh auth login' first." >&2; exit 1; }

# Three long-lived branches from the current commit (existing ones are left alone).
for b in development staging prod; do
  if git ls-remote --exit-code --heads origin "$b" >/dev/null; then
    echo "Branch $b already exists"
  else
    git push origin "HEAD:refs/heads/$b"
  fi
done

# development is the default branch (agent PRs target it; "Fixes AB#123" closes work items on merge).
gh api --method PATCH "repos/$REPO" -f default_branch=development >/dev/null
echo "Default branch: development"

ruleset_id() {
  gh api "repos/$REPO/rulesets" --jq ".[] | select(.name == \"$1\") | .id"
}

# Create or update a ruleset by name.
upsert_ruleset() {
  local name="$1" body="$2" id
  id=$(ruleset_id "$name")
  if [ -n "$id" ]; then
    gh api --method PUT "repos/$REPO/rulesets/$id" --input - <<<"$body" >/dev/null
  else
    gh api --method POST "repos/$REPO/rulesets" --input - <<<"$body" >/dev/null
  fi
  echo "Ruleset: $name"
}

# Earlier versions of this script required PRs on all three branches.
OLD=$(ruleset_id three-branch-protection)
if [ -n "$OLD" ]; then
  gh api --method DELETE "repos/$REPO/rulesets/$OLD"
  echo "Removed old ruleset: three-branch-protection"
fi

# development + staging: direct pushes allowed, but never deleted or force-pushed.
upsert_ruleset branch-safety '{
  "name": "branch-safety",
  "target": "branch",
  "enforcement": "active",
  "conditions": { "ref_name": { "include": ["refs/heads/development", "refs/heads/staging"], "exclude": [] } },
  "rules": [ { "type": "deletion" }, { "type": "non_fast_forward" } ]
}'

# prod: only through a pull request with a green ci check.
upsert_ruleset prod-protection "{
  \"name\": \"prod-protection\",
  \"target\": \"branch\",
  \"enforcement\": \"active\",
  \"conditions\": { \"ref_name\": { \"include\": [\"refs/heads/prod\"], \"exclude\": [] } },
  \"rules\": [
    { \"type\": \"deletion\" },
    { \"type\": \"non_fast_forward\" },
    { \"type\": \"pull_request\", \"parameters\": {
        \"required_approving_review_count\": $APPROVALS,
        \"dismiss_stale_reviews_on_push\": true,
        \"require_code_owner_review\": false,
        \"require_last_push_approval\": false,
        \"required_review_thread_resolution\": false } },
    { \"type\": \"required_status_checks\", \"parameters\": {
        \"strict_required_status_checks_policy\": false,
        \"required_status_checks\": [{ \"context\": \"ci\" }] } }
  ]
}"

# Environments named like the branches (add secrets/variables in the UI afterwards).
for env in development staging prod; do
  gh api --method PUT "repos/$REPO/environments/$env" >/dev/null
done
echo "Environments: development, staging, prod"

echo "Done. Now add the variables and secrets per environment (see the devops-github-actions skill)."
```

On Windows the same script runs from cmd through a small wrapper, `scripts/setup-github-repo.cmd` (CRLF):

```bat
@echo off
rem Runs setup-github-repo.sh with Git for Windows' bash, so it also works from cmd.
rem Usage (from the repo root): scripts\setup-github-repo.cmd ^<owner/repo^> [prod-approvals]
setlocal
set "GITBASH=%ProgramFiles%\Git\bin\bash.exe"
if exist "%GITBASH%" goto run
for /f "delims=" %%G in ('where git 2^>nul') do if exist "%%~dpG..\bin\bash.exe" set "GITBASH=%%~dpG..\bin\bash.exe"
if not exist "%GITBASH%" (
  echo Could not find bash.exe from Git for Windows. 1>&2
  exit /b 1
)
:run
"%GITBASH%" "%~dp0setup-github-repo.sh" %*
exit /b %ERRORLEVEL%
```

Add `.gitattributes` so the bash script keeps LF line endings on Windows (CRLF breaks it with `$'\r': command not found`):

```gitattributes
* text=auto
*.sh text eol=lf
*.cmd text eol=crlf
```

## 6. Angular configurations per environment

`web/src/environments/`: `environment.ts` (local dev), `environment.dev.ts`, `environment.staging.ts`, `environment.prod.ts`, each with its own `apiUrl` (e.g. `https://shop-api-staging.azurewebsites.net`).

In `angular.json` → `projects.web.architect.build.configurations`, add next to the generated `production`:

```json
"dev":     { "fileReplacements": [{ "replace": "src/environments/environment.ts", "with": "src/environments/environment.dev.ts" }] },
"staging": { "fileReplacements": [{ "replace": "src/environments/environment.ts", "with": "src/environments/environment.staging.ts" }] }
```

and give `production` the replacement with `environment.prod.ts`. GitHub environment variable `NG_CONFIGURATION` selects the configuration (`dev`, `staging`, `production`).

## 7. SPA fallback files (put both in `web/public/`, Angular copies them into the build)

`web.config` (MonsterASP / IIS):

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <system.webServer>
    <rewrite>
      <rules>
        <rule name="Angular routes" stopProcessing="true">
          <match url=".*" />
          <conditions logicalGrouping="MatchAll">
            <add input="{REQUEST_FILENAME}" matchType="IsFile" negate="true" />
            <add input="{REQUEST_FILENAME}" matchType="IsDirectory" negate="true" />
          </conditions>
          <action type="Rewrite" url="/index.html" />
        </rule>
      </rules>
    </rewrite>
  </system.webServer>
</configuration>
```

`staticwebapp.config.json` (Azure Static Web Apps):

```json
{
  "navigationFallback": {
    "rewrite": "/index.html",
    "exclude": ["/*.{js,css,ico,png,jpg,svg,woff2,json,txt}"]
  }
}
```

## 8. GitHub Environments checklist (`development`, `staging`, `prod`)

Repo-level:
- Variables: `DEPLOY_TARGET` (`azure` | `monsterasp`), `API_PROJECT` (e.g. `Shop.Api`), `WEB_DIST` (e.g. `web/dist/web/browser`)
- Secrets: `BOT_TOKEN` (bot identity, for release/back-merge PRs)

Per environment:
- Variable `NG_CONFIGURATION` (`dev` | `staging` | `production`)
- `azure`: secrets `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID` (an Entra app registration with a federated credential for `repo:<owner>/<repo>:environment:<env>`), `AZURE_STATIC_WEB_APPS_API_TOKEN`; variable `AZURE_WEBAPP_NAME`
- `monsterasp`: secrets `MONSTERASP_API_SERVER`, `MONSTERASP_API_SITE`, `MONSTERASP_API_USERNAME`, `MONSTERASP_API_PASSWORD` and the same four `MONSTERASP_WEB_*` for the Angular site (each from that site's *Visual Studio access* card, Web Deploy enabled)
- `prod`: add yourself as required reviewer on the environment, so a production deploy waits for a click

App settings per environment (`Jwt__SigningKey`, `ConnectionStrings__Default`, `Cors__Origins__0`) go in the host's configuration (App Service → Environment variables, MonsterASP control panel), never in the repo or workflow files.
