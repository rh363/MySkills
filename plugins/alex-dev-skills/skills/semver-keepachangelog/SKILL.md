---
name: semver-keepachangelog
description: Usa questa skill quando stai decidendo un version bump, preparando una release, scrivendo/aggiornando il CHANGELOG, o definendo la policy di breaking change. Enforce SemVer 2.0 strict (MAJOR su breaking, MINOR su additive backward-compat, PATCH su bugfix) e il formato Keep a Changelog (sezioni Added/Changed/Deprecated/Removed/Fixed/Security con [Unreleased] in cima, date ISO, link compare in fondo). Copre anche tag git, release workflow, ADR minimi e MIGRATION.md su MAJOR.
---

# SemVer + Keep a Changelog

## Principio

La versione è un **contratto** con chi usa il tuo codice (librerie, API, plugin). **SemVer 2.0** definisce la semantica di ogni bump. **Keep a Changelog** è l'artefatto human-readable che spiega cosa cambia tra due versioni. Entrambi sono disciplina editoriale, non automatismo: un changelog autogenerato dai commit produce rumore, non segnale.

## Perché

- Un bump sbagliato (breaking in patch) rompe downstream senza preavviso → fiducia persa, issue rumorose
- CHANGELOG assente o incompleto = utente costretto a leggere `git log` → adozione lenta, PR duplicate
- Breaking non annunciato = migration cost trasferito a N utenti senza consenso

## SemVer 2.0 — regole del bump

Formato: `MAJOR.MINOR.PATCH` (+ opzionale `-<prerelease>` e `+<build>`).

| Bump | Quando | Esempi |
|---|---|---|
| **MAJOR** | Breaking change pubblico | Rimozione API, rename campo, cambio tipo, nuovo parametro obbligatorio, cambio comportamento osservabile |
| **MINOR** | Additive backward-compat | Nuovo endpoint, nuovo metodo, nuovo campo opzionale, **deprecazione** (ma non rimozione) |
| **PATCH** | Bugfix backward-compat | Fix bug senza cambiare l'API, fix security senza cambiare firma |

**Zero-major (`0.x.y`)**: qualunque bump può breakare, SemVer non garantisce nulla. Usa per pre-1.0, ma appena il codice ha utenti esterni stabili, passa a `1.0.0` — è un **commitment**.

**Pre-release**: `1.0.0-beta.1`, `2.0.0-rc.3`. Instabili per definizione, non aggiornare dipendenze di produzione a versioni `-*`.

## Keep a Changelog — formato

File `CHANGELOG.md` alla root. In **inglese** (lingua internazionale per release notes, anche se il codice/commit sono in italiano).

```markdown
# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- New `/users/{id}/roles` endpoint returning aggregated role claims.

### Fixed
- Handle empty JWKS response gracefully (previously raised 500).

## [1.2.0] - 2026-04-17

### Added
- Support for OAuth 2.1 PAR (Pushed Authorization Requests).

### Changed
- Upgraded Django 4.2 → 5.0 (no API change, requires Python 3.10+).

### Deprecated
- `UserService.get_by_legacy_id` — will be removed in 2.0. Use `get_by_id`.

## [1.1.1] - 2026-03-20

### Fixed
- Race condition in `celery_task_idempotency` when two workers pick the same task.

### Security
- Bumped `cryptography` to 42.0.4 (CVE-2024-XXXXX).

[Unreleased]: https://github.com/owner/repo/compare/v1.2.0...HEAD
[1.2.0]: https://github.com/owner/repo/compare/v1.1.1...v1.2.0
[1.1.1]: https://github.com/owner/repo/compare/v1.1.0...v1.1.1
```

### Sezioni (usa solo quelle applicabili alla release)

- **Added** — feature nuove
- **Changed** — modifiche a feature esistenti
- **Deprecated** — feature che verrà rimossa in una major futura
- **Removed** — feature rimossa in questa release
- **Fixed** — bugfix
- **Security** — fix di vulnerabilità (linka CVE se pubblico)

### Regole editoriali

- `[Unreleased]` sempre in cima, link a `compare/<ultima-tag>...HEAD`
- Ogni release ha **data ISO** (`YYYY-MM-DD`), mai "TBD", "soon", "coming"
- Link in fondo al file, non inline
- **Una entry = una frase chiara**, mai "various fixes" o "improvements"
- Riferimenti issue/PR opzionali ma utili: `Fixed login redirect (#123)`
- Scrivi le entry **al momento del PR**, non "a fine sprint raccolgo tutto"

## Release workflow

1. **Durante sviluppo**: ogni PR che merita menzione aggiorna `## [Unreleased]`
2. **Prima della release**:
   - Decidi bump (MAJOR/MINOR/PATCH) in base a **cosa c'è in `[Unreleased]`**: se c'è una entry breaking, è MAJOR; se tutto è Added/Changed non-breaking, è MINOR; se solo Fixed/Security, è PATCH
   - Sposta `[Unreleased]` → `[X.Y.Z] - YYYY-MM-DD`, aggiungi nuovo `[Unreleased]` vuoto
   - Bump version in **tutti** i manifest: `pyproject.toml`, `package.json`, `plugin.json`, `go.mod` (per moduli), `Cargo.toml`, `.claude-plugin/marketplace.json`
   - Aggiorna link compare in fondo al CHANGELOG
3. **Tag git**: `git tag -a v1.2.0 -m "Release 1.2.0"` + `git push origin v1.2.0`
4. **Release CI**: la tag innesca build/publish (PyPI, npm, container registry, GitHub Release)
5. **GitHub Release notes = copia della sezione CHANGELOG** (non rigenerare, è già scritta)

