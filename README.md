# QuGents

Rollebaserede agenter og stack-specifikke skills til Claude Code, som kan genbruges på tværs af projekter (.NET + Angular nu, Python senere).

Repoet er både en **marketplace** (kataloget) og indeholder ét **plugin**, `qugents`.

## Princip: rolle ≠ stack

- **Agenter** (`agents/`) er roller: *hvem* og *hvordan man arbejder* (architecture, reviewer, tester ...).
- **Skills** (`skills/`) er viden om en stack: *hvad reglerne er* (dotnet-n-layer, python-... ).
- En agent preloader de skills den skal bruge via `skills:` i frontmatter. Så kan den samme rolle senere få en Python-skill uden at agenten skrives om.

## Indhold

```
.claude-plugin/marketplace.json        kataloget (navn: qugents)
plugins/qugents/
  .claude-plugin/plugin.json           plugin-manifest
  agents/
    planner.md                         epic → ordnede opgaver (vertical slices) med acceptkriterier
    architecture.md                    hvor kode hører til, strukturreview
    fullstack-developer.md             implementerer én opgave: API → Service → DAL + Angular
    tester.md                          NUnit + AAA + FakeItEasy, kun testkode
    reviewer.md                        korrekthed, sikkerhed, robusthed i et diff
    devops.md                          GitHub Actions, development → staging → prod, deploy til Azure/MonsterASP
  skills/
    dotnet-n-layer/                    API → Service → DAL, thin controllers, slices, C#-skabeloner (backend er altid Web API)
    dotnet-identity-jwt/               ASP.NET Core Identity + JWT + roterende refresh tokens (backend + Angular AuthStore)
    dotnet-new-solution/               opstart af ny løsning: webapi + Services + DataAccess + NUnit + SQLite (app.db) + Identity/JWT + Angular
    dotnet-testing/                    NUnit 4, Arrange-Act-Assert, FakeItEasy, test-skabeloner
    angular-frontend/                  Angular 22 + PrimeNG 22 + Tailwind 4 (signals, httpResource, Signal Forms), TS-skabeloner
    secure-code-review/                review-tjekliste .NET + Angular, severity, output-format
    ado-backlog/                       Azure DevOps (Scrum): typer, felter, tags, WIQL, git/PR-konventioner
    ado-plan/                          PBI'er tagget agent-plan → child Tasks (agent-proposed)
    ado-deliver/                       agent-ready Task → holdet → PR linket til Task'en
    devops-github-actions/             3-branch-strategi, CI/deploy/release/back-merge-workflows, rulesets, environments
```

### Agenter

| Agent | Rolle | Redigerer | Preloader | Model |
|---|---|---|---|---|
| `qugents:planner` | Bryder en epic/feature ned i opgaver | Nej | dotnet-n-layer, angular-frontend | sonnet |
| `qugents:architecture` | Placering, lag, DI, udskillelse af moduler | Nej | dotnet-n-layer | claude-opus-5-5 |
| `qugents:fullstack-developer` | Bygger én opgave i ét repo (API eller Web) | Ja | dotnet-n-layer, angular-frontend | inherit (sessionens model) |
| `qugents:tester` | Skriver og kører tests, finder huller | Kun testkode | dotnet-testing | sonnet |
| `qugents:reviewer` | Verdict + fund efter alvor | Nej | secure-code-review | sonnet |
| `qugents:verifier` | Tjekker acceptkriterierne før en agent-merge (PASS/FAIL pr. kriterie) | Nej | dotnet-testing | sonnet |
| `qugents:devops` | CI/CD, branches, environments, release-PR'er | Ja (workflows/scripts) | devops-github-actions | sonnet |

`model` kan ændres i hver agents frontmatter (`sonnet`, `opus`, `haiku`, `inherit` eller et fuldt model-id som `claude-opus-5-5`).

### Skills

| Skill | Kald direkte |
|---|---|
| `dotnet-n-layer` | `/qugents:dotnet-n-layer` |
| `dotnet-identity-jwt` | `/qugents:dotnet-identity-jwt` |
| `dotnet-new-solution` | `/qugents:dotnet-new-solution` |
| `dotnet-testing` | `/qugents:dotnet-testing` |
| `angular-frontend` | `/qugents:angular-frontend` |
| `secure-code-review` | `/qugents:secure-code-review` |
| `ado-plan` | `/qugents:ado-plan [PBI-id]` |
| `ado-deliver` | `/qugents:ado-deliver [Task-id]` |
| `ado-backlog` | (konventioner, hentes af de to ovenfor) |
| `devops-github-actions` | `/qugents:devops-github-actions` |

