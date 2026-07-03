# Internet.nl Codex Plugin - Werkinstructies

## Taal

Alle content in deze repo is in het Nederlands: skill descriptions, bodytekst,
commit messages en documentatie. Gebruik Nederlands tenzij de gebruiker expliciet
in het Engels communiceert.

## Repo structuur

- `.plugin/plugin.json` - neutrale bronmetadata voor versie en beschrijving
- `.codex-plugin/plugin.json` - Codex plugin manifest, gegenereerd uit `.plugin/plugin.json`
- `.claude-plugin/plugin.json` - Claude Code manifest, gegenereerd uit `.plugin/plugin.json`
- `.cursor-plugin/plugin.json` - Cursor manifest, gegenereerd uit `.plugin/plugin.json`
- `.agents/plugins/marketplace.json` - Codex marketplace voor installatie uit deze repo
- `skills/inet/SKILL.md` - meta-skill voor overzicht en routing
- `skills/inet-<domein>/SKILL.md` - domein-skills
- `skills/inet-<domein>/reference.md` - achtergrondinfo per domein
- `scripts/generate_plugin.py` - genereert platform-specifieke plugin manifests
- `scripts/extract_urls.py` - extraheert monitorbare URLs uit skill-bestanden
- `scripts/monitor_content.py` - detecteert content-wijzigingen
- `tests/` - pytest tests voor de scripts

## Codex conventies

Codex gebruikt de `name` en `description` uit elke `SKILL.md` om skills te
triggeren. Houd descriptions kort en triggerbaar met Nederlandse termen en
technische eigennamen zoals DNSSEC, DMARC, DANE, TLS en security.txt.

Gebruik in Codex-documentatie `$inet`, `$inet-web`, `$inet-mail`, `$inet-api`
en `$inet-toolbox`. Claude Code slash-command voorbeelden horen alleen in
`CLAUDE.md` of expliciet Claude-gerichte documentatie.

Na wijzigingen aan `.plugin/plugin.json`:

```bash
python scripts/generate_plugin.py
python scripts/generate_plugin.py --check
```

## Development

Vereisten:

- Python 3.12+
- `uv` als package manager

Controleer wijzigingen met:

```bash
uvx ruff@0.9.7 check scripts/ tests/
uvx ruff@0.9.7 format --check scripts/ tests/
uv run --group dev pytest -v
python scripts/generate_plugin.py --check
```

## Contentstrategie

De GitHub repos van `internetstandards` zijn de bron van waarheid. Skills
bevatten samenvattingen, routing en praktische configuratievoorbeelden; actuele
details worden waar nodig live opgehaald via `gh api`, `curl`, browser/webfetch
of de officiële internet.nl-bronnen.

Deze plugin is een concept en geen officieel product van internet.nl. Houd de
disclaimer en verwijzingen naar Forum Standaardisatie en internet.nl intact.