### Automation — quando sì, quando no

- ✅ CI che blocca merge se `[Unreleased]` non è stato toccato (su label `changelog-required`)
- ✅ `release-please` / `changeset` per bumpare i file version a partire dal CHANGELOG scritto a mano
- ❌ Autogenerare CHANGELOG da commit message: produce entries rumorose, perde nuance, incentiva commit message banali
- ❌ Autobump version senza review umana quando il contenuto è breaking

## Breaking change policy

- **Annunciato almeno una release prima** come `### Deprecated`
- **Deprecation window** documentata nella entry (es. "will be removed in 2.0, target date 2026-06-01")
- Fornire path di migrazione dove possibile: shim, adapter, script di codemod
- `MIGRATION.md` alla root per breaking **non triviale** (cambio schema DB, rename massivo, cambio auth flow)

## Documentazione che accompagna ogni release

- **README** aggiornato (install, quickstart, breaking change evidenziati in alto)
- **CHANGELOG** aggiornato (sempre, non negoziabile)
- **MIGRATION.md** se MAJOR con breaking non triviali
- **ADR** se la release contiene decisioni architetturali significative (`docs/adr/NNNN-<slug>.md`)

### ADR (Architecture Decision Record) — formato minimo

```markdown
# ADR-0012: Passaggio da Celery a Temporal per workflow long-running

- **Status**: Accepted
- **Date**: 2026-04-17
- **Deciders**: backend team

## Context
Celery non supporta nativamente workflow multi-step con compensazione e stato
persistente. I nostri task di provisioning richiedono retry granulare e
ispezione real-time dello stato.

## Decision
Migriamo i workflow long-running (>5 min) a Temporal. Celery resta per task
idempotenti brevi.

## Consequences
- +: observability nativa, retry policy ricche, workflow versioning
- -: nuovo runtime da operare, learning curve
- -: migrazione incrementale (stimata 6 mesi)
```

Gli ADR sono **append-only**: una decisione superata si marca `Superseded by ADR-NNNN`, non si riscrive.

## Anti-rationalization

| Scusa | Risposta |
|---|---|
| "È piccolo, lo chiamo 1.0.0 subito" | Se è breakable senza preavviso → resta 0.x. Passare a 1.0 è un commitment stabile. |
| "PATCH perché solo un campo nuovo opzionale" | Campo nuovo = additive = **MINOR**. Anche se è un field. |
| "Il breaking è obvious, skippo deprecation" | Tu sai cosa hai cambiato; gli utenti no. Deprecation window, sempre. |
| "CHANGELOG lo genero da git log a fine sprint" | Risultato: entries rumorose, signal perso. Scrivilo in PR review, non dopo. |
| "Tanto metto `^1.2.0` su tutte le dep, SemVer mi tutela" | SemVer ti tutela SE tu lo rispetti quando rilasci. Doppia responsabilità: consumer E publisher. |
| "0.x.y quindi posso rompere a piacere" | Tecnicamente sì, ma se hai utenti reali rompere silenziosamente = costo reputazionale. Annuncia comunque. |
| "Il CHANGELOG è in italiano perché il team è italiano" | Se il progetto è pubblico (anche solo open source interno), inglese. Coerenza con release note GitHub. |

## Esempio: decisione di bump

**Contenuto di `[Unreleased]`**:
- Added: nuovo endpoint `GET /v1/users/{id}/roles`
- Fixed: race condition in Celery idempotency
- Removed: campo `User.legacyId` (deprecato da 3 release)

→ **MAJOR bump** (Removed → breaking). L'additive + fix non "annacquano" il breaking: la regola è "il bump più alto vince".

## Esempio: pre-commit hook che blocca push senza `[Unreleased]`

```yaml
# .pre-commit-config.yaml
- repo: local
  hooks:
    - id: changelog-unreleased
      name: CHANGELOG has [Unreleased] section
      entry: scripts/check_changelog.sh
      language: system
      stages: [pre-push]
      always_run: true
```

```bash
# scripts/check_changelog.sh
#!/usr/bin/env bash
set -euo pipefail
if ! grep -q "^## \[Unreleased\]" CHANGELOG.md; then
  echo "ERROR: CHANGELOG.md manca la sezione [Unreleased]" >&2
  exit 1
fi
```

## Esempio: git tag annotata per release

```bash
# Commit CHANGELOG + version bump
git add CHANGELOG.md pyproject.toml
git commit -m "release: 1.2.0"

# Tag annotata (non lightweight)
git tag -a v1.2.0 -m "Release 1.2.0

$(sed -n '/## \[1.2.0\]/,/## \[/p' CHANGELOG.md | sed '$d')"

git push origin main
git push origin v1.2.0
```

## Exit criteria

- [ ] `CHANGELOG.md` presente alla root, formato Keep a Changelog 1.1.0
- [ ] `[Unreleased]` sempre in cima, popolata prima di ogni merge non triviale
- [ ] Version bumpata coerentemente con SemVer 2.0 in **tutti** i manifest del repo
- [ ] Git tag `vX.Y.Z` (annotata, non lightweight) pushata
- [ ] Release CI attiva e verde sulla tag
- [ ] Breaking change → Deprecation → Removal con window ≥ 1 release minore
- [ ] ADR scritto per decisioni architetturali non ovvie, filed sotto `docs/adr/`
- [ ] Se MAJOR: `MIGRATION.md` con path di upgrade concreto