## To repos: backend og frontend

Backend (.NET Web API) og frontend (Angular) ligger altid i hvert sit repo, som søskende-mapper:

| Repo | Indhold | Eksempel |
|---|---|---|
| `<App>` | .NET-solution (API, Services, DataAccess, Tests) | `C:\_Github\Pos` |
| `<App>-web` | Angular + PrimeNG + Tailwind i repo-roden | `C:\_Github\Pos-web` |

En opgave hører altid til ét repo, så én opgave er én PR. En feature bliver typisk til to opgaver: `T1 — API: …` (endpoint) og `T2 — Web: …` (skærm), hvor T2 afhænger af T1. Begge repos har samme branches, rulesets og environments, og hver sine CI/deploy-workflows. Ved release går backend først.

## Arbejdsgang med holdet

```
planner ──► (architecture: tjek planen) ──► fullstack-developer ──► tester ──► reviewer ──► PR til development
                                               ▲                        │
                                               └──── fejl fundet ◄──────┘
```

Eksempel i Claude Code:

> Brug planner til at planlægge "ejere kan arkivere produkter". Tag derefter T1: lad fullstack-developer bygge den, tester skrive og køre tests, og reviewer reviewe diffet. Stop og vis mig resultatet før næste opgave.

Hovedagenten (din session) styrer rækkefølgen. Hver agent starter i en frisk kontekst og returnerer et resumé, så tester og reviewer ser koden med friske øjne.

## Installation i Claude Code

