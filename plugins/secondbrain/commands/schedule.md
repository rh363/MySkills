---
description: Aggiungi una o più task alla sezione ## Schedule del daily di oggi senza fare lo standup completo.
argument-hint: "[descrizione task] (opzionale; se vuota o multipla, modalità interattiva)"
---

# /schedule — aggiungi task al Schedule odierno

Sei dentro il second brain di Alex. L'utente vuole aggiungere una o più task alla sezione `## Schedule` del daily note di oggi, **senza** rifare lo standup mattutino. Carica la skill `vault-structure` come fonte di verità per layout e convenzioni del daily.

## Input

- Argomenti grezzi dal comando: `$ARGUMENTS`
- Data di oggi: usa `date +%Y-%m-%d` via Bash se non sei sicuro.
- Vault root: `~/Documents/Obsidian Vault/`
- File target: `Daily/YYYY-MM-DD.md` (oggi).

## Workflow

### Step 1 — Apri il daily di oggi

- Path: `Daily/<oggi>.md`.
- Se non esiste, **chiedi** ad Alex se vuole che venga creato dal template (vedi `vault-structure`) o se è meglio fare prima lo standup (`/daily-standup`). Non crearlo silenziosamente.
- Se esiste ma manca la sezione `## Schedule`, vai allo Step 4 (la creerai da zero).

### Step 2 — Parse argomenti

Comportamento basato su `$ARGUMENTS`:

- **Vuoto** → modalità interattiva: chiedi ad Alex le task da aggiungere (può dettare una lista). Per ogni task chiedi (in modo conciso, non un questionario):
  - descrizione
  - priorità P0/P1/P2 (default P1 se non specificata)
  - time slot (`HH:MM–HH:MM`) oppure stima (`~30min`, `~1h`); se Alex non lo dice usa `~30min` come default.
  - tag `#project/<nome>` se la task appartiene a un progetto.
- **Singola task** (testo libero senza newline) → aggiungi quella task. Inferisci tag e priorità dal testo se evidenti (es. `P0`, `#project/...` già presenti). Altrimenti chiedi UNA sola domanda di conferma sintetica con i campi mancanti (priorità + time slot).
- **Multi-line / lista** (newline o virgole tra task) → trattale come elenco. Una domanda riepilogativa sola per riempire priorità e slot mancanti, non una domanda per task.

Non inventare time slot orari precisi. Se Alex non li dà, usa stime tipo `~1h`.

### Step 3 — Calcolo dell'ordine

L'ordinamento del `## Schedule` è strettamente:
1. P0 → P1 → P2
2. A parità di priorità, per ora di inizio cronologica (le stime senza orario `~Xh` vanno **dopo** le task con clock time, nell'ordine in cui Alex le ha dettate).

### Step 4 — Scrittura

Formato di ogni riga (identico a quello prodotto da `daily-standup`):

```markdown
- [ ] <time-slot> — <descrizione> (#project/<nome> se rilevante) — P<n>
```

Esempi:

```markdown
- [ ] 14:30–15:30 — Review PR ermes (#project/ermes) — P1
- [ ] ~30min — Rispondere mail Antonello (#project/keycloak-oidc) — P2
```

Regole di inserimento:

- **Se `## Schedule` esiste già**: integra le nuove task **mantenendo l'ordine globale P0 → P1 → P2 + cronologico**. Non riscrivere o riordinare task esistenti completate (`- [x]`) — restano dove sono come memoria del piano. Inserisci le nuove `- [ ]` nella loro posizione corretta rispetto alle righe ancora aperte.
- **Se `## Schedule` non esiste**: creala subito dopo il titolo H1 e prima di `## Work log`.
- **Mai usare strikethrough `~~...~~`** — rompe il rendering Obsidian. Solo `- [ ]` / `- [x]` puliti.
- **Append-only sul piano**: non rimuovere mai task pre-esistenti (anche se Alex sembra volerle cancellare, chiedi conferma esplicita; di solito si marca come `#postponed` invece).
- Se è un re-run mid-morning e c'è già un blocco `### Update HH:MM` dal `daily-standup`, integra le nuove task **dentro** il blocco update più recente o crea un nuovo `### Update HH:MM` se sono passati più di ~30 minuti dall'ultimo. Usa l'ora corrente locale (`date +%H:%M`).

Aggiorna il frontmatter:

- `updated: <oggi>`
- Aggiungi tag `#schedule-update` se non presente (per distinguere da uno standup pulito).

### Step 5 — Conferma finale

Riepiloga in chat in italiano, conciso:

> "Aggiunte N task allo schedule di oggi: [breve elenco]. Schedule ora ha X task aperte."

Non chiedere conferma prima di scrivere se i dati sono completi — l'utente ha già lanciato il comando esplicitamente. Chiedi solo se mancano dati essenziali (priorità ambigua, time slot mancante quando ce ne sono altri orari).

## Stile

- Italiano di default. Tecnicismi in inglese ok.
- Sii rapido: questo comando deve essere più veloce di uno standup. Idealmente una o due interazioni totali.
- Se Alex passa già tutto pronto come arg (es. `/schedule 14:00–15:00 Review PR ermes #project/ermes P1`), scrivi direttamente senza fare domande.
- Non toccare `## Work log`, `## Done today`, `## Blocked`, `## Retro`, `## Planned tasks`. Questo comando lavora **solo** su `## Schedule`.
