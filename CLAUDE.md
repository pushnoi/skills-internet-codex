# Internet.nl Plugin - Werkinstructies

## Taal

Alle content in deze repo is in het **Nederlands**: skill descriptions, body tekst, commit messages, en documentatie. Gebruik Nederlands tenzij de gebruiker expliciet in het Engels communiceert.

## Repo structuur

- `.plugin/plugin.json` - Neutrale bronmetadata voor versie en beschrijving
- `.codex-plugin/plugin.json` - Codex plugin manifest (gegenereerd uit `.plugin/plugin.json`)
- `.claude-plugin/plugin.json` - Claude Code plugin manifest (gegenereerd uit `.plugin/plugin.json`)
- `.cursor-plugin/plugin.json` - Cursor plugin manifest (gegenereerd uit `.plugin/plugin.json`)
- `.agents/plugins/marketplace.json` - Codex marketplace voor installatie uit deze repo
- `skills/inet/SKILL.md` - Meta-skill (overzicht, routing)
- `skills/inet-<domein>/SKILL.md` - Domein-skills (4 stuks)
- `skills/inet-<domein>/reference.md` - Achtergrondinfo per domein
- `scripts/extract_urls.py` - Extraheert monitorbare URLs uit skill-bestanden
- `scripts/monitor_content.py` - Detecteert content-wijzigingen (GitHub API + HTTP checks)
- `tests/` - Pytest tests voor de scripts
- `.github/workflows/ci.yml` - CI: structuurvalidatie, ruff, markdownlint, pytest
- `.github/workflows/release-please.yml` - Automatische releases via conventional commits
- `.github/workflows/monitoring-content.yml` - Dagelijkse content monitoring (07:00 UTC)
- `.github/workflows/monitoring-links.yml` - Dagelijkse link checks met lychee (06:00 UTC)
- `.github/workflows/labeler.yml` - Automatische PR-labels op basis van gewijzigde bestanden

## Conventies voor skills

### SKILL.md opbouw
1. YAML frontmatter met `name`, `description`, `model: sonnet`, `allowed-tools`
2. Agent-instructie (2-3 zinnen, vetgedrukt: wanneer wordt deze skill gebruikt, wat moet de agent doen)
3. Korte intro (wat is dit domein, link naar Forum Standaardisatie)
4. Standaarden overzicht (per standaard: wat, waarom verplicht, hoe testen)
5. Repository tabel(len)
6. Implementatievoorbeelden - realistische configuratie en code
7. Veelvoorkomende problemen - troubleshooting, foutmeldingen
8. Achtergrondinfo (verwijzing naar reference.md)

Elke skill moet genoeg bevatten zodat een agent standaard-conforme configuratie kan genereren zonder externe bronnen op te halen. Encyclopedische uitleg hoort in `reference.md`, niet in `SKILL.md`.

### Repository tabel format

```markdown
| Repository | Beschrijving |
|-----------|-------------|
| [Naam](https://github.com/internetstandards/REPO) | Omschrijving |
```

### reference.md patroon
Achtergrondkennis die niet direct nodig is voor configuratie-generatie wordt verplaatst naar `reference.md`. Denk aan:
- Technische details per protocol (DMARC alignment, DKIM key rotation, DANE TLSA records)
- Relatie tussen protocollen (SPF+DKIM+DMARC samenwerking)
- Server-specifieke configuratie (Postfix, Exchange, Nginx, Apache)
- Exploratie commando's (`dig`, `openssl`, `gh api`)

### Description triggers
Descriptions bevatten zowel Nederlandse als technische Engelse triggerwoorden zodat de skill bij relevante vragen wordt geactiveerd. Houd descriptions kort: doel **~150 chars, max 200**. Eén functionele zin met de eigennamen die mensen daadwerkelijk gebruiken (DMARC, DNSSEC, security.txt) is genoeg; voeg alleen een korte triggerstaart toe voor termen die niet uit de hoofdzin volgen. Geen "Triggers:"-staarten die de hoofdzin dupliceren — die vreten skill listing budget op zonder triggerwaarde toe te voegen.

### Allowed tools
Standaard set:
- `Bash(gh api *)`, `Bash(curl -s *)`, `WebFetch(*)`
- `Bash(dig *)`, `Bash(openssl *)` voor DNS/TLS diagnostiek

### Maximale omvang
SKILL.md bestanden mogen maximaal **500 regels** bevatten. Grotere content hoort in `reference.md`.

## Content strategie

**On-demand ophalen, niet opslaan.** De GitHub repos van internetstandards zijn de bron van waarheid. Skills bevatten samenvattingen en structuur; actuele content wordt live opgehaald via `gh api` of `WebFetch`.

## GitHub organisatie

Alle repos staan onder: `https://github.com/internetstandards/`
Internet.nl website: `https://internet.nl/`
Toolbox wiki: `https://github.com/internetstandards/toolbox-wiki/`

## Development

### Vereisten

- Python 3.12+
- [uv](https://docs.astral.sh/uv/) als package manager
- `uv sync` om dependencies te installeren

### Linting en tests

```bash
uvx ruff@0.9.7 check scripts/ tests/       # Linting
uvx ruff@0.9.7 format --check scripts/ tests/  # Format check
uv run pytest -v                         # Tests
```

Pre-commit hooks draaien automatisch ruff + markdownlint bij elke commit.

### CI/CD

**Branch protection** is actief op `main` met verplichte status checks: `validate`, `lint-python`, `lint-markdown`, `test-python`. Directe pushes naar main zijn geblokkeerd — alle wijzigingen gaan via PRs.

**Releases** worden automatisch beheerd door [release-please](https://github.com/googleapis/release-please):
1. Gebruik [Conventional Commits](https://www.conventionalcommits.org/): `feat:`, `fix:`, `chore:`, `docs:`
2. release-please maakt automatisch een Release PR aan met gebumpte versies en CHANGELOG
3. Bij merge van Release PR: GitHub Release + git tag + versie gebumpt in `plugin.json`, `pyproject.toml`, `publiccode.yml` en `.codex-plugin/plugin.json`

**Belangrijk:** Release PRs van release-please hebben geen CI checks (GITHUB_TOKEN pushes triggeren geen workflows). Push een lege commit om CI te triggeren:
```bash
gh pr checkout <nr> && git commit --allow-empty -m "chore: trigger CI" && git push && git checkout main
```

## Plugin testen

Verificatievragen:
- "Hoe test ik mijn website op internet.nl?" -> moet `/inet-web` triggeren
- "Hoe configureer ik DMARC?" -> moet `/inet-mail` of `/inet-toolbox` triggeren
- "Hoe gebruik ik de internet.nl API?" -> moet `/inet-api` triggeren
- "Welke internetstandaarden zijn er?" -> moet `/inet` triggeren
- "Hoe stel ik DANE in?" -> moet `/inet-toolbox` triggeren
