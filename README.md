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

I Claude Code (én gang pr. maskine):

```
/plugin marketplace add GitQuuu/QuGents
/plugin install qugents@qugents
```

Eller fra en terminal:

```bash
claude plugin marketplace add GitQuuu/QuGents
claude plugin install qugents@qugents
```

Skal et projekt altid have QuGents (også i CI og for kollegaer), så læg det i projektets `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "qugents": { "source": { "source": "github", "repo": "GitQuuu/QuGents" } }
  },
  "enabledPlugins": { "qugents@qugents": true }
}
```

## Opdatering

`plugin.json` har bevidst ingen `version`. Claude Code bruger så commit-SHA som version, og alle får den nye udgave ved `/plugin update` eller auto-update. Når QuGents er stabilt, kan du skifte til semver (`"version": "1.0.0"`) og kun bumpe ved releases.

## Tilføj en ny agent eller skill

1. Agent: læg `plugins/qugents/agents/<rolle>.md` med frontmatter (`name`, `description`, `tools`, `model`, `skills`).
2. Skill: læg `plugins/qugents/skills/<navn>/SKILL.md` med frontmatter (`name`, `description`). Lange detaljer i `references/`.
3. Validér: `claude plugin validate .`
4. Test lokalt: `claude --plugin-dir ./plugins/qugents`

## Brug

- Spørg normalt, fx *"Hvor skal en ny use case 'ArchiveProduct' ligge?"* eller *"Review min ændring mod arkitekturen"*. Claude uddelegerer selv til `qugents:architecture`, når beskrivelsen passer.
- Eller eksplicit: *"Brug architecture-agenten til at reviewe `git diff main`"*.
- Skillen bruges også direkte af hovedagenten, når den skriver C# i et N-layer projekt.
