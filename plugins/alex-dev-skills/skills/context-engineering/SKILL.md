---
name: context-engineering
description: Usa questa skill quando stai setuppando un nuovo progetto con Claude Code, quando una sessione inizia a divagare, o quando le risposte di Claude diventano vaghe/ripetitive. Enforce una strategia per popolare CLAUDE.md, decidere cosa includere nel contesto, e mantenere il context window pulito. Evita context overload e ripetizioni.
---

# Context Engineering

## Principio

**Il context window è una risorsa scarsa, non una discarica.** Ogni token consumato da rumore è un token che non può essere speso in ragionamento utile. L'obiettivo è **massima densità informativa** per la task corrente, non massima copertura del progetto.

Regole cardine:
1. **CLAUDE.md è l'orientamento iniziale**, non la documentazione completa
2. **Packing selettivo**: includi per task, non per progetto
3. **Freshness > completezza**: informazioni datate sono peggio di mancanti
4. **Delegate, don't duplicate**: punta a file/comandi, non replicarli inline

## Processo

### 1. Struttura `CLAUDE.md` (orientamento base)
Deve rispondere in < 200 righe a queste domande:

1. **Cos'è il progetto** (1-2 righe)
2. **Stack tecnico** principale (linguaggi, framework, DB)
3. **Come si lancia/testa** (comandi esatti)
4. **Convenzioni** non ovvie (naming, struttura cartelle, pattern)
5. **Cosa NON fare** (anti-pattern, trappole del progetto)
6. **Riferimenti**: dove cercare info più dettagliate

### 2. Packing per task
Prima di iniziare una task significativa, decidi cosa caricare:

| Tipo di task | Includi | Escludi |
|---|---|---|
| Bug fix su endpoint X | `CLAUDE.md`, file endpoint X, test di X, modelli usati da X | Altri endpoint, docs generali |
| Refactor architetturale | `CLAUDE.md`, ADR rilevanti, moduli coinvolti | Test dettagliati (solo header) |
| Nuova feature | `CLAUDE.md`, esempi di feature simili esistenti, schema DB | Deploy, CI/CD |
| Debug produzione | Log recenti, `CLAUDE.md`, codice del path critico | Tutto il resto |

### 3. Gestione della sessione
- **Checkpoint**: ogni tot di lavoro, chiedi a Claude di riassumere dove sei e committa il riassunto in un file
- **Context reset**: quando le risposte degradano, apri nuova sessione con riassunto + file rilevanti — meglio che proseguire una sessione satura
- **Evita catch-all**: "leggi tutta la codebase" raramente aiuta, quasi sempre confonde

### 4. Scrittura di `CLAUDE.md` efficace
- **Direttive esatte**, non suggerimenti vaghi. ❌ "Scrivi codice pulito". ✅ "Segui la skill `tdd-prove-it` per ogni fix."
- **Esempi di comandi** copiabili (`pytest tests/`, `go test ./internal/...`)
- **Link a skill/ADR/README** invece di espandere il contenuto
- **Aggiorna quando cambia qualcosa di strutturale**, non a ogni commit

### 5. Selective inclusion
Se usi Claude Code con tool di lettura file:
- **Inizia dalla minima informazione**, aggiungi solo quando serve
- **Non leggere file "per sicurezza"**: se non hai un'ipotesi su dove cercare, raffina l'ipotesi prima
- Usa `Grep`/`Glob` per restringere, poi `Read` mirato

## Anti-rationalization

| Scusa | Realtà |
|---|---|
| "Metto tutto in `CLAUDE.md`, così Claude sa tutto" | Claude legge tutto ma diluisce le priorità. Le sezioni importanti si perdono nel rumore. |
| "Carico il README completo in ogni sessione" | Il README è per umani. Per Claude, estrai le parti operative in `CLAUDE.md`. |
| "Più file leggo, meglio capisco il contesto" | Oltre una certa soglia, il context overload peggiora le risposte. Meno è più. |
| "Aggiorno `CLAUDE.md` solo quando mi ricordo" | Un `CLAUDE.md` obsoleto è un attivo tossico. Trattalo come `go.mod`: sempre al corrente. |

## Esempio `CLAUDE.md` minimale ma efficace

```markdown
# Project: ecs2 (Elastic Container Service v2)

Backend Django + Celery per gestione container su cloud Seeweb.

## Stack
- Python 3.11, Django 5, Celery 5, PostgreSQL 15, Redis 7
- Deploy: Docker Compose (dev), Kubernetes (stage/prod)

## Comandi
- Test unit: `pytest src/ -x`
- Test integrazione (richiede docker-compose up): `pytest src/ -m integration`
- Lint: `ruff check . && ruff format --check .`
- Type check: `mypy src/`
- Run dev: `docker compose up api worker`

## Convenzioni
- Use case in `src/ecs2/domain/use_cases/`, mai logica nei view.
- Task Celery in `src/ecs2/tasks/`, **sempre idempotenti** (vedi skill `celery-idempotency`).
- Migration: `python manage.py makemigrations --name descriptive_name`.
- Commit: conventional (`feat:`, `fix:`, `chore:`, `refactor:`).

## Cosa NON fare
- NON chiamare API esterne dentro `transaction.atomic()`.
- NON scrivere logica nei serializer DRF — delegata agli use case.
- NON usare `acks_early`, sempre `acks_late=True`.
- NON toccare `settings.production.py` senza ADR (vedi `docs/adr/`).

## Riferimenti
- Skill rilevanti: `tdd-prove-it`, `celery-idempotency`, `hexagonal-architecture`, `keycloak-oidc-patterns`
- Architettura: `docs/architecture.md`
- ADR: `docs/adr/`
- Runbook oncall: `docs/runbook.md`
```

## Esempio pattern di packing per una bug fix

```text
Task: "Fix bug: alcune task di allocazione backend lasciano lo stato in 'allocating'"

Context da caricare:
1. CLAUDE.md (sempre)
2. src/ecs2/domain/use_cases/allocate_backend.py
3. src/ecs2/tasks/allocate.py
4. src/ecs2/adapters/repositories/backend_repo.py
5. tests/test_allocate_backend.py (esempi)
6. Log recente: ultimi 200 righe del worker per ID backend affetti

Context da NON caricare:
- Altri moduli (billing, monitoring, notifications)
- Il frontend
- CI/CD config
- Docs ad alto livello
```

## Pitfall ricorrenti

- **CLAUDE.md "a due facce"**: metà direttive per Claude, metà note personali. Separa: direttive in `CLAUDE.md`, note in Obsidian o `docs/`.
- **Skill referenziate ma non installate**: inutile scrivere "usa `celery-idempotency`" se la skill non è attivata. Verifica il plugin manifest.
- **Duplicazione con README**: se cambi il comando test, cambia in entrambi. Meglio: link da `CLAUDE.md` → `README.md#testing` e un solo aggiornamento.
- **Segreti nel context**: mai includere `.env`, token, chiavi. Vedi skill `secrets-hygiene`.

## Exit criteria

- [ ] `CLAUDE.md` sotto 200 righe, risponde alle 6 domande strutturali
- [ ] Comandi riportati sono quelli **effettivi** (testati di recente)
- [ ] La lista "NON fare" ha almeno una voce specifica di questo progetto (non generica)
- [ ] File e docs linkati esistono davvero (no dead link)
- [ ] Per la task corrente, hai scelto cosa caricare **prima** di iniziare, non reattivamente
