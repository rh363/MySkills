---
name: dependency-upgrade-discipline
description: Usa questa skill quando stai configurando o valutando upgrade di dipendenze (librerie, framework, base image Docker) o quando arriva una CVE. Enforce il pattern "upgrade come routine, non come emergenza" — Renovate/Dependabot batched settimanali, changelog review obbligatoria sui major, SCA (pip-audit / govulncheck / npm audit) in CI, auto-merge solo per dev deps a basso rischio.
---

# Dependency Upgrade Discipline

## Principio

Gli upgrade sono **routine settimanale**, non panico trimestrale. Un repo con 200 dipendenze outdated è un repo esposto: a ogni CVE sei in emergenza, e i salti da 3 major insieme costano 10× di uno solo.

## Perché

- **CVE → patch urgente** su una lib ferma a 3 major fa = upgrade obbligato + breaking change simultanei. Notti perse.
- **Lock-in silenzioso**: lasci andare Django 3, quando serve passare a 5 è un progetto di un mese.
- **Supply chain**: dependency confusion, typosquatting, maintainer compromessi. SCA deve vederli in CI, non in prod.
- **Cognitive load**: un solo bump alla volta, con CI verde, è una review da 5 minuti. Dieci bump insieme sono un dramma.

## Processo

### 1. Configura Renovate (preferito) o Dependabot
- `renovate.json` alla root
- Schedule: settimanale (lunedì notte, UTC locale)
- **Group** patch/minor insieme per ecosistema (meno PR)
- **Isolate** ogni major in PR separata con dependency dashboard approval
- **Auto-merge** solo dev deps (linter, test tool, formatter) su patch — mai su app deps

### 2. Review mandatoria del changelog per i major
- Leggi il CHANGELOG upstream (non solo la release note GitHub)
- Cerca `BREAKING`, `deprecat`, `remove`
- Se il changelog è povero, cerca issue aperte sul version bump
- Test suite verde è **necessaria ma non sufficiente** — code review obbligatoria

### 3. SCA (Software Composition Analysis) in CI
- Python: `pip-audit` oppure `safety`
- Node: `npm audit --audit-level=high`
- Go: `govulncheck ./...`
- Container: `trivy image ...` o `grype`
- Policy: **fail build su HIGH/CRITICAL**, MEDIUM = warning + review manuale, LOW = ignore con TTL

### 4. Policy di pinning
- **App**: lock file commitato (`poetry.lock`, `package-lock.json`, `pnpm-lock.yaml`, `go.sum`)
- **Library**: range nel manifest (`^1.2.0`), no lock file pubblicato
- **Base image Docker**: pin **a digest** (`python:3.12-slim@sha256:...`), non a tag mutabile
- Renovate aggiorna anche i digest automaticamente

### 5. Escape hatch: CVE su dep senza patch upstream
- Prima scelta: **override transitivo**
  - npm: `overrides`
  - yarn: `resolutions`
  - Python pip: `constraints.txt`
  - Poetry: version constraint diretta che forza la sub-dep
  - Go: `replace` in `go.mod`
- Seconda scelta: vendoring + patch applicata in build (`patch-package`, o manuale per Python)
- Sempre documenta il workaround con TODO + link a issue upstream + date di review

## Anti-rationalization

| Scusa | Risposta |
|---|---|
| "Non ho tempo di fare upgrade ora" | Proprio per questo serve Renovate: costo distribuito, non concentrato. Bump patch settimanali sono 10 min, non 10 ore. |
| "Auto-merge è pericoloso" | Su dev deps a patch, con CI verde, rischio < rischio di non aggiornare mai. Scope ristretto, non totale. |
| "Aggiungo SCA dopo" | Una CVE critical scoperta via tweet è il peggior momento per scoprire che SCA non c'era. |
| "Pin tutto con lock file, sono al sicuro" | Lock file non aggiornato = CVE vecchie esposte. Serve **lock + SCA + upgrade routine**, non una cosa sola. |
| "Il major è già approvato upstream, auto-merge ok" | I major richiedono review manuale sempre. Anche se ben testato, le breaking di comportamento esistono. |

## Esempio: `renovate.json` minimale

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:recommended", ":dependencyDashboard"],
  "schedule": ["before 6am on Monday"],
  "timezone": "Europe/Rome",
  "packageRules": [
    {
      "matchUpdateTypes": ["patch", "pin", "digest"],
      "groupName": "patch updates",
      "automerge": true,
      "platformAutomerge": true
    },
    {
      "matchUpdateTypes": ["minor"],
      "groupName": "minor updates"
    },
    {
      "matchUpdateTypes": ["major"],
      "dependencyDashboardApproval": true
    },
    {
      "matchDepTypes": ["devDependencies"],
      "matchUpdateTypes": ["minor", "patch"],
      "automerge": true
    }
  ],
  "vulnerabilityAlerts": {
    "enabled": true,
    "labels": ["security"]
  }
}
```

## Esempio: SCA in GitHub Actions

```yaml
# .github/workflows/security.yml
name: security
on:
  pull_request:
  schedule:
    - cron: '0 6 * * 1'  # lunedì mattina

jobs:
  deps:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: pip-audit (Python)
        run: pipx run pip-audit --strict -r requirements.txt
      - name: govulncheck (Go)
        run: |
          go install golang.org/x/vuln/cmd/govulncheck@latest
          govulncheck ./...
      - name: npm audit (Node)
        run: npm audit --audit-level=high
      - name: trivy image
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'myorg/myapp:${{ github.sha }}'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'
```

## Esempio: override transitivo Python per CVE

```toml
# pyproject.toml — fix CVE-2024-XXXXX su transitive dep
[tool.poetry.dependencies]
cryptography = ">=42.0.4"  # forzato per CVE, rimuovere quando dep parent aggiorna
# TODO: verificare il 2026-07-01 se ancora necessario (issue: upstream/repo#123)
```

## Esempio: override npm

```json
{
  "overrides": {
    "minimist": "^1.2.8"
  }
}
```

## Esempio: Go replace per CVE

```go
// go.mod
replace golang.org/x/net => golang.org/x/net v0.23.0
// TODO: rimuovere quando upstream bumpa (issue: ...)
```

## Exit criteria

- [ ] Renovate (o Dependabot) attivo, schedule settimanale, dashboard abilitata
- [ ] Grouping patch + isolation major configurati
- [ ] SCA in CI (pip-audit / govulncheck / npm audit / trivy) con fail su HIGH/CRITICAL
- [ ] Base image pin-at-digest
- [ ] Policy documentata per gestione CVE senza patch upstream
- [ ] Review periodica del dependency dashboard (mensile, non solo ad-hoc)
- [ ] Lock file committato per le app, non per le lib
