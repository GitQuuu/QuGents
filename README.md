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
  skills/
    dotnet-n-layer/                    API → Service → DAL, thin controllers, slices, C#-skabeloner (backend er altid Web API)
    dotnet-identity-jwt/               ASP.NET Core Identity + JWT + roterende refresh tokens (backend + Angular AuthStore)
    dotnet-new-solution/               opstart af ny løsning: webapi + Services + DataAccess + NUnit + Identity/JWT + Angular
    dotnet-testing/                    NUnit 4, Arrange-Act-Assert, FakeItEasy, test-skabeloner
    angular-frontend/                  Angular 22 + PrimeNG 22 + Tailwind 4 (signals, httpResource, Signal Forms), TS-skabeloner
    secure-code-review/                review-tjekliste .NET + Angular, severity, output-format
```

### Agenter

| Agent | Rolle | Redigerer | Preloader | Model |
|---|---|---|---|---|
| `qugents:planner` | Bryder en epic/feature ned i opgaver | Nej | dotnet-n-layer, angular-frontend | sonnet |
| `qugents:architecture` | Placering, lag, DI, udskillelse af moduler | Nej | dotnet-n-layer | claude-opus-5-5 |
| `qugents:fullstack-developer` | Bygger én opgave fra database til UI | Ja | dotnet-n-layer, angular-frontend | inherit (sessionens model) |
| `qugents:tester` | Skriver og kører tests, finder huller | Kun testkode | dotnet-testing | sonnet |
| `qugents:reviewer` | Verdict + fund efter alvor | Nej | secure-code-review | sonnet |

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

## Arbejdsgang med holdet

```
planner ──► (architecture: tjek planen) ──► fullstack-developer ──► tester ──► reviewer ──► PR
                                               ▲                        │
                                               └──── fejl fundet ◄──────┘
```

Eksempel i Claude Code:

> Brug planner til at planlægge "ejere kan arkivere produkter". Tag derefter T1: lad fullstack-developer bygge den, tester skrive og køre tests, og reviewer reviewe diffet. Stop og vis mig resultatet før næste opgave.

Hovedagenten (din session) styrer rækkefølgen. Hver agent starter i en frisk kontekst og returnerer et resumé, så tester og reviewer ser koden med friske øjne.

## Installation

### Pr. projekt (følger med repoet)

Kør i projektets rodmappe:

```bash
claude plugin marketplace add GitQuuu/QuGents --scope project
claude plugin install qugents@qugents --scope project
```

Det skriver `.claude/settings.json`. Commit den, så får kollegaer og CI også QuGents, og andre projekter påvirkes ikke. Første gang en kollega åbner projektet, skal mappen godkendes som betroet, før pluginet hentes.

Resultatet i `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "qugents": { "source": { "source": "github", "repo": "GitQuuu/QuGents" } }
  },
  "enabledPlugins": { "qugents@qugents": true }
}
```

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
