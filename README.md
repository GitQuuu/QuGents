# QuGents

Rollebaserede agenter og stack-specifikke skills til Claude Code, som kan genbruges på tværs af projekter (.NET, Python ...).

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
    architecture.md                    rådgiver/reviewer for N-layer modular monolith (redigerer ikke filer)
  skills/
    dotnet-n-layer/
      SKILL.md                         API → Service → DAL, thin controllers, slices, regler
      references/code-templates.md     C#-skabeloner for hvert lag
      references/review-checklist.md   review-tjekliste + kendt drift
```

| Komponent | Navn i Claude Code |
|---|---|
| Agent | `qugents:architecture` |
| Skill | `qugents:dotnet-n-layer` (`/qugents:dotnet-n-layer`) |

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

- Spørg normalt, fx *"Hvor skal en ny use case 'ArchiveProduct' ligge?"* eller *"Review min ændring mod arkitekturen"*. Claude uddelegerer selv til `qugents:architecture`, når beskrivelsen passer.
- Eller eksplicit: *"Brug architecture-agenten til at reviewe `git diff main`"*.
- Skillen bruges også direkte af hovedagenten, når den skriver C# i et N-layer projekt.
