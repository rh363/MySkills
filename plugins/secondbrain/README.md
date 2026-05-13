# secondbrain

Plugin Cowork per gestire il second brain di Alex (Obsidian vault in `~/Documents/Obsidian Vault`).

## Cosa fa

Codifica la struttura del vault e fornisce i workflow giornalieri:

- **Standup mattutino**: lettura dei planned tasks, prioritizzazione interattiva, scrittura di una sezione `## Schedule` nel daily di oggi.
- **Session teardown**: checkpoint tra una sessione di chat e l'altra. Appende una o più voci timestamped (`- **HH:MM — titolo**: ...`) sotto `## Work log` riassumendo la sessione appena chiusa, senza pianificare il giorno dopo né fare rollup.
- **Teardown serale**: chiusura della giornata. Conversazione libera + domande mirate, pianificazione task per il giorno target, e rollup automatico del daily appena chiuso nella struttura archivio.
- **Rollup**: cascata progressiva Daily → Weekly → Monthly → Yearly. Invocata automaticamente dal teardown e disponibile manualmente.

## Componenti

### Skills

| Skill | Tipo | Trigger principali |
|---|---|---|
| `vault-structure` | Reference | Domande sul vault, struttura, naming, frontmatter, "dove va questa nota?" |
| `daily-standup` | Azione | "standup", "pianifica oggi", "buongiorno" |
| `session-teardown` | Azione | "chiudo la sessione", "wrap-up sessione", "checkpoint sessione" |
| `daily-teardown` | Azione | "fai il teardown", "chiudo la giornata", "review serale" |
| `rollup` | Azione | "fai il rollup", "archivia il daily" (anche invocata da `daily-teardown`) |

### Slash commands

| Comando | Cosa fa |
|---|---|
| `/schedule [task]` | Aggiunge una o più task alla sezione `## Schedule` del daily di oggi, **senza** rifare lo standup. Supporta input rapido (`/schedule 14:00–15:00 Review PR — P1`) o modalità interattiva (`/schedule` senza argomenti). |

Nessun MCP server, hook o agent: il plugin lavora interamente sul filesystem del vault tramite i tool standard (Read, Write, Edit, Glob, Bash).

## Setup

Nessuna configurazione. Il plugin assume che il vault sia in `~/Documents/Obsidian Vault` (la cartella già selezionata in Cowork).

Le cartelle `Daily/`, `Weekly/`, `Monthly/`, `Yearly/` vengono create dal plugin la prima volta che servono.

## Convenzioni archivio

Cascata progressiva (un file vive UNA volta sola, viene spostato):

```
Daily/YYYY-MM-DD.md                                  # giornata corrente
Weekly/MM-DD/YYYY-MM-DD.md                           # settimana in corso (MM-DD = lunedì-venerdì del range)
Monthly/MM/MM-DD/YYYY-MM-DD.md                       # mese chiuso
Yearly/YYYY/MM/MM-DD/YYYY-MM-DD.md                   # anno chiuso
```

- Il teardown serale sposta il daily appena chiuso in `Weekly/MM-DD/`.
- Quando il teardown rileva fine settimana, sposta tutta la cartella `Weekly/MM-DD/` in `Monthly/MM/MM-DD/`.
- Quando rileva fine mese, sposta tutta `Monthly/MM/` in `Yearly/YYYY/MM/`.
- Idem per fine anno.

## Usage

Vedi le singole skill per i dettagli operativi:

- `skills/vault-structure/SKILL.md`
- `skills/daily-standup/SKILL.md`
- `skills/session-teardown/SKILL.md`
- `skills/daily-teardown/SKILL.md`
- `skills/rollup/SKILL.md`
- `commands/schedule.md`