QuGents er et plugin til **Claude Code** (CLI'en `claude`). Kommandoerne virker i cmd, PowerShell og bash.

### Forudsætninger

- Claude Code installeret og logget ind: kør `claude` én gang og log ind, hvis den beder om det.
- Git og adgang til github.com (QuGents hentes derfra).

### Trin for trin (pr. projekt, anbefalet)

1. Åbn en terminal i projektets rodmappe (ved to repos: **backend-repoet**, det er der Claude Code startes).
2. Tilføj marketplacen og installér pluginet:

   ```bash
   claude plugin marketplace add GitQuuu/QuGents --scope project
   claude plugin install qugents@qugents --scope project
   ```

   Siger `marketplace add`, at den allerede findes, så kør `claude plugin marketplace update qugents` og fortsæt med `install`.
3. Tjek installationen:

   ```bash
   claude plugin list
   ```

   `qugents@qugents` skal stå som **enabled**.
4. Start Claude Code (`claude`) og skriv `/qugents:`. Du skal se `ado-plan`, `ado-deliver`, `dotnet-n-layer` osv. Agenterne ses med `/agents` (fx `qugents:planner`).
5. Commit `.claude/settings.json`, så projektet husker, at det bruger QuGents.

Resultatet i `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "qugents": { "source": { "source": "github", "repo": "GitQuuu/QuGents" } }
  },
  "enabledPlugins": { "qugents@qugents": true }
}
```

### Ny maskine eller kollega

`.claude/settings.json` i repoet *slår* QuGents til, men pluginet skal stadig *hentes* på hver maskine. Når en mappe åbnes første gang og godkendes som betroet, tilbyder Claude Code normalt at installere det. Kommer der ingen forespørgsel (fx fordi mappen allerede er betroet), så kør trin 2–4 ovenfor. Det er ufarligt at køre dem igen.

### Inde fra Claude Code

Skriv `/plugin` og brug menuen: **Marketplaces → Add** `GitQuuu/QuGents`, derefter **Discover → qugents → Install** og vælg scope. Eller direkte: `/plugin marketplace add GitQuuu/QuGents` og `/plugin install qugents@qugents`. Genstart Claude Code bagefter.

### Fejlfinding

| Symptom | Løsning |
|---|---|
| `/qugents:` viser ingenting | Pluginet er ikke hentet på maskinen. Kør trin 2–3, og start `claude` igen |
| `claude plugin list` viser qugents som disabled | `claude plugin enable qugents@qugents --scope project` |
| "Marketplace already exists" | `claude plugin marketplace update qugents` |
| Nye ændringer i QuGents ses ikke | Se **Opdatering** nedenfor, og genstart Claude Code |
| Install fejler med en politik-/organisationsfejl | Organisationen (Claude Team/Enterprise) tillader ikke eksterne marketplaces. En admin skal godkende `GitQuuu/QuGents` |

### Kun for dig i ét projekt

Brug `--scope local` i stedet for `--scope project`. Så havner det i `.claude/settings.local.json`, som ikke committes.

### På hele maskinen (alle projekter)

```bash
claude plugin marketplace add GitQuuu/QuGents
claude plugin install qugents@qugents
```

Eller inde i Claude Code: `/plugin marketplace add GitQuuu/QuGents` og `/plugin install qugents@qugents`.

### Fjern den maskin-globale installation

```bash
claude plugin uninstall qugents@qugents
```

Uden `--scope` rammer den den globale installation. Kør den, hvis QuGents kun skal være aktiv i de projekter, der selv slår den til. Fra et projekt fjernes den med `claude plugin uninstall qugents@qugents --scope project`.

## Product Owner-flow med Azure DevOps (Scrum)

Du skriver backloggen i Azure DevOps; holdet planlægger, bygger og åbner PR'er. Intet bygges uden dit `agent-ready`, og intet merges uden din godkendelse.

| Trin | Hvem | Hvad |
|---|---|---|
| 1 | Dig | Opret en **Product Backlog Item** med Description + Acceptance Criteria. Tag den `agent-plan` |
| 2 | `/qugents:ado-plan` | planner bryder PBI'en ned i child **Tasks** (`T1 — API: …`, `T2 — Web: …`) med kriterier, kontrakt og testcases, tagget `agent-proposed` og `repo-api` eller `repo-web`. PBI'en får `agent-planned` + en kommentar |
| 3 | Dig | Ret opgaverne til, og sæt **`agent-ready` på PBI'en** for at godkende alle dens Tasks på én gang (tag en Task `agent-hold` for at springe den over). Du kan også sætte `agent-ready` på enkelte Tasks |
| 4 | `/qugents:ado-deliver` | Tager næste godkendte Task (i `T<n>`-rækkefølge) og arbejder i dens repo: branch `agent/<id>-…` → fullstack-developer → tester → reviewer (maks 2 rette-runder) → commit med `AB#<id>` → push → PR til `development` med `Fixes AB#<id>`. **Er PBI'en `agent-ready`:** venter på grøn CI → verifier tjekker acceptkriterierne → squash-merge til `development` → næste Task, indtil PBI'en er færdig (`agent-done`) |
| 5 | Dig | PBI-godkendt: tjek de manuelle punkter i PBI-kommentaren og luk PBI'en. Task-godkendt eller `agent-no-merge` på PBI'en: review og merge hver PR selv |
| — | Ved problemer | Task/PBI får `needs-human` + en kommentar med spørgsmålet eller fejlen |

### Opsætning (én gang)

Kør i **backend-repoet**. Claude Code startes altid derfra, med adgang til frontend-mappen:

```bash
# Azure DevOps MCP-server (lokal, Microsofts officielle). Skriver .mcp.json - commit den.
# På Windows (cmd/PowerShell): erstat "npx" med "cmd /c npx".
claude mcp add --scope project --transport stdio azure-devops -- npx -y @azure-devops/mcp <organisation> -d core work work-items
```

Giv Claude Code adgang til frontend-repoet: start med `claude --add-dir ../<App>-web`, eller læg `"permissions": { "additionalDirectories": ["../<App>-web"] }` i backend-repoets `.claude/settings.json`.

Første gang et værktøj bruges, åbner browseren et Microsoft-login (eller tilføj `--authentication azcli` og brug `az login`).

**Personlig Microsoft-konto (hotmail/outlook/live)?** Så afviser login'et med "You can't sign in here with a personal account". Brug en PAT i stedet: tilføj `--authentication pat` til MCP-argumenterne, opret en PAT i Azure DevOps (User settings → Personal access tokens, scope *Work Items: Read & write* og *Project and Team: Read*), og sæt miljøvariablen `PERSONAL_ACCESS_TOKEN` til base64 af `ado:<pat>` (aldrig i `.mcp.json`). PowerShell:

```powershell
$pat = Read-Host "Azure DevOps PAT" -AsSecureString
$plain = [Net.NetworkCredential]::new('', $pat).Password
[Environment]::SetEnvironmentVariable('PERSONAL_ACCESS_TOKEN', [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes("ado:$plain")), 'User')
```

Åbn en ny terminal bagefter, så Claude Code ser variablen. Tilføj så i begge repos' `CLAUDE.md` (stierne er relative til det repo, filen ligger i):

```markdown
## Azure DevOps
- Organization: <organisation>
- Project: <projekt>
- Team: <team>
- Code host: GitHub
- Backend repository: <owner>/<App> (local: .)
- Frontend repository: <owner>/<App>-web (local: ../<App>-web)
- Integration branch: development
```

Branches i begge repos: `development` (default, agenternes PR'er) → `staging` → `prod`. devops-agenten sætter rulesets op: kun `prod` kræver PR (med grøn `ci`), og ingen af de tre branches kan slettes eller force-pushes. Agenterne leverer stadig altid via PR'er mod `development`, så de venter på dig. Giv agenterne deres egen GitHub-identitet (GitHub App eller maskinbruger), da GitHub ikke lader dig godkende dine egne PR'er. Installér Azure Boards-appen på begge GitHub-repos, så `Fixes AB#<id>` lukker Task'en ved merge.

### Dagligt

- `/qugents:ado-plan` (eller "planlæg backloggen") → læs opgaverne i Boards → sæt `agent-ready`.
- Sæt `agent-ready` på PBI'en → `/qugents:ado-deliver <PBI-id>` → holdet bygger, verificerer og merger alle Tasks. Stopper kun ved `needs-human`.
- Vil du selv merge, så tag PBI'en `agent-no-merge` også.
- Stol du på planen fra start? Sæt `agent-ready` direkte på en uplanlagt PBI: holdet planlægger og bygger den uden en separat gennemgang af Tasks.

## Opdatering

Hent den nyeste udgave efter et push til QuGents (genstart Claude Code bagefter):

```bash
claude plugin marketplace update qugents
claude plugin update qugents@qugents
```

`plugin.json` har bevidst ingen `version`. Claude Code bruger så commit-SHA som version, så hvert push er en ny udgave. Når QuGents er stabilt, kan du skifte til semver (`"version": "1.0.0"`) og kun bumpe ved releases.

## Tilføj en ny agent eller skill

1. Agent: læg `plugins/qugents/agents/<rolle>.md` med frontmatter (`name`, `description`, `tools`, `model`, `skills`).
2. Skill: læg `plugins/qugents/skills/<navn>/SKILL.md` med frontmatter (`name`, `description`). Lange detaljer i `references/`.
3. Validér: `claude plugin validate .`
4. Test lokalt: `claude --plugin-dir ./plugins/qugents`

## Brug

- Spørg normalt, fx *"Hvor skal en ny use case 'ArchiveProduct' ligge?"* eller *"Review mit diff"*. Claude uddelegerer selv til den agent, hvis beskrivelse passer.
- Eller eksplicit: *"Brug reviewer-agenten på `git diff main...HEAD`"*.
- Skills bruges også direkte af hovedagenten, når den skriver C# eller Angular.

## Status og verifikation

- Alle agenter er testet: skills preloades (debug-log), planner lavede en brugbar plan, reviewer fandt plantede fejl (IDOR, `AllowAnonymous` med persondata, null-crash, hent-alt-forespørgsel).
- Angular-skabelonerne bygger med Angular 22.2 + PrimeNG 22.1 + Tailwind 4.3 (`strictTemplates`), spec'erne kører grønt i Vitest, og en browser-test bekræfter: Tailwind-klasser vinder over PrimeNG, Signal Forms binder til PrimeNG-inputs, toast/confirm virker, og ProblemDetails vises som toast.
- fullstack-developer byggede en ny detaljeside med PrimeNG + Tailwind ud fra skillen alene; build og tests grønne.
- **PrimeNG 22 kræver en PrimeUI-licensnøgle** (`providePrimeNG({ license })`). Community License er gratis under 1 mio. USD omsætning, < 5 udviklere og < 10 ansatte; ellers betalt licens pr. udvikler. Uden nøgle vises et licens-banner.
- C#-skabelonerne (dotnet-n-layer, dotnet-testing) er ikke kompileret endnu. Tjek dem første gang de bruges.
